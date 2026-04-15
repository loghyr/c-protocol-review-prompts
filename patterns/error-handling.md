# Error Handling and Return Value Checking

Load this file when CHANGE categories involve system calls, I/O
operations, library calls, or any function whose return value
indicates success or failure.

---

## 1. Unchecked System Call Returns

Every POSIX system call that can fail returns -1 (or NULL) and sets
`errno`. Ignoring the return value silently continues on error, often
leading to use of invalid state.

```c
/* WRONG: return value ignored */
open(path, O_RDWR);
read(fd, buf, n);
write(fd, buf, n);

/* CORRECT */
fd = open(path, O_RDWR);
if (fd < 0) {
    return -errno;
}
```

**Common unchecked calls:** `open`, `close`, `read`, `write`,
`ftruncate`, `fsync`, `fdatasync`, `rename`, `unlink`, `mkdir`,
`stat`, `fstat`, `mmap`, `munmap`, `ioctl`, `bind`, `connect`,
`accept`, `sendmsg`, `recvmsg`, `pthread_create`, `pthread_join`.

**Detection:** Look for call statements (not assignments) to
functions that return a meaningful status. Look for assigned return
values that are never tested before use.

---

## 2. Partial I/O: Short Read and Short Write

`read()` and `write()` on files, pipes, sockets, and character
devices may transfer fewer bytes than requested without error:

- `read()`: a short read is not an error; it can occur at end-of-file,
  on signals (if `SA_RESTART` is not set), or when fewer bytes are
  available.
- `write()`: a short write is not an error; it occurs when the kernel
  buffer is full (pipes, sockets) or when the call is interrupted.

**Wrong — assumes full transfer:**
```c
read(fd, buf, sizeof(buf));
write(fd, buf, sizeof(buf));
```

**Correct — loop until complete:**
```c
ssize_t n, total = 0;
while (total < (ssize_t)sizeof(buf)) {
    n = read(fd, buf + total, sizeof(buf) - total);
    if (n < 0) {
        if (errno == EINTR) continue;
        return -errno;  /* real error */
    }
    if (n == 0)
        return -EIO;    /* unexpected EOF */
    total += n;
}
```

**Exception:** Regular file reads on Linux with no concurrent
truncation and a sufficiently large buffer will return the requested
count or EOF. A short read on a regular file is still correct to
handle defensively. For sockets and pipes, the loop is mandatory.

---

## 3. EINTR: Interrupted System Calls

Any blocking system call can be interrupted by a signal and return
-1 with `errno == EINTR`. The call made no progress and must be
retried from the beginning.

```c
/* WRONG: EINTR propagated as a fatal error */
if (read(fd, buf, n) < 0)
    return -errno;   /* -EINTR returned to caller */

/* CORRECT: retry on EINTR */
ssize_t ret;
do {
    ret = read(fd, buf, n);
} while (ret < 0 && errno == EINTR);
if (ret < 0)
    return -errno;
```

**Note:** If `sigaction` is used with `SA_RESTART`, many (but not
all) system calls are automatically restarted. The calls that are
NOT automatically restarted even with `SA_RESTART` include:
`nanosleep`, `ppoll`, `pselect`, `sigtimedwait`, `io_uring_enter`.
Always check the man page for the specific call.

---

## 4. EAGAIN / EWOULDBLOCK on Non-Blocking I/O

Non-blocking file descriptors (opened with `O_NONBLOCK` or set via
`fcntl(fd, F_SETFL, O_NONBLOCK)`) return -1 with `errno == EAGAIN`
(or `EWOULDBLOCK` on some platforms; they are equal on Linux) when
no data is available or the buffer is full. This is not an error —
it means "try again later."

```c
/* WRONG: EAGAIN treated as fatal */
if (send(sock, buf, n, 0) < 0)
    goto error;

/* CORRECT: distinguish EAGAIN from real errors */
ssize_t ret = send(sock, buf, n, 0);
if (ret < 0) {
    if (errno == EAGAIN || errno == EWOULDBLOCK)
        return 0;   /* or enqueue for retry */
    return -errno;  /* real error */
}
```

---

## 5. close() Errors Indicate Data Loss

`close()` can return -1 with `errno == EIO` or other errors when:
- The underlying write back of dirty pages failed (NFS, CIFS).
- The kernel was unable to flush buffered writes.

Ignoring `close()` errors silently discards write failures that
would otherwise go unreported.

```c
/* WRONG: silently loses I/O errors on close */
write(fd, data, len);
close(fd);

/* CORRECT */
if (write(fd, data, len) < 0) {
    int err = errno;
    close(fd);   /* best-effort, error already captured */
    return -err;
}
if (close(fd) < 0)
    return -errno;
```

