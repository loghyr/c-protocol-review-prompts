# Reachability-Change Patterns

Load this file when CHANGE categories involve:

- Extraction refactors that move code between compilation units (e.g.,
  from a backend-specific `.c` into a shared `.c`).
- Backend / platform ports that activate previously-stub code paths.
- New callers introduced for pre-existing functions.
- Feature flags flipping a guarded region into the default path.
- Removing `__attribute__((unused))`, `#if 0`, or `static` that had
  been hiding code from the linker.

In short: loaded code that was never *reached* at runtime is
effectively untested. When a change makes such code reachable for the
first time, every latent bug it contains becomes a live bug with the
same commit.

---

## 1. The Bidirectional Reachability Gate

`review-core.md`'s standard Reachability Gate asks a forward question:

> Is the **new** code reachable by the workloads described?

That's necessary but not sufficient for refactors and ports. Also
ask the reverse:

> Does the new code make any **old** code reachable for the first time?

If yes, that old code is now in scope. It hasn't been exercised by any
test suite or workload before this commit; its bugs have been
dormant. The review must cover it as if it were new.

### Concrete example: the accept-NULL hotfix

A server's kqueue backend skeleton was merged weeks earlier, with
`io_request_accept_op` implemented:

```c
int io_request_accept_op(int fd, struct connection_info *ci,
                         struct ring_context *rc)
{
    struct io_context *ic = io_context_create(OP_TYPE_ACCEPT, fd, NULL, 0);

    if (!ic)
        return -ENOMEM;

    ic->ic_ci = *ci;               /* <-- unconditional deref */
    ...
}
```

The sibling liburing implementation guarded the copy with `if (ci)`.
The kqueue version didn't. It had sat this way for a month — but the
path was never exercised because the main server code hadn't yet
reached `io_request_accept_op` on FreeBSD (earlier layers were
stubbed).

A later extraction PR finished wiring the network handlers so the
full data path ran. Reviewers of that PR carefully audited the new
code; reviewers of the previous PR had long ago signed off. Nothing
in the extraction PR was wrong.

The server crashed at `memcpy` on first run. `main()` calls
`io_request_accept_op` with `ci == NULL` for the initial listener
arming. The bug had been latent since the month-old skeleton commit;
the extraction PR made it reachable.

The fix was a one-line `if (ci)` guard. The lesson is that the
*review scope* for the extraction PR was too narrow — it treated the
old code as already-reviewed and out of scope. A review that asks
"what did this change newly reach?" would have pulled
`io_request_accept_op` into scope.

---

## 2. Cross-Backend Invariant Audit

Multi-backend codebases (io_uring / kqueue / thread-pool, POSIX /
Windows, glibc / musl) encode the same invariants using different
primitives. An invariant that holds trivially on backend A may fail
on backend B because B's primitive has different semantics.

When a change unifies code across backends — e.g., extracts a
handler into a shared file that both backends compile — every
invariant the handler assumes must be **re-verified against each
backend's primitives**.

### Concrete example: EV_ADD | EV_ONESHOT collision

A server had a per-fd write-serialization gate (flag plus linked
list in `conn_info`) that ensured at most one write completion could
be in flight per fd. Under io_uring this matched the kernel's
behavior: submitting two write SQEs on the same fd is allowed, and
the kernel linearizes them by TCP order.

After an extraction PR, a kqueue backend started using
`kevent(EV_ADD | EV_ONESHOT, EVFILT_WRITE, fd, udata=ic)` for write
submission. Two concurrent registrations on the same (fd, filter)
do not linearize — FreeBSD's kqueue *replaces* the udata on the
existing knote:

> Re-adding an existing event will modify the parameters of the
> original event.
> — kqueue(2), FreeBSD 13

So the per-fd write-serialization gate was newly *load-bearing* on
the kqueue backend in a way it hadn't been on io_uring. If any code
path submitted via `io_request_write_op` outside the gate (for
example, the TLS-data submission in `io_do_tls`), a subsequent
normal writer would silently orphan the TLS-data ic: its knote got
replaced, its buffer leaked, its reply never sent.

The review that caught this asked: "does the gate invariant hold on
both backends given their different primitives?" The answer was
"yes for the normal path, no for the TLS path — but TLS is
deferred, so the violation is unreachable today." That analysis
produced a precondition documented in the commit message and a
follow-up blocker recorded for the TLS port.

### Review technique

For each invariant the moved code assumes, list the two backends'
primitives side by side:

| Invariant | Backend A primitive | Backend B primitive | Still holds on B? |
|---|---|---|---|
| At most one write in flight per fd | io_uring: kernel-linearized | kqueue: knote replacement on re-add | **Only if gate is respected** |
| Error returned as negative errno | io_uring: cqe->res | kqueue: `write(2)` ret + errno | No — needs `-errno` translation |
| ic ownership transits kernel boundary once | io_uring: SQE submit | kqueue: kevent ADD | Same, but retry semantics differ |

---

## 3. Extraction / Port Pattern

Extractions follow a characteristic shape. Each step has a
specific review concern:

1. **New shared file compiled on all backends.** Review: does the
   new file depend on anything backend-specific? Is the include set
   minimal?
2. **Old backend-specific file shrinks.** Review: does anything
   still in the old file depend on the moved code via file-scope
   statics? Are forward declarations stale?
3. **Stubs deleted from inactive backend.** Review: does removing
   the stub break the link on the inactive backend? If so, the
   shared code must be compilable there (or the stub must stay for
   link-only paths).
