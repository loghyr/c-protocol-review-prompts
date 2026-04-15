# RCU and Lock-Free Patterns

Load this file when CHANGE categories involve RCU (Read-Copy-Update),
seqlocks, lock-free data structures, or any access to data without traditional
locking.

---

## 1. RCU Core Pattern: Remove Before Free

**The invariant:** When removing an object from an RCU-protected data structure,
the object must be removed from the structure FIRST, then the RCU grace period
must elapse, THEN the object may be freed.

**Correct order:**
```
1. Remove object from data structure (list, hash table, tree)
2. Call synchronize_rcu() [or schedule call_rcu() + free in callback]
3. Free the object
```

**WRONG pattern — use-after-free:**
```c
/* WRONG: removal inside RCU callback */
void my_callback(struct rcu_head *rcu) {
    struct obj *o = container_of(rcu, struct obj, rcu);
    remove_from_hash(o);   /* WRONG: readers may already have found it */
    free(o);
}
```

When `call_rcu()` or `synchronize_rcu()` is seen:
- Immediately check: does removal from the data structure happen BEFORE or
  AFTER the RCU call?
- If removal is in the RCU callback → flag as use-after-free (BLOCKER).

---

## 2. RCU Reader-Side Must Not Block

Within an RCU read-side critical section (`rcu_read_lock()` / `rcu_read_unlock()`
or equivalent), the reader must not block, sleep, or call any allocator that
can block.

```c
rcu_read_lock();
ptr = rcu_dereference(global_ptr);
/* WRONG: sleeping inside RCU read-side */
sleep(1);
rcu_read_unlock();
```

**Context check:** In kernel contexts, `rcu_read_lock` also disables preemption
in some configurations. In userspace RCU libraries (liburcu), the semantics
depend on the chosen flavor. Verify the specific RCU implementation before
reporting.

---

## 3. RCU Dereference: Must Use Proper Accessor

Accessing RCU-protected pointers requires the appropriate dereferencing macro
or function to ensure correct memory ordering:

- In Linux kernel: `rcu_dereference()` inside read-side, `rcu_assign_pointer()`
  on the writer side.
- In liburcu: `rcu_dereference()` / `rcu_assign_pointer()` equivalents.

Direct pointer dereference without the accessor is a data race.

```c
/* WRONG: direct access without rcu_dereference() */
ptr = global_rcu_ptr;     /* data race: compiler may cache across barriers */
use(ptr->field);

/* CORRECT */
rcu_read_lock();
ptr = rcu_dereference(global_rcu_ptr);
use(ptr->field);
rcu_read_unlock();
```

**Exception:** When the data structure is protected by a lock that is currently
held, `READ_ONCE()` or plain access may be acceptable because the lock provides
the necessary serialization. Verify the specific convention.

---

## 4. RCU Grace Period and New Thread Registration

Threads that participate in RCU must register with the RCU runtime before
performing RCU read-side operations. In liburcu, this means calling
`rcu_register_thread()` at thread start and `rcu_unregister_thread()` at exit.

New threads created after initialization — including worker threads, thread
pools, and background tasks — must register if they use RCU. Check:
- Does the new thread call `rcu_register_thread()` at startup?
- Is `rcu_unregister_thread()` called on all exit paths?

---

## 5. Seqlock: Reader Must Re-Check

Seqlock readers must re-read the sequence counter after reading the data and
retry if the counter changed, indicating a concurrent write.

```c
/* WRONG: read data without verifying sequence */
seq = read_seqcount_begin(&sl);
val = shared_data;
/* no retry check → stale data */

/* CORRECT */
do {
    seq = read_seqcount_begin(&sl);
    val = shared_data;
} while (read_seqcount_retry(&sl, seq));
```

---

## 6. Lock-Free List / Stack Operations: ABA Problem

Lock-free stacks and lists using compare-and-swap (CAS) are subject to the
ABA problem: a pointer is removed, a new object is allocated at the same
address, and the CAS succeeds incorrectly.

**Detection:** Look for `__atomic_compare_exchange` / `std::atomic::compare_exchange`
on pointer values where the pointer could be recycled. Check whether the
implementation uses a version/tag counter to prevent ABA.