**Note:** `close()` must be called exactly once per file descriptor
regardless of whether `write()` succeeded. Do not skip `close()` on
write error — that leaks the file descriptor.

---

## 6. EOF vs. Error: Distinguishing read() Return Values

`read()` returns three distinct outcomes:

| Return value | Meaning |
|-------------|---------|
| `> 0` | Bytes transferred (may be fewer than requested) |
| `0` | End of file (no more data) |
| `< 0` | Error; check `errno` |

Treating 0 as success and continuing to use the buffer is a bug:

```c
/* WRONG: EOF treated as success */
if (read(fd, buf, n) < 0)
    return -errno;
process(buf);  /* buf may be uninitialized if EOF returned 0 */

/* CORRECT */
ssize_t ret = read(fd, buf, n);
if (ret < 0)  return -errno;
if (ret == 0) return -EIO;     /* or -ENODATA, or EOF sentinel */
process(buf, ret);
```

---

## 7. errno Is Overwritten by Subsequent Calls

`errno` reflects the error from the most recent failing system call.
Any intervening call — even one that succeeds — may reset it. Save
`errno` immediately after a failing call if you need it later.

```c
/* WRONG: errno may be clobbered before it is read */
if (write(fd, buf, n) < 0) {
    log_message("write failed");  /* may call write() or printf() internally */
    return -errno;                /* WRONG: errno may have changed */
}

/* CORRECT */
if (write(fd, buf, n) < 0) {
    int saved = errno;
    log_message("write failed");
    return -saved;
}
```

---

## 8. fsync / fdatasync Before rename for Crash Safety

Writing data to a file and renaming it into place is a common
atomic-update pattern. However, if the writes are not synced to disk
before the rename, a power failure after the rename but before the
kernel flushes write-back can leave the new file with zero-length or
partial content.

```c
/* WRONG: rename before sync */
write(fd, data, len);
close(fd);
rename(tmp, final);

/* CORRECT: sync before rename */
write(fd, data, len);
fdatasync(fd);  /* flush data to disk */
close(fd);
rename(tmp, final);
```

**Check return values of all three:** If `write`, `fdatasync`, or
`close` fails, do not call `rename`. If `rename` fails, the old file
is still intact.

---

## 9. Ignoring Constructor / Initialization Return Values

Initialization functions that can fail are often called for their
side effect, and their return value is silently dropped:

```c
/* WRONG */
pthread_mutex_init(&mtx, NULL);  /* can fail with ENOMEM */
pthread_create(&tid, NULL, fn, arg);  /* can fail */

/* CORRECT */
if ((ret = pthread_mutex_init(&mtx, NULL)) != 0) {
    errno = ret;
    return -ret;
}
```

**Detection:** Look for calls to `pthread_*_init`, `pthread_create`,
`sem_init`, `pipe`, `socketpair`, `posix_memalign`, and similar
functions used as statements without capturing the return value.

---

## 10. Ignoring the Return of Functions That Can Partially Succeed

Some library functions return the count of items processed, not a
binary success/failure. Ignoring the count means ignoring partial
failure:

```c
/* fwrite returns the number of items written, not bytes */
fwrite(buf, sizeof(*buf), n, fp);   /* WRONG: may have written < n items */

size_t written = fwrite(buf, sizeof(*buf), n, fp);
if (written < (size_t)n)
    return -EIO;   /* short write */
```

**Also:** `fread`, `scanf`, `sscanf`, `fscanf` return the number of
items successfully read. A return value less than expected means
parse failure or EOF:

```c
if (sscanf(line, "%u %u", &a, &b) != 2)
    return -EINVAL;   /* not enough fields */
```

---

## 11. Unchecked Allocation Followed by Immediate Use

Even if the allocation null-check is present, some code checks the
result late, after already using the pointer:

```c
/* WRONG: ptr used before null check */
ptr = malloc(n);
ptr->field = 0;     /* crash if malloc returned NULL */
if (!ptr) return -ENOMEM;

/* CORRECT: check immediately */
ptr = malloc(n);
if (!ptr) return -ENOMEM;
ptr->field = 0;
```

---

## Verification Output Requirements

For any error handling issue:
1. Show the call site and the return value that is ignored or
   mishandled (file:line).
2. For partial I/O: describe the scenario in which the short
   transfer occurs (signal, full buffer, EOF) and what the caller
   does with the incomplete data.
3. For `errno` clobbering: show the call between the failure and the
   `errno` read that may reset it.
4. For crash-safety issues (§8): show the write, the missing sync,
   and the rename.
