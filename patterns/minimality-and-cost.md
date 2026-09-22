# Minimality, Reuse, and Hot-Path Cost Patterns

Load this file when a CHANGE category adds a new file, type, helper, wrapper,
cache, queue, registry, parser, state machine, allocator, config flag, or
alternate code path; when it adds a fallback, retry, or recovery branch; or
when it touches a path that runs per request, per I/O, per packet, or per
lock acquisition.

The other pattern files ask "is this code correct?"  This one asks "should
this code exist, and what does it cost every time it runs?"  A retained line
is a future obligation; at equal behaviour, clarity, and safety, the version
with less live semantic surface -- fewer owners, paths, states, branches,
modes, conversions, allocations, copies, locks, atomics -- is the better one.
Unnecessary code is a maintainability finding, not a nit.  Do not reward
code golf: dense, clever, macro-hidden, or generated complexity is not
simpler, and necessary validation and invariant rationale stay.

---

## 1. New Mechanism Without a Reuse Decision

A second map, registry, queue, cache, state machine, allocator, or parser
next to an existing one creates coherence, lifetime, and performance bugs
unless it has a clear ownership boundary.  Before a change adds one, the
author must have inspected the existing owner and decided to extend, repair,
delete-and-replace, or leave it alone -- and said why.

```c
/* SUSPECT: a per-connection lookup table added beside the existing
 * per-server one, keyed the same way */
struct conn {
    struct cds_lfht *c_sessions;   /* new */
};
/* ... while server_state already owns ss_sessions keyed by sessionid */
```

Ask: which existing module owns this behaviour today?  Could it be
parameterised or repaired instead of wrapped?  A convenience wrapper is
acceptable only when it removes real duplication without hiding cost,
allocation, locking, error behaviour, or invariants.  Moving logic sideways
into a new file to avoid understanding the current owner is a design smell.

---

## 2. Second Source of Truth

State mirrored into an auxiliary structure that must be kept in sync with
the owner -- a cached count, a shadow flag, a parallel list -- is a bug
waiting for the one path that updates one side and not the other.

```c
/* WRONG: nlink kept on the inode AND recomputed into a dirent count */
inode->i_nlink++;
parent->rd_child_count++;        /* second source of truth */
```

Require one owner per state bit.  If a mirror is genuinely needed (a
read-mostly cache, a lock-free snapshot), the change must name the owner,
the synchronisation protocol, the lifecycle, and the test that exercises
the two falling out of step.

---

## 3. Replaced Behaviour Whose Old Path Stays Live

When new code replaces old behaviour, the old path is deleted or quarantined
in the same change.  Two live paths are acceptable only for an explicit,
tested compatibility window with a stated end.

```c
/* WRONG: new encoder wired in, old one still reachable via a flag
 * nobody sets and nobody tests */
if (use_v2_encoder)
    return encode_v2(...);
return encode_v1(...);   /* dead in practice, live to the compiler */
```

Check `git grep` for callers of the old path after the change.  If none
remain, the old path goes.  If some remain, the change either converts them
or explains the window.

---

## 4. Cope Code: Continuing Past an Impossible State

A fallback, retry, cleanup, remapping, widening, clamping, silent repair, or
alternate path whose real purpose is to make progress after an internal
invariant, precondition, ownership rule, or structural contract was violated
hides the bug in the owner that violated it.

```c
/* WRONG: the lookup "can't fail" here; the fallback hides the caller
 * that broke the invariant */
entry = table_find(ht, id);
if (!entry)
    entry = table_insert(ht, id);   /* cope: who forgot to insert? */

/* CORRECT: fail at the owner boundary with the context that names it */
entry = table_find(ht, id);
if (!entry) {
    LOG("id %lu not in table: owner %s never registered it", id, who);
    return -EINVAL;   /* or assert in a debug build; do not continue */
}
```

Retries and backoff are correct only for explicitly modelled external
conditions: a documented protocol state, resource contention, a transient
hardware or I/O status, a race with another actor the design admits.  They
are never correct for a logically impossible state.  When a change adds a
retry, ask which of those it models; if the answer is "just in case", it is
cope code.  Turning a wrong internal decision into a success path is a
BLOCKER even when the tests pass.

Related: `technical-patterns.md` on defensive checks that mask bugs, and
`patterns/error-handling.md` on which failures are expected.

---

## 5. Unbudgeted Cost on a Hot Path

On a path that runs per request, per I/O, per packet, or under a lock, every
allocation, bulk copy, lock, atomic, syscall, page fault, `snprintf`, and log
line is a cost that must be justified.  Code that looks allocation-free can
still be wrong if it moves bytes: `memcpy`/`memmove`, large structs passed or
returned by value, aggregate assignment, compaction, serialisation staging.

```c
/* SUSPECT on a per-RPC path: a 4 KiB struct copied by value to
 * pass three fields */
static int handle(struct compound c)      /* by value */
```

```c
/* SUSPECT under a lock: formatting that runs whether or not the
 * trace category is enabled */
pthread_mutex_lock(&sb->sb_lock);
TRACE("sb %s: %s", sb_path(sb), describe(sb));   /* describe() allocates */
```

Ask the author to classify the path -- hot, warm, cold, init-only,
shutdown-only, test-only -- and, for a hot path, to state the allocations,
bytes copied, locks, and syscalls the change adds per operation.  A
performance claim without a measurement or a credible measurement plan is
not evidence; a profile sample inside a lock is not evidence that the lock
is the problem rather than a caller taking it too often.

---

## 6. Special-Case Fast Path Ahead of the General Bottleneck

A narrow fast path for one input shape is a finding when the general path
still carries a known shared-writable-cacheline, lock, allocator, or
accounting cost on every operation.  The special case adds surface and
distracts from the owning bottleneck.  Require evidence that the general
path has been fixed or is explicitly out of scope before accepting the
narrow one.

---

## 7. Change Too Large to Review

Defect-finding quality collapses past a few hundred meaningful lines.  If
the change cannot be reasoned about in one pass, ask for a split or an
author-provided review map (which hunks are mechanical, which carry the
behaviour change, in what order to read them) before reviewing, and say so
in the report rather than reviewing it badly.

---

## Verification Output Requirements

For any minimality, reuse, or cost finding:
1. Name the mechanism being added and the existing owner it duplicates, by
   file:line for both.  A finding that only says "this could reuse
   something" without naming what is not actionable.
2. For cope code: name the invariant being violated, the owner that is
   supposed to uphold it, and the path on which it is violated.
3. For hot-path cost: name the path, why it is hot (per what?), and the
   cost added per operation (allocations, bytes copied, locks, syscalls).
4. For a retained old path: show the `git grep` for its remaining callers.
5. Line count, single-use status, and reviewer preference alone do not
   justify a finding.  If the evidence only shows that a reuse decision or
   cost justification is missing from the change, report that as the
   finding -- do not assert redundancy you have not shown.