---

## 7. Memory Ordering for Lock-Free Code

Lock-free access to shared data requires explicit memory ordering. Plain reads
and writes may be reordered by the compiler or CPU.

| Requirement | C11 / C++11 construct |
|-------------|----------------------|
| Prevent all reordering | `memory_order_seq_cst` |
| Ensure prior writes visible before this read | `memory_order_acquire` |
| Ensure this write visible before later reads | `memory_order_release` |
| Producer–consumer pairing (release + acquire) | common for flag/pointer publish |
| Plain read of atomic (no ordering needed) | `memory_order_relaxed` |

**Common error:** Using `memory_order_relaxed` for a flag that signals data
is ready, when the reader does not use `memory_order_acquire`. The reader may
see the flag set but not the data.

---

## 8. Publish-Subscribe Pattern: Ordering Is Not Optional

When thread A writes data and then sets a flag for thread B to read the data:

```c
/* Thread A */
data = compute();
atomic_store(&ready, 1, memory_order_release);  /* publishes data */

/* Thread B */
while (!atomic_load(&ready, memory_order_acquire)) {}
use(data);  /* sees data because of release-acquire pair */
```

Without the release/acquire pair, the compiler or CPU may reorder the stores
and loads, and thread B may use stale data even after seeing the flag set.

---

## 9. call_rcu Callbacks Must Not Call synchronize_rcu

`call_rcu` callbacks run in the RCU callback thread. Calling
`synchronize_rcu()` from inside a callback deadlocks: `synchronize_rcu`
waits for all outstanding callbacks to complete, including the one that
called it.

```c
/* WRONG: deadlock */
static void obj_rcu_free(struct rcu_head *head)
{
    struct obj *obj = caa_container_of(head, struct obj, rcu_head);
    synchronize_rcu();   /* WRONG: waits for this callback to finish */
    free(obj);
}
```

The correct pattern: free directly in the callback (the grace period
has already elapsed) or schedule further deferred work with another
`call_rcu`.

```c
/* CORRECT */
static void obj_rcu_free(struct rcu_head *head)
{
    struct obj *obj = caa_container_of(head, struct obj, rcu_head);
    free(obj);  /* grace period elapsed; no extra synchronize needed */
}
```

**Detection:** Grep for `synchronize_rcu` or `rcu_barrier` inside
functions that are registered as `call_rcu` callbacks.

---

## 10. Iterator Must Advance Before put() (Hash Table Traversal)

When traversing an RCU-protected lock-free hash table and dropping
references inside the loop, the iterator must advance to the next node
BEFORE calling `put()`. The reason: `put()` can reach refcount zero,
triggering the release callback, which calls `cds_lfht_del` on the
current node. After deletion, `cds_lfht_next` from the deleted node
produces undefined behavior.

```c
/* WRONG: put() may delete node, then next() walks deleted node */
rcu_read_lock();
cds_lfht_first(ht, &iter);
while ((node = cds_lfht_iter_get_node(&iter)) != NULL) {
    obj = caa_container_of(node, struct obj, ht_node);
    obj_put(obj);                /* may delete node */
    cds_lfht_next(ht, &iter);   /* WRONG: iterating from deleted node */
}
rcu_read_unlock();

/* CORRECT: advance BEFORE put */
rcu_read_lock();
cds_lfht_first(ht, &iter);
while ((node = cds_lfht_iter_get_node(&iter)) != NULL) {
    obj = caa_container_of(node, struct obj, ht_node);
    cds_lfht_next(ht, &iter);   /* advance first */
    obj_put(obj);               /* now safe to trigger release */
}
rcu_read_unlock();
```

This pattern applies to any loop that drops the creation ref on the
current entry (reaper threads, shutdown drain, bulk revoke).

---

## Verification Output Requirements

For any RCU or lock-free issue:
1. Quote the remove/free sequence (for RCU): show the order of removal, grace
   period, and free.
2. For races: show two concurrent code paths with file:line references.
3. For memory ordering: state what ordering is used at each access site and
   why it is insufficient.
