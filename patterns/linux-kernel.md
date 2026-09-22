# Linux Kernel Patterns

Load this file when the code under review is in-kernel or kernel-adjacent:
an in-tree subsystem, an out-of-tree module, a driver, or userspace code
whose contract is a kernel interface.

**This file is a bridge, not a kernel reviewer.**  The kernel-specific
machinery -- per-subsystem guides, `Fixes:` tag derivation, lore thread
handling, checkpatch conventions, inline reply formatting, coccinelle
sweeps -- lives in Chris Mason's
[masoncl/review-prompts](https://github.com/masoncl/review-prompts), whose
methodology this suite already adapts for userspace.  When reviewing a
kernel patch, load that suite's `kernel/technical-patterns.md`,
`kernel/review-core.md`, and the matching file from `kernel/subsystem/`.
Come back here for the two things it does not cover: what changes in this
suite's own patterns when the target is the kernel, and the evidence
discipline that wire-testing a kernel change demands.

---

## 1. Execution Context Is the First Question

Every userspace pattern in this suite assumes a thread that may sleep,
allocate, and take a mutex.  In the kernel none of that is given.  Before
analyzing any CHANGE in kernel code, name the context of the path:

- process, workqueue, softirq, hardirq, NMI;
- RCU read-side or not;
- preemptible or not, sleepable or atomic;
- reclaim, usercopy, VFS, MM, block, device/interrupt;
- holding which locks, entered from which caller.

Then ask, for each line the patch adds: may it sleep?  May it allocate,
and with which `GFP_` flags?  May it fault?  May it take this lock here?

```c
/* WRONG: GFP_KERNEL allocation under a spinlock -- may sleep */
spin_lock(&foo->lock);
p = kmalloc(sizeof(*p), GFP_KERNEL);

/* WRONG: the same allocation on the reclaim path -- recursion */
/* CORRECT: allocate before the lock, or GFP_ATOMIC with a failure path,
 * or a mempool if failure is not acceptable */
```

A patch that does not let the reviewer establish the context has not been
reviewed.  Ask for it rather than guessing; a userspace pass, a local
module smoke test, or a clean compile proves nothing about context.

---

## 2. Kernel Primitives, Not Rolled Ones

`patterns/ref-counting.md` and `patterns/rcu-and-lockless.md` describe
liburcu and C11 shapes.  In the kernel the same jobs have owners:

| Job | Kernel primitive |
|-----|------------------|
| Reference count | `refcount_t`, `kref` |
| Sparse integer map | `xarray`, `idr` |
| Hash table | `rhashtable`, `hlist` |
| Intrusive list | `list_head`, `hlist` |
| Deferred free | `call_rcu`, `kfree_rcu` |
| Read-mostly sequence | `seqlock_t`, `seqcount_t` |
| Wait / wake | wait queues, `completion` |
| Deferred work | workqueue, `delayed_work` |
| Per-CPU counter | `percpu_counter`, `this_cpu_*` |

A new allocator, registry, cache, lifetime protocol, or work scheduler in
a patch needs a subsystem-level reason.  Flag one that has none, and name
the primitive it should have used.

`refcount_t` is not `atomic_t`: it saturates instead of wrapping and warns
on use-after-free shapes.  A patch that converts `refcount_t` back to
`atomic_t` to get an operation the API deliberately omits is a finding.

---

## 3. LKMM, Not Userspace Intuition

`patterns/atomics.md` reasons in C11 memory ordering.  The kernel has its
own model.  Do not carry conclusions across without translating them:

- `READ_ONCE` / `WRITE_ONCE` for plain accesses that race benignly;
- `smp_load_acquire` / `smp_store_release`, `smp_mb`, `smp_rmb`, `smp_wmb`
  rather than `memory_order_*`;
- `rcu_dereference` under `rcu_read_lock`, `rcu_assign_pointer` to publish;
- `lockdep` classes and documented lock ordering, not a comment claiming it;
- interrupt and preemption disable regions as part of the ordering argument.

A concurrency claim in a kernel patch should say what a reader may observe
during publication and during teardown.  If the changelog says "this is
safe because of the barrier" and there is no barrier in the diff, that is
the finding.

---

## 4. Patch Hygiene Is Part of Correctness

- Small, subsystem-scoped patches; one logical change each.
- `Signed-off-by:` present; `Fixes:` when the patch repairs a specific
  commit; stable tags per the documented rules.
- Kernel coding style, with checkpatch clean for new code.
- No private compatibility layer that bypasses a subsystem maintainer or
  an existing upstream helper.
- The changelog explains *why*, and every claim in it is checkable against
  the diff.

Per `patterns/minimality-and-cost.md`, treat an added abstraction the
subsystem already provides as a finding.  In the kernel the bar is higher,
not lower: the maintainer will ask the same question and the patch dies in
review if the answer is not in the changelog.

### Source comments carry upstream references only

A kernel source comment is read by people with none of the context the
change was developed in.  Cite only what an upstream reviewer can look up:
a specification section, an RFC, another site in the kernel tree, the
standard that defines the wire behavior.

```c
/* WRONG: private process trail, means nothing upstream */
/* memo rev-6 reviewer-2 :174-180; slice ordering [1,2,4,3]; task #508 */

/* CORRECT: the constraint, and where it comes from */
/* Gate on minor_version >= 2: this operation is defined only for the
 * later protocol revision; see <spec> section <n>.
 */
```

Review history, internal revision numbers, and reviewer dispositions do
not belong in source.  Flag them in a diff: they are noise to the
maintainer and they date the code.  Tool use is different when it
materially generated the contribution: follow
`Documentation/process/generated-content.rst` and disclose it in the
cover letter or changelog, not in an implementation comment.

---

## 5. Verify the Artifact Under Test Is the Artifact You Built

This is the hardest-earned rule here, and it is a *review* rule: evidence
measured against an unidentified binary is not evidence.  When a finding,
a benchmark, or a "this is fixed" claim rests on a run, ask which build
produced it and how that was established.  Four ways it has gone wrong,
each seen in practice:

1. **The installed image is not the built image.**  An install step that
   silently did not copy leaves a previous kernel in place under the new
   version's name -- byte-identical to the older build, paired with a new
   initramfs, hanging early in boot.  Only a checksum against the build
   tree's image shows it.
2. **Building a directory does not build the module.**  `make <subdir>/`
   compiles objects; only a modules build runs modpost and links the
   `.ko`.  A stale `.ko` sits next to fresh `.o` files and looks current.
3. **An external module build writes beside the source, not into the
   build directory.**  An earlier in-tree build leaves a `.ko` in the
   output tree that still looks exactly like the fresh output.  A
   checksum that merely matches *a* candidate is not enough when more
   than one plausible output exists -- that check has passed against the
   wrong file.
4. **Installed is not loaded.**  After install and depmod, the previous
   module can still be resident.  Every on-disk check passes while the
   running kernel executes the old code.

Discipline that catches all four, before the run rather than after:

- checksum the installed image against the build tree's;
- verify module *content*, not just that a file matched: `nm` for a symbol
  only the new code defines, or disassemble the one function the change
  touched and read the instructions;
- pick an anchor symbol that survives compilation -- a static function may
  be inlined away, so inspect the `.ko`'s symbols before choosing;
- reload and confirm residency; on-disk freshness is not residency;
- record the identity (checksum, symbol, version string) in the evidence
  alongside the result.

Timestamps lie in both directions, especially across a shared or
network-mounted build directory.  Do not reason from mtime.

---

## 6. A Hang Is Evidence; Do Not Convert It Into an Error Code

When a kernel path wedges under test -- an uninterruptible sleep, a
blocked task warning, a stuck mount -- the wedge is the signal that
something cannot make progress.  The tempting fix is to change the test
posture so the symptom becomes a clean failure: a soft or timing-out
mount, a shorter retry limit, a fallback path enabled only in the harness.

Three reasons to refuse:

1. The posture under test should be the posture that ships.  Testing a
   different one answers a different question.
2. Softening a failure mode is a project, not a knob: it needs timeout
   and retry tuning, and every caller must handle mid-operation failure
   without silent truncation or corruption.  Turning it on for
   convenience buys a hidden correctness risk.
3. The hang localizes the defect.  Converted to an error return, it
   becomes an unexplained failure somewhere else, and the protocol-level
   livelock or missed wakeup that caused it is buried.

Find what cannot make progress: the stack of the blocked task, what it
waits on, which side owes a wakeup or a reply.  Flag a patch that adds a
timeout, retry, or fallback whose real purpose is to stop a hang from
being visible -- `patterns/minimality-and-cost.md` calls that cope code,
and in the kernel it is worse, because the state it papers over is
usually a lifetime or wakeup bug.

---

## 7. Config, Module, and Reachability Gates

`patterns/reachability-change.md` applies with kernel-specific entry
points.  Before reporting a finding, and before accepting a "this path is
dead" dismissal, check:

- the `Kconfig` symbol that guards the code, and whether the test
  configuration set it;
- `#ifdef CONFIG_*` inside the file, including the `#else` arm;
- whether the module is built at all, and whether it is loaded;
- whether a tristate is built-in or modular in the configuration used;
- whether the caller is reached only from a path the boot or the mount
  options disable.

A commit that enables a Kconfig symbol, removes a stub, or adds the first
caller of an existing function makes pre-existing code reachable for the
first time.  That code is in scope for the review, and it has never been
exercised by any test.

---

## 8. A Green Command Is Not Necessarily Evidence

Kernel validation is a ladder, not one result:

1. the changed translation unit compiled;
2. the final built-in image or module linked and passed modpost;
3. the intended configuration selected the changed implementation;
4. that artifact was installed and loaded;
5. the intended test case executed;
6. the test exercised the predicate or race it claims to cover.

A result proves only the rung it reached.  In particular:

- A KUnit suite that compiled was not necessarily run.  Check the raw
  output, suite and case names, and the executed test count; a bad filter
  can select zero tests and still leave an apparently successful command.
- For lockdep evidence, confirm that lockdep was enabled in the running
  configuration and that the log does not say the locking correctness
  validator was turned off.  Absence of a splat alone is not evidence.
- An empty sparse, Smatch, Coccinelle, sanitizer, or checkpatch log is not
  a pass until the tool version, exit status, input files and output
  destination show that the tool actually ran.  Compare base and tip under
  the same configuration, and rebuild the relevant objects when cached
  output could hide the changed lines.
- A fault-injection test must show that the intended injection point fired.
  Run a negative control with injection disabled, and where practical a
  deliberately broken control that makes the test fail for the expected
  reason.
- A race test needs deterministic handshakes around the contested window.
  Sleeps, scheduler luck and "the worker had started" do not prove that a
  waiter was blocked, teardown was waiting, or the target instruction ran.
  Keep ownership of every test thread and join it before its fixture can be
  freed.

When a branch cannot be driven on the available host, label it source-only
analysis.  Do not promote it to runtime evidence because a neighboring case
passed.  Report the exact command, config, test count and limitations so the
next reviewer can tell which rung was reached.

---

## 9. Publication, Admission, and Teardown Are One Protocol

A registry or provider is not safe merely because lookup is locked.  Review
the whole lifetime as one state machine:

1. initialize the object and all failure unwinds before publication;
2. publish under the registry's exclusion rule;
3. let each caller acquire a stable object/module/generation reference and
   an active admission before dropping lookup protection;
4. refuse new admissions after retirement starts;
5. unpublish first, then wait for existing admissions and callbacks;
6. release owned generations and storage only after the drain; and
7. destroy the anchor only after no waiter can still name it.

Keep ownership references distinct from active-operation admissions.  A
published object may own a generation, but it must not hold a permanent
"active admission" that retirement waits to drain.  Likewise, an object
whose teardown is a per-net exit callback must not keep its own network
namespace alive until that callback: both patterns create a reference cycle
that prevents the teardown event needed to release the reference.

Wait primitives carry their own cardinality and lifetime contracts:

- `complete()` satisfies one waiter; use `complete_all()` only when the
  event is a broadcast and account for future generations of the event;
- do not `reinit_completion()` while a waiter or signaler can still refer
  to the old generation;
- publish the state that satisfies a wait before waking it, with the lock or
  LKMM ordering that makes the state visible; and
- the completion or wait queue must outlive every waiter, including timeout,
  cancellation and error paths.

For teardown tests, prove both halves separately: the fence returns only
after active work drains, and object destruction itself remains blocked
until the last owned reference is released.  Fixture cleanup that runs only
after the test body has already released the worker proves neither.

---

## 10. Tool Disclosure Is Not Authorship

Read the target tree's `Documentation/process/generated-content.rst`.
Material tool-generated code, tests, changelog text, or problem discovery
should be disclosed as that document requests, with enough information for
reviewers to understand what was generated and how it was validated.  The
human submitter must understand and be able to defend the entire change.

Do not turn a tool or model into a person-shaped trailer.  `Signed-off-by:`
records the DCO and the real delivery path; `Co-developed-by:` names a human
co-developer and requires that person's sign-off.  A tool has neither role.
Also reject duplicated sign-offs and mechanical
`(cherry picked from commit ...)` debris when a series is being rebuilt for
upstream review.  The trailer block must describe what actually happened,
not expose the mechanics of a private patch-replay workspace.

---

## 11. Pin Every Side of a Cross-Tree Claim

Kernel protocol work often has three authorities in different trees: the
kernel endpoint, a peer or reference implementation, and a specification.
"Matches the server" is not review evidence until all three are identified
by exact revision.  Before checking a wire or persistent-format claim,
record:

- the kernel commit and branch under review;
- the exact peer/reference commit and the path treated as canonical; and
- the specification revision and section defining the field or operation.

Do not substitute a nearby clone, an old topic worktree, a generated header
whose source may have moved, or line numbers from another revision.  Verify
the checkout's commit before citing it.  If two peer trees disagree, report
the disagreement as a cross-tree dependency instead of choosing the version
that makes the patch pass.

For every changed wire shape, independently check:

- operation numbers, discriminants, field order and fixed widths;
- byte order, opaque padding and alignment;
- counted-array bounds and checked arithmetic before allocation or walking;
- what bytes are consumed before a length or discriminator gate fails;
- reserve-size and page/segment-boundary calculations; and
- error/status preservation at each decoder and dispatch boundary.

Generated code and native C layouts are not the format definition.  Do not
derive a wire size from `sizeof(struct ...)` unless the protocol explicitly
defines that native layout, and never let a successful compile stand in for
an interoperability run.  When client and peer must change together, name
the compatibility interval, the flag-day order, and how each mismatched pair
fails closed.

---

## Verification Output Requirements

For any kernel finding:

1. Name the execution context of the path: process / softirq / hardirq /
   NMI, RCU read-side, sleepable or atomic, locks held on entry.
2. State the sleep, allocation, or locking rule the patch violates, and
   the line that violates it.
3. For concurrency: state the ordering claim in LKMM terms and what a
   reader may observe, not in C11 terms.
4. For a finding supported by a run: state the artifact identity that was
   verified (checksum, anchor symbol, loaded module version) and how.  A
   result whose binary was not established is reported as an observation,
   not a finding.
5. For a config- or module-gated path: state the symbol and whether the
   test configuration reached it.
6. Route subsystem-specific questions to masoncl/review-prompts rather
   than inventing a local rule.
7. For test evidence: state the executed suite/case count and the
   handshake, injection hit or negative control that proves the intended
   edge was reached.
8. For publication or teardown: state what owns each reference, where new
   admissions close, what drains, and what event permits final destruction.
9. For cross-tree protocol evidence: state the exact kernel, peer and spec
   revisions, and distinguish compile/layout checks from a real wire run.
