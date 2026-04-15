# Clock Selection Patterns

Load this file when CHANGE categories involve time, timers, timeouts,
leases, sleep calls, `pthread_cond_timedwait`, or timestamp storage.

---

## 1. Two-Clock Rule

Two clocks serve two different purposes. Using the wrong one is a
correctness bug:

| Use case | Clock | Reason |
|----------|-------|--------|
| Timers, timeouts, lease expiry, interval measurement | `CLOCK_MONOTONIC` | Immune to NTP / admin `date` adjustments |
| Persistent timestamps (file mtime, atime, ctime) | `CLOCK_REALTIME` | Must survive reboot; correlates with wall time |
| Logging and trace timestamps | `CLOCK_REALTIME` | Human-readable; correlates with external logs |
| `pthread_cond_timedwait` | `CLOCK_REALTIME` | POSIX default; must match `pthread_condattr_setclock` if changed |
| RPC / op latency measurement | `CLOCK_MONOTONIC` | Interval only; wall-clock not needed |

**Wrong pattern — timer with REALTIME:**
```c
struct timespec deadline;
clock_gettime(CLOCK_REALTIME, &deadline);
deadline.tv_sec += 30;
/* ntpd adjusts clock forward 60s -> timer fires 30s early */
while (work_pending) {
    pthread_cond_timedwait(&cond, &lock, &deadline);  /* WRONG clock */
}
```

**Correct pattern:**
```c
struct timespec deadline;
clock_gettime(CLOCK_MONOTONIC, &deadline);  /* immune to NTP */
deadline.tv_sec += 30;
/* ... */
```

---

## 2. NTP / Clock Adjustment Pitfall

`CLOCK_REALTIME` can jump forward or backward:

- **Jump forward**: A timer set 30s in the future fires immediately
  if NTP steps the clock forward 60s.
- **Jump backward** (rare but legal for large corrections): A timer
  may never fire or fire much later than intended.

This is not hypothetical — NTP can step the clock by hours on first
sync or after hibernation.

**Detection:** Look for `clock_gettime(CLOCK_REALTIME, ...)` where
the result is used for a timeout deadline (not a timestamp to store
in metadata or a log).

---

## 3. `gettimeofday` and `time()`

Both return wall time and are subject to NTP steps. Neither is
suitable for timeout computations.

```c
struct timeval tv;
gettimeofday(&tv, NULL);   /* WRONG for timeouts */

time_t t = time(NULL);     /* WRONG for timeouts */
```

Use `clock_gettime(CLOCK_MONOTONIC, ...)` instead.

---

## 4. Comparing Timestamps Across Clocks

Never compute a duration by subtracting a `CLOCK_REALTIME` value
from a `CLOCK_MONOTONIC` value or vice versa. The two clocks have
different epochs and may drift relative to each other.

```c
/* WRONG: mixed-clock subtraction */
struct timespec start, end;
clock_gettime(CLOCK_MONOTONIC, &start);
do_work();
clock_gettime(CLOCK_REALTIME, &end);     /* WRONG: different epoch */
uint64_t elapsed_ns = timespec_diff_ns(&end, &start);
```

---

## 5. `pthread_cond_timedwait` and Clock Attributes

The default `CLOCK_REALTIME` condition variable is vulnerable to
NTP steps during a wait. To use `CLOCK_MONOTONIC`:

```c
pthread_condattr_t attr;
pthread_condattr_init(&attr);
pthread_condattr_setclock(&attr, CLOCK_MONOTONIC);
pthread_cond_init(&cond, &attr);
pthread_condattr_destroy(&attr);

/* Now timedwait takes a CLOCK_MONOTONIC deadline */
struct timespec deadline;
clock_gettime(CLOCK_MONOTONIC, &deadline);
deadline.tv_sec += 5;
pthread_cond_timedwait(&cond, &mutex, &deadline);
```

**Check:** When reviewing a `pthread_cond_timedwait`, verify the
condvar was initialized with the same clock used to compute
`abstime`. A mismatch is a silent bug.

---

## 6. Converting Between Clocks for Cross-System Storage

When a wire protocol or persistent format carries a wall-clock
expiry (e.g., "expires at epoch second N"), but the receiver wants
a monotonic deadline:

```c
/* Snapshot both clocks at the same instant */
struct timespec rt_now, mono_now;
clock_gettime(CLOCK_REALTIME,  &rt_now);
clock_gettime(CLOCK_MONOTONIC, &mono_now);

int64_t wall_expire_ns  = wire_expire_sec * 1000000000LL;
int64_t wall_now_ns     = rt_now.tv_sec * 1000000000LL + rt_now.tv_nsec;
int64_t remaining_ns    = wall_expire_ns - wall_now_ns;

if (remaining_ns <= 0) {
    /* already expired */
    return -ETIME;
}

uint64_t mono_now_ns    = (uint64_t)mono_now.tv_sec * 1000000000ULL
                          + (uint64_t)mono_now.tv_nsec;
uint64_t mono_deadline  = mono_now_ns + (uint64_t)remaining_ns;
```

**Why both clocks at once:** Minimizes the window between the two
`clock_gettime` calls. Snapshot them in back-to-back calls before
any other work.

---

## 7. Monotonic Nanosecond Helpers

Many codebases provide a `now_ns()` helper returning a `uint64_t`
of `CLOCK_MONOTONIC` nanoseconds. Verify it uses `CLOCK_MONOTONIC`,
not `CLOCK_REALTIME`:

```c
static inline uint64_t now_ns(void)
{
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);    /* MUST be MONOTONIC */
    return (uint64_t)ts.tv_sec * 1000000000ULL + (uint64_t)ts.tv_nsec;
}
```

---

## 8. Duration-Based Sleep

`clock_nanosleep` with `CLOCK_MONOTONIC` and a relative offset
avoids the NTP pitfall for sleep operations:

```c
/* WRONG: sleep for 1 second using nanosleep (uses CLOCK_REALTIME) */
struct timespec ts = { .tv_sec = 1 };
nanosleep(&ts, NULL);  /* affected by NTP step */

/* CORRECT: sleep for 1 second with CLOCK_MONOTONIC */
struct timespec ts = { .tv_sec = 1 };
clock_nanosleep(CLOCK_MONOTONIC, 0 /* TIMER_RELTIME */, &ts, NULL);
```

---

## Verification Output Requirements

For any clock misuse:
1. State which clock is used and which should be used.
2. Identify the scenario where the wrong clock produces incorrect
   behavior (NTP step, reboot, hibernation).
3. For `pthread_cond_timedwait`: verify the condvar's clock
   attribute matches the `abstime` computation.
