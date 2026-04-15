# Async State Transfer Patterns

Load this file when CHANGE categories involve coroutine-style
suspension (task_pause / task_resume, green threads, stackful
coroutines, or any pattern where an operation is split across a
suspend point and a resume callback).

---

## 1. The Core Invariant: After Suspend Succeeds, Do Not Touch Transferred State

When a task/coroutine is suspended, another thread may resume and
free its state before the suspending thread returns from the suspend
call. There is no guaranteed ordering between the suspender's
next instruction and the resume callback.

**Wrong pattern:**
```c
task_pause(task);             /* suspend */
submit_async_io(rt->iobuf);  /* WRONG: rt may be freed */
return -EINPROGRESS;
```

**Correct pattern:**
```c
rt->rt_next_action = my_resume;

if (task_pause(task)) {
    /* task is suspended; another thread may resume it at any time */
    if (submit_async_io(rt->iobuf) < 0) {
        /* submit failed: must undo the suspend before accessing rt */
        task_resume(task);   /* cancels the suspend; safe to use rt now */
        ret = -EIO;
        goto out;
    }
    return -EINPROGRESS;    /* DO NOT touch rt after this */
}
/* task_pause returned false: suspend failed, proceed synchronously */
out:
    /* cleanup */
```

---

## 2. Transfer Ownership Before Suspending

Any state the resume callback needs must be moved out of the
suspending function's scope BEFORE the suspend call. After the
suspend call succeeds, the resume callback may run and free
everything before the suspending function returns.

**Wrong:**
```c
/* inode still owned by this stack frame */
went_async = task_pause(task);
if (went_async) {
    ph->ph_inode = inode;   /* WRONG: too late; task may be resumed */
    return -EINPROGRESS;
}
```

**Correct:**
```c
/* Transfer ownership FIRST, then suspend */
ph->ph_inode = inode;   inode = NULL;   /* transfer before pause */
ph->ph_sb    = sb;      sb    = NULL;

went_async = task_pause(task);
if (went_async) {
    if (submit_async_io(...) < 0) {
        /* Failed: restore ownership before exiting */
        task_resume(task);
        inode = ph->ph_inode;  ph->ph_inode = NULL;
        sb    = ph->ph_sb;     ph->ph_sb    = NULL;
        went_async = false;
        ret = -EIO;
        goto out;
    }
    return -EINPROGRESS;
}
/* Suspend failed: restore ownership */
inode = ph->ph_inode;  ph->ph_inode = NULL;
sb    = ph->ph_sb;     ph->ph_sb    = NULL;
```

**Why null out the local pointers:** Cleanup at `out:` must not
double-release resources. If ownership was transferred to `ph`,
the local pointer must be NULL so `inode_put(inode)` (if
NULL-tolerant) is a no-op.

---

## 3. The `went_async` Local Flag

When a resume callback can free the task's containing state, do not
dereference the task pointer after a successful suspend. Instead,
capture the suspend result in a local variable:

```c
bool went_async = false;

/* Transfer state to persistent storage before suspending */
ph->ph_inode = inode;  inode = NULL;

went_async = task_pause(task);
if (went_async) {
    if (submit(...) < 0) {
        task_resume(task);
        inode = ph->ph_inode;  ph->ph_inode = NULL;
        went_async = false;
        ret = -EIO;
        goto out;
    }
    return -EINPROGRESS;
}
inode = ph->ph_inode;  ph->ph_inode = NULL;

out:
    inode_put(inode);   /* NULL-safe: no-op if transferred */
    return ret;
```

**Why not check the task after suspend:** `rt->rt_task` is only
reachable through `rt`. If the resume completed before `out:` is
reached, `rt` is freed — dereferencing it to get `rt_task` is
use-after-free.

---

## 4. When NULL-Safe Cleanup Matters

After an ownership transfer, cleanup functions must tolerate NULL.
If they do not, guard the call:

```c
/* If put() is NULL-safe: */
inode_put(inode);       /* safe: checks NULL internally */
super_block_put(sb);    /* safe: checks NULL internally */

/* If put() is NOT NULL-safe: */
if (inode)
    inode_put(inode);
```

**Audit:** If an async path uses `goto out` and the function
transfers ownership to a persistent struct before suspending, verify
that all cleanup calls at `out:` are NULL-safe (or guarded).

---

## 5. Error Recovery After Suspend

If the async submission fails AFTER a successful `task_pause`:

1. Call `task_resume` to cancel the suspension.
2. Restore ownership from the persistent struct back to the local
   scope (reverse the transfer).
3. Set local pointers to NULL in the persistent struct to prevent
   double-release.
4. Fall through to the synchronous error path.

Skipping step 1 leaves the task permanently suspended (leaked).
Skipping step 2 leaves ownership in the persistent struct; when the
caller's `out:` label calls cleanup functions, ownership is split
and one path double-releases.

---

## 6. Resume Callbacks Must Not Access Caller's Stack

A resume callback runs in a different thread after `task_resume` is
called. It receives only the state that was explicitly transferred
(step 2 above) or stored in globally reachable structures. Accessing
the original caller's stack variables from a resume callback is
use-after-return.

```c
int my_op(struct rt *rt)
{
    int local_fd = open_file();

    ph->ph_fd = local_fd;   /* transfer, not share */
    local_fd = -1;

    if (task_pause(rt->rt_task)) {
        submit_io(ph->ph_fd, my_resume);
        return -EINPROGRESS;
    }
    /* ... */
}

static void my_resume(struct rt *rt)
{
    /* local_fd is gone -- correct, we use ph->ph_fd */
    process_result(rt->rt_ph->ph_fd);  /* correct */
}
```

---

## 7. Checklist for Code Review

When reviewing async state transfer code, verify:

1. **Ownership transferred before suspend?** All state the callback
   needs is in a persistent (heap) structure, with local pointers
   nulled out.
2. **Nothing accessed after successful suspend?** The line
   `return -EINPROGRESS` (or equivalent) is the last line in the
   suspend-succeeded branch.
3. **Submit failure handles undo?** If async submission fails after
   suspend, `task_resume` is called and ownership is restored.
4. **Cleanup is NULL-safe?** After ownership transfer, cleanup calls
   at `out:` tolerate NULL (or are guarded).
5. **Resume callback does not use caller's stack?** All data the
   resume needs was moved to the heap before the suspend.
6. **`went_async` flag used instead of re-reading task state?**
   The task's containing struct may be freed; the local bool is safe.

---

## Verification Output Requirements

For any async state transfer issue:
1. Show the ownership transfer sequence: what moved where, and when.
2. Show the point after which the caller must not access transferred
   state, and whether any access occurs after that point.
3. For cleanup issues: show the null-out and the `out:` cleanup path.
