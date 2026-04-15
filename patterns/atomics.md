# Atomics and Data Race Patterns

Load this file when CHANGE categories involve atomic operations, `_Atomic`,
`std::atomic`, `__atomic` builtins, `volatile` used for concurrent access,
or any variable accessed from multiple threads without a traditional lock.

---

## 1. Non-Atomic Access to a Shared Variable

Any variable accessed from multiple threads must be protected by either:
- A mutex or rwlock held on every access (read AND write).
- An atomic type (`_Atomic T` in C11, `std::atomic<T>` in C++11) with
  appropriate memory ordering.
- An RCU mechanism (see `patterns/rcu-and-lockless.md`).

**Plain reads and writes to a shared variable are a data race** — undefined
behavior in both C11 and C++11. A data race can produce any result, including
intermittent crashes and silent corruption.

```c
/* WRONG: two threads access 'counter' without synchronization */
static int counter;
void thread_a(void) { counter++; }
void thread_b(void) { counter++; }

/* CORRECT */
static _Atomic int counter;
void thread_a(void) { atomic_fetch_add(&counter, 1, memory_order_relaxed); }
void thread_b(void) { atomic_fetch_add(&counter, 1, memory_order_relaxed); }
```

---

## 2. `volatile` Is Not Atomic

`volatile` prevents the compiler from caching a variable in a register between
reads. It does NOT:
- Make reads or writes atomic.
- Provide memory ordering between threads.
- Replace a mutex.

Using `volatile int` for inter-thread communication is undefined behavior.
Use `_Atomic int` (C11) or `std::atomic<int>` (C++) instead.

**Exception:** `volatile` is correct for memory-mapped hardware registers, where
the hardware itself performs the I/O and the compiler must not cache the value.
Do not report `volatile` in driver-level hardware register code as a bug.

---

## 3. Memory Ordering: Choose the Right Order

C11 and C++11 atomics support five memory orderings. Using the wrong one
produces incorrect behavior.

| Ordering | Guarantees | Typical use |
|----------|-----------|-------------|
| `relaxed` | Atomicity only. No ordering wrt other memory ops. | Counters, statistics. |
| `acquire` | All prior writes by the releasing thread are visible after this load. | Read the flag, then read the data. |
| `release` | All prior writes by this thread are visible to a thread that does an acquire load. | Write the data, then set the flag. |
| `acq_rel` | Combines acquire and release. | Read-modify-write on a sync object. |
| `seq_cst` | Total order across all seq_cst operations. | Highest correctness, highest cost. |

**Common error — relaxed store of a ready flag:**
```c
/* Thread A writes data then signals it is ready */
data = compute();
atomic_store_explicit(&ready, 1, memory_order_relaxed);  /* WRONG */
/* Thread B may see ready=1 but stale data due to missing release */
```

**Correct (release-acquire pair):**
```c
/* Thread A */
data = compute();
atomic_store_explicit(&ready, 1, memory_order_release);   /* publishes data */

/* Thread B */
while (!atomic_load_explicit(&ready, memory_order_acquire)) {}
use(data);  /* data is now guaranteed to be visible */
```

---

## 4. Read-Modify-Write Operations Must Be Atomic

An increment (`x++`), decrement (`x--`), or bit-set on a shared variable is
three operations: read, modify, write. If two threads perform this simultaneously,
one update is silently lost.

```c
/* WRONG: lost update */
static int refcount;
void get(void) { refcount++; }

/* CORRECT */
static _Atomic int refcount;
void get(void) { atomic_fetch_add(&refcount, 1, memory_order_relaxed); }
```

---

## 5. Double-Checked Locking Without Memory Barrier

A classic pattern broken without proper ordering:

```c
/* WRONG: broken double-checked locking in C */
if (!instance) {
    pthread_mutex_lock(&mtx);
    if (!instance) {
        instance = create();  /* other thread may see partially constructed object */
    }
    pthread_mutex_unlock(&mtx);
}
use(instance);
```

Without a memory barrier between the store to `instance` and the store of its
fields, another thread may read `instance` as non-NULL but see uninitialized fields.

**Correct pattern:** Use `atomic_store` with `memory_order_release` when writing
the pointer and `atomic_load` with `memory_order_acquire` when reading it, or
use a mutex for the read as well.

In C++, use `std::call_once` or `static` local variable initialization (which
is guaranteed thread-safe since C++11) to implement singleton initialization.

---

## 6. Tearing on Wide Types

On 32-bit architectures (and sometimes on 64-bit platforms for 128-bit types),
reads and writes to wide types may not be atomic at the hardware level — a write
of a 64-bit value may be split into two 32-bit stores. Another thread reading
the value between the two stores sees a "torn" (corrupted) value.

**Rule:** Use `_Atomic` / `std::atomic` for any type accessed from multiple
threads, not just `int`.

---

## 7. `uatomic_xchg` / `__atomic_exchange` for Exclusive Ownership Transfer

Atomic exchange (`xchg`) is commonly used to claim exclusive ownership of a
pointer from a shared field:

```c
ptr = uatomic_xchg(&shared_ptr, NULL);
if (ptr) {
    /* we exclusively own ptr; no other thread can get it */
}
```

**Verify:**
- Only one consumer of the exchange result takes ownership.
- The producer sets the field before any consumer can observe it.
- All memory ordering is correct (typically `acq_rel` or `seq_cst` for exchange).

---

## 8. Atomic Operations Are Not Composable Without a Lock

Two separate atomic operations are NOT atomic together:

```c
/* WRONG: load and store are individually atomic, but NOT together */
int v = atomic_load(&x);
atomic_store(&x, v + 1);  /* another thread may have modified x between these */

/* CORRECT: use atomic fetch-add */
atomic_fetch_add(&x, 1, memory_order_relaxed);
```

If you need to read-then-conditionally-write atomically, use
`atomic_compare_exchange` (CAS).

---

## Verification Output Requirements

For any atomics or data race issue:
1. Identify the shared variable: name, type, and location.
2. Show the concurrent access sites (both threads): file:line.
3. For memory ordering issues: state what ordering is used and what ordering
   is required, with an explanation of what can go wrong with the current ordering.
4. For torn reads/writes: state the type width and the target architecture.
