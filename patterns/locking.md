# Locking Patterns

Load this file when CHANGE categories involve mutex acquisition, spinlock,
rwlock, condition variable, or any other synchronization primitive.

---

## 1. Lock Ordering (ABBA Deadlock)

**The invariant:** When multiple locks are acquired simultaneously, all code
paths must acquire them in the same order. Acquiring them in different orders
on different code paths creates a deadlock cycle (ABBA deadlock).

**Detection:** Find every site that holds lock A and then acquires lock B.
Check whether any other code path holds B and then acquires A.

```c
/* Path 1: A → B */
pthread_mutex_lock(&lock_a);
pthread_mutex_lock(&lock_b);  /* acquires B while holding A */

/* Path 2: B → A — ABBA deadlock */
pthread_mutex_lock(&lock_b);
pthread_mutex_lock(&lock_a);  /* acquires A while holding B */
```

**Verification:** Trace the full lock chain from entry point. Locks held by
callers count. A function that acquires lock B is dangerous if any caller
holds lock A and there exists a path that acquires A under B.

**Bidirectional check:** If the patch changes lock acquisition order in one
function, check all other functions that acquire the same pair of locks. The
patch may create the ABBA even if the changed function itself looks correct.

---

## 2. Sleep Under Non-Recursive Lock / Spinlock

**The invariant:** Blocking operations (I/O, `malloc`, `sleep`, `pthread_cond_wait`,
`sem_wait`) must not be called while holding a spinlock or a non-recursive mutex
in a context where the lock is not re-entrant.

**Also applies to:** Signal handlers and interrupt handlers must not acquire
blocking locks.

**Detection:**
- Find every call to a blocking function inside a lock critical section.
- Check whether the lock type permits blocking (recursive mutex → generally
  OK; spinlock → never OK; non-recursive mutex → check for recursion).

```c
pthread_spin_lock(&spin);
ptr = malloc(n);     /* WRONG: malloc may block waiting for memory */
pthread_spin_unlock(&spin);
```

---

## 3. Lock Not Held on Access (Missing Lock)

**The invariant:** Shared mutable data must be accessed only while the lock
that protects it is held (or via appropriate atomic/lock-free mechanism).

**Verification checklist:**
1. Identify the lock that guards the data structure.
2. Check every access to the data (read AND write) to confirm the lock is held.
3. Trace 2-3 levels up in the call stack — the lock may be acquired by a caller,
   not at the immediate access site.
4. Do not report a missing lock if the access is under a lock held by a caller.

**Common false positive:** A helper function that accesses shared data without
taking a lock is often correct because all callers hold the lock before calling it.
Always check callers before reporting.

---

## 4. Double Lock (Deadlock by Recursive Acquisition)

A non-recursive mutex acquired twice on the same thread deadlocks.

**Detection:** Find every path where the same mutex is acquired while already
held. This can happen through recursive calls, callbacks, or function pointers
invoked inside critical sections.

```c
void callback(void) {
    pthread_mutex_lock(&mtx);  /* DEADLOCK if caller holds mtx */
    ...
    pthread_mutex_unlock(&mtx);
}

void process(void) {
    pthread_mutex_lock(&mtx);
    invoke_callback(callback);  /* WRONG: callback reacquires mtx */
    pthread_mutex_unlock(&mtx);
}
```

**Note:** Check all function pointers, callbacks, and virtual dispatch called
within a critical section. They may be overridden to acquire the same lock.

---

## 5. Lock Released on Wrong Path (Lock Leak)

Every lock acquisition must be matched by exactly one release on every exit path
(including error paths, early returns, and goto).

**Detection:**
- Count lock/unlock pairs for each code path.
- Check every `return`, `goto`, `break`, and `continue` inside a critical
  section to confirm the lock is released before exit.

**Pattern to watch:**
```c
pthread_mutex_lock(&mtx);
if (error_condition) {
    return -1;              /* WRONG: lock not released */
}
pthread_mutex_unlock(&mtx);
```

**Correct pattern:**
```c
pthread_mutex_lock(&mtx);
if (error_condition) {
    pthread_mutex_unlock(&mtx);
    return -1;
}
pthread_mutex_unlock(&mtx);
```

Or use RAII (`std::lock_guard`, `std::unique_lock`) in C++ to guarantee release.

---

## 6. TOCTOU (Time-Of-Check to Time-Of-Use)

When a condition is checked and then a decision is made based on that check,
but the condition can change between the check and the use, you have a TOCTOU.

```c
/* Check */
if (list_has_item(list, key)) {
    /* Time passes; another thread removes the item */
    item = list_get(list, key);  /* item may be NULL or invalid */
}
```

