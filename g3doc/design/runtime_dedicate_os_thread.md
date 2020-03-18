# Proposal: `runtime.DedicateOSThread`

-   Status: Draft as of 2020-03-18
-   Author: jamieliu@google.com

## Summary

Allow goroutines to bind to kernel threads in a way that allows their scheduling
to be kernel-managed rather than runtime-managed.

## Objectives

-   Reduce Go runtime scheduling overhead in the gVisor sentry. (See e.g.
    https://gvisor.dev/issue/205, https://gvisor.dev/issue/1942.)

-   Minimize intrusiveness of changes to the Go runtime.

## Background

### Go Runtime Scheduling

In Go, execution contexts are referred to as goroutines, which the runtime calls
Gs. The Go runtime maintains a variably-sized pool of threads (called Ms by the
runtime) on which Gs are executed, as well as a pool of "virtual processors"
(called Ps by the runtime) of size equal to `runtime.GOMAXPROCS()`. Each M
requires a P in order to execute Gs, limiting the number of threads concurrently
executing goroutines to `runtime.GOMAXPROCS()`.

### Goroutine Scheduling

Goroutines usually block for one of three reasons:

-   Blocking on socket I/O

-   Inter-goroutine communication (via channels or `sync` package primitives)

-   Invocation of a blocking system call

In the first two cases, the Go runtime deschedules the blocked G from the M and
switches to an unblocked G. In the third case (syscall), the M executing the
syscall cannot switch to other Gs. Instead, the runtime maintains a "sysmon"
thread. Loosely speaking, if the sysmon thread observes that there are runnable
goroutines without enough Ms to run them, the sysmon thread will steal Ps from
Ms blocked in long (> 20us) syscalls, and transfer them to idle or new Ms.

### `runtime.LockOSThread`

The `runtime.LockOSThread` function temporarily locks the invoking goroutine to
its current thread. It is primarily useful for interacting with OS or non-Go
library facilities that are per-thread. It does not reduce interactions with the
Go runtime scheduler: locked Ms relinquish their P when they become blocked, and
only continue execution after another M "chooses" their locked G to run and
donates their P to the locked M instead, ensuring that `runtime.LockOSThread`
does not circumvent `GOMAXPROCS`.

## Problem

Most goroutines in the gVisor sentry are task goroutines, which back application
threads. Task goroutines spend large amounts of time blocked on syscalls that
execute untrusted application code. Since one task goroutine is needed per
application thread, it is common for the sentry to be running hundreds or
thousands of task goroutines simultaneously, each of which requires its own M.
With or without `runtime.LockOSThread`, each of these Ms must coordinate with
the Go runtime scheduler on each blocking and unblocking event, creating an
immense scalability burden.

## Proposal

We propose to add a function, tentatively called `DedicateOSThread`, to the Go
`runtime` package, documented as follows:

```go
// DedicateOSThread wires the calling goroutine to its current operating system
// thread, and exempts it from counting against GOMAXPROCS. The calling
// goroutine will always execute in that thread, and no other goroutine will
// execute in it, until the calling goroutine has made as many calls to
// UndedicateOSThread as to DedicateOSThread. If the calling goroutine exits
// without unlocking the thread, the thread will be terminated.
//
// DedicateOSThread should only be used by long-lived goroutines that usually
// block due to blocking system calls, rather than interaction with other
// goroutines.
func DedicateOSThread()
```

Mechanically, `DedicateOSThread` implies `LockOSThread` (i.e. it locks the
invoking G to its M), but additionally locks the invoking M to its P. Ps locked
by `DedicateOSThread` are not counted against `GOMAXPROCS`; that is, the actual
number of Ps in the system (`len(runtime.allp)`) is `GOMAXPROCS` plus the number
of bound Ps (plus some slack to avoid frequent changes to `runtime.allp`).
Corollaries:

-   If `runtime.ready` observes that a readied G is locked to a M locked to a P,
    it immediately wakes the locked M instead of waiting for a future call to
    `runtime.schedule` to select the readied G in `runtime.findrunnable`.

-   `runtime.stoplockedm` and `runtime.reentersyscall` skip the release of
    locked Ps; the latter also skips sysmon wakeup. `runtime.stoplockedm` and
    `runtime.exitsyscall` skip re-acquisition of Ps if one is locked.

-   New goroutines created by goroutines with locked Ps are enqueued on the
    global run queue rather than the invoking P's local run queue.

While gVisor's use case does not strictly require that the association is
reversible (with `runtime.UndedicateOSThread`), such a feature is required to
allow reuse of locked Ms, which is likely to be critical for performance.

## Alternatives Considered

-   Make task goroutines execute untrusted application code asynchronously, in a
    way that allows them to block in Go code rather than on syscalls. This would
    reduce concurrency pressure on the Go runtime scheduler, but only at
    unacceptable cost to the latency of switches to and from application code
    (in fact, a significant motivator for platforms other than ptrace is
    avoidance of asynchronicity overhead).

-   Make P-locking part of `LockOSThread`'s behavior. This would likely
    introduce performance regressions in existing uses of `LockOSThread` that do
    not fit this usage pattern. In particular, since `DedicateOSThread`
    transitions the invoker's P from "counted against `GOMAXPROCS`" to "not
    counted against `GOMAXPROCS`", it may need to wake another M to run a new P
    (that is counted against `GOMAXPROCS`), and the converse applies to
    `UndedicateOSThread`.

-   Improve the Go runtime scheduler to the point where it is equivalent or
    superior in performance to the host kernel scheduler in all circumstances.
    This is not considered realistic.

-   Rewrite the gVisor sentry in a language that does not force userspace
    scheduling. This is a poor option due to the amount of code involved.

## Related Issues

Outside of gVisor:

-   https://github.com/golang/go/issues/21827#issuecomment-595152452 describes a
    use case for this feature in go-delve, where the goroutine that would use
    this feature spends much of its time blocked in `ptrace` syscalls.

-   This feature may improve performance in the use case described in
    https://github.com/golang/go/issues/18237, given the prominence of
    syscall.Syscall in the profile given in that bug.