4. **Move is semantically identical — no behavior change.** This is
   usually the commit message's claim. Review: the move truly is
   identical. Any "drive-by" fix (variable rename, NULL guard, lock
   rebalance) hidden in the move is a BEHAVIOR DELTA that must be
   called out.

### Example of a BEHAVIOR DELTA hidden in an extraction

A move of `io_handle_write` from backend A to a shared file added
`conn_buffers[i] = NULL;` after `free()` in the cleanup loop. The
pre-move version did not null the slot. The commit message said
"pure code move." In practice this is a safety improvement (the
slot is no longer a dangling pointer), not a regression, but it
is a BEHAVIOR DELTA — the move was not pure. Reviewers should call
this out and ask the author to either split it into a second commit
or call it out in the commit message body.

---

## 4. Stub Removal Must Be Atomic with Caller Activation

When a stub `-ENOSYS` implementation is replaced by the real thing,
the caller that newly activates it must land in the same commit.
Splitting these across commits produces bisect traps:

- Commit N: remove stub, caller still calls fake-success stub.
  Build breaks on the backend that had the stub (duplicate symbol
  or missing symbol), even though each commit individually "works
  on my machine."
- Commit N+1: wire up caller. Now the real code runs. But a bisect
  to find a regression between commit N-1 and N+2 can't build at
  commit N.

Review: verify the commit that removes a stub also removes (or
updates) every `-ENOSYS` caller that was masking the absence. Grep
for every callsite of the stub before approving.

---

## 5. Smoke-Run Mandate for Reachability Changes

Static code review does not catch runtime-only bugs. For:

- Backend / platform ports (new OS, new I/O model, new arch).
- Extraction refactors that activate previously-stubbed code.
- First-time wiring of a feature flag default.

A **smoke run** is a required review step. That means: build on
each target, run the binary end-to-end, exercise the newly-reached
code path with representative input, and verify no crash / leak /
hang.

### Minimum smoke for a backend port

1. Binary starts and enters main loop on the new backend.
2. Simplest client connects and does one round-trip (e.g., NFS
   `NULL` procedure, HTTP GET, one SQL `SELECT 1`).
3. A non-trivial payload round-trips intact (e.g., 16 MB file write
   + sha256 read-back match).
4. Two concurrent clients produce independent results (exercises
   any per-fd serialization).
5. Graceful shutdown: SIGTERM produces clean exit, no leaked fds /
   memory visible via `fstat` / `vmstat` within 5 seconds.

### Smoke catches what static review misses

Three kinds of bug pass static review but die at the first smoke:

- **NULL-pointer deref at a rarely-reached callsite.** The reviewer
  sees the code, the reviewer checks it matches the sibling backend.
  The reviewer does not notice that the *caller* of this
  backend-specific code passes different arguments on the new
  backend. Example: `io_request_accept_op(fd, ci=NULL, rc)` from
  `main()`.
- **Calling-convention mismatch.** One backend's completion comes
  with a negative errno; another returns -1 + errno. Reviewer sees
  the handler expects "negative errno" and the submission path uses
  "-1 + errno," but the intermediate dispatch that glues them is
  subtle. Smoke reveals `strerror(1)` ("Operation not permitted")
  in logs instead of the real error.
- **Buffer offset ignored.** Handler uses `ic->ic_buffer` from the
  start. Reviewer sees correct offset math in the sibling backend.
  Reviewer does not notice the port omits the offset. First
  multi-chunk write corrupts data on the wire. Unit tests with
  <4 KB payloads never trigger it.

Smoke is not a substitute for review — review catches many bugs
smoke misses (lock order, UAF-after-suspend, sanitizer-only races).
But for reachability-change commits, review alone is not enough.

---

## 6. Checklist for Reachability-Change Review

Before approving a commit that changes reachability:

1. **Forward reachability confirmed?** The new code path runs for
   the workloads the commit message describes.
2. **Backward reachability audited?** Every function the new code
   calls for the first time has been re-reviewed for NULL guards,
   edge-case inputs, and error paths, as if it were new code.
3. **Cross-backend invariants verified?** For each shared
   invariant, listed both backends' primitives and confirmed the
   invariant holds on both (or documented the precondition under
   which it holds on the weaker one).
4. **Stub removal atomic with activation?** No intermediate commit
   has a caller calling a removed stub.
5. **BEHAVIOR DELTAs called out?** Any drive-by changes hidden in
   an "extraction" commit are either split out or called out in
   the body.
6. **Smoke run done on each target?** For a port, build and run on
   the new target with the minimum smoke from Section 5. Include
   the smoke evidence in the review (log excerpt, sha256 match,
   etc.).
7. **Preconditions documented for deferred work?** If the change
   relies on unreachability of some sub-path (e.g., "TLS is
   deferred so `io_do_tls` is never called"), that precondition is
   explicit in the commit message and recorded as a blocker for the
   PR that removes the unreachability.

---

## Verification Output Requirements

For any reachability-change issue:

1. Name the commit that introduced the latent bug AND the commit
   that activated it. Both are in scope; the activating commit
   makes it live.
2. Quote the latent-bug site and the activating call site with
   file:line for each.
3. If cross-backend: show the two backends' primitives and the
   invariant mismatch.
4. If smoke-run-would-catch: describe the minimum smoke that would
   have surfaced the bug. Encourage the author to add it to the
   project's smoke harness.