**Correct pattern:** Hold the lock across both the check and the use, or use
a single atomic operation that both checks and retrieves.

```c
pthread_mutex_lock(&list->lock);
item = list_get(list, key);     /* atomic lookup-and-get */
pthread_mutex_unlock(&list->lock);
if (item) { ... }
```

---

## 7. Condition Variable Correctness

`pthread_cond_wait` must always be called in a loop that re-checks the
predicate, because spurious wakeups can occur.

```c
/* WRONG: single if check */
if (!queue_empty(&q)) {
    pthread_cond_wait(&q.cond, &q.lock);
}

/* CORRECT: loop until predicate is true */
while (queue_empty(&q)) {
    pthread_cond_wait(&q.cond, &q.lock);
}
```

**Deadlock from missing signal:** If one thread waits on a condition and
another thread that would signal it cannot reach the signal point (blocked on
the same lock, terminated, etc.), the wait is a deadlock. Check:
- Does a signal/broadcast always happen when the predicate becomes true?
- Is there a timeout? If not, can the wake condition fail to occur?

---

## 8. Read-Write Lock Upgrade (Upgrade Deadlock)

Upgrading a read lock to a write lock while still holding the read lock can
deadlock if another thread holds a read lock simultaneously (most rwlock
implementations do not support atomic upgrade).

```c
/* WRONG */
pthread_rwlock_rdlock(&rwl);
/* ... */
pthread_rwlock_wrlock(&rwl);  /* deadlock if any other reader exists */
```

**Correct pattern:** Release the read lock before acquiring the write lock.
Re-check the predicate after acquiring the write lock (state may have changed).

---

## 9. Lock Granularity Changes

If the patch changes lock granularity (coarse → fine, or fine → coarse):

- **Coarsening:** Check that data that was previously protected by its own lock
  is now correctly protected by the coarser lock everywhere it is accessed.
- **Splitting:** Check that no code path accesses two fine-grained locks in an
  inconsistent order. A split from one lock to two is a common source of new
  ABBA deadlocks.

---

## 10. Cooperative Thread Cancellation (Stop-Flag Pattern)

Multi-threaded programs that need clean shutdown typically use a shared
`_Atomic` stop flag: worker threads periodically check the flag and exit
cooperatively rather than being forcibly killed.

**The invariant:** A worker thread must check the stop flag BEFORE every
write or mutation — not merely before blocking on a condvar or returning
from the main loop. An operation that starts after the shutdown signal
leaves orphaned state that the cleanup path may not handle.

```c
/* WRONG: check only at the top of the loop */
void *worker(void *arg)
{
    while (!atomic_load_explicit(&g_stop, memory_order_relaxed)) {
        data = compute();
        write_to_filesystem(data);   /* proceeds even during shutdown */
        commit_metadata(data);
    }
    return NULL;
}

/* CORRECT: check before each mutation */
void *worker(void *arg)
{
    while (!atomic_load_explicit(&g_stop, memory_order_relaxed)) {
        data = compute();
        if (atomic_load_explicit(&g_stop, memory_order_relaxed))
            break;
        write_to_filesystem(data);
        if (atomic_load_explicit(&g_stop, memory_order_relaxed))
            break;
        commit_metadata(data);
    }
    return NULL;
}
```

**Notification discipline:** Setting the stop flag alone does not wake
threads blocked in `pthread_cond_wait`, `read()`, `poll()`, or similar.
The setter must also send a wake-up signal:
- `pthread_cond_broadcast`: for threads waiting on a condvar.
- `eventfd` write of 1: for threads blocked in `poll`/`epoll`/`select`.
- `write` to a pipe/socket: for threads blocked in `read`.

Missing the wake-up causes threads to hang until the next naturally
occurring event (which may never come), producing a stuck shutdown.

**Memory ordering:** `memory_order_relaxed` is acceptable for reading
the stop flag in a polling loop — the flag is a hint and the next
iteration will re-check. The write that sets the stop flag should use
at least `memory_order_release` so that all preceding writes are visible
before any thread acts on the stop signal.

**Per-thread state:** Each worker thread must have its own mutable
state (RNG seed, I/O buffer, error counter). Sharing mutable per-thread
state across threads without synchronization is a data race even if each
thread only reads `g_stop` atomically.

**Detection:** Look for write/create/delete/fsync calls inside a loop
that also checks a stop flag. Verify the stop check occurs BEFORE each
mutation, not only at the loop top or at blocking points.

---

## Verification Output Requirements

For any locking issue, produce:
1. The lock name and where it is defined.
2. The code path showing the acquisition.
3. The code path showing the problem (missing release, wrong order, etc.).
4. Two or more concurrent paths for race or deadlock issues.
