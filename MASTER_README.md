# Pintos — Master README (Lab 1 + Lab 2)

A single place that explains this whole repository from A to Z: what Pintos
is, what we changed, how every piece works, how to build and test it, and
the questions a TA is likely to ask with the answers.

Read this top to bottom once. Afterwards use the table of contents to jump
to whatever the TA is asking about.

---

## Table of contents

1. [What Pintos is](#1-what-pintos-is)
2. [Repository map](#2-repository-map)
3. [Branches and commit history](#3-branches-and-commit-history)
4. [Toolchain, build, run, test](#4-toolchain-build-run-test)
5. [Background you need: threads, interrupts, kernel stacks](#5-background-you-need)
6. [Lab 1 — Threads and scheduling](#6-lab-1--threads-and-scheduling)
   - 6.1 Alarm clock (sleep without busy-waiting)
   - 6.2 Priority scheduling
   - 6.3 Priority donation
   - 6.4 MLFQS (advanced scheduler)
   - 6.5 Interactive kernel shell
7. [Lab 2 — User programs](#7-lab-2--user-programs)
   - 7.1 User mode vs kernel mode
   - 7.2 Process creation: exec → load → start
   - 7.3 Argument passing and the initial stack
   - 7.4 System calls: the `int 0x30` path
   - 7.5 Safe access to user memory
   - 7.6 File descriptors
   - 7.7 Parent/child lifetime: `exec`, `wait`, `exit`
   - 7.8 Deny writes to running executables
   - 7.9 Process exit and cleanup
8. [Data structures reference](#8-data-structures-reference)
9. [Bugs we hit and fixed](#9-bugs-we-hit-and-fixed)
10. [Known limitations](#10-known-limitations)
11. [TA question bank](#11-ta-question-bank)
12. [Where to look for each file](#12-where-to-look-for-each-file)

---

## 1. What Pintos is

Pintos is a small teaching operating system for 32-bit x86 written at
Stanford. It boots on real hardware or in an emulator (QEMU / Bochs) and
is just big enough to show how a real kernel works: threads, scheduling,
interrupts, virtual memory, a file system, and user programs.

The course gives you a skeleton that boots but is missing key features.
Each lab fills one gap:

| Lab | Directory | What you add |
|---|---|---|
| Lab 1 | `src/threads/` | A working scheduler: sleeping, priorities, donation, MLFQS |
| Lab 2 | `src/userprog/` | The ability to run user programs: loading, syscalls, processes |
| Lab 3 | `src/vm/` | Virtual memory (not started in this repo) |
| Lab 4 | `src/filesys/` | Real file system (not started) |

This repo is a fork of the Johns Hopkins CS 318 version (PintosM), adapted
for our course (CS 2043).

---

## 2. Repository map

```
pintos-os/
├── README.md             Upstream README (JHU)
├── LAB2_GUIDE.md         Narrative walkthrough of Lab 2
├── MASTER_README.md      This file
└── src/
    ├── threads/          Lab 1 — kernel threads, scheduler, synchronisation
    │   ├── thread.c/h    struct thread, scheduler, priority, MLFQS
    │   ├── synch.c/h     semaphores, locks, condition variables, donation hooks
    │   ├── fixed-point.h 17.14 fixed-point math macros for MLFQS
    │   ├── init.c        Kernel entry point (pintos_init) + our interactive shell
    │   ├── interrupt.c   Interrupt controller / handler registration
    │   ├── palloc.c      Page allocator (4 kB pages)
    │   ├── malloc.c      Small-object allocator built on palloc
    │   ├── switch.S      Context switch (assembly)
    │   └── vaddr.h       PHYS_BASE, is_user_vaddr(), page macros
    ├── userprog/         Lab 2 — user programs
    │   ├── process.c/h   exec / wait / exit, ELF loader, argument passing
    │   ├── syscall.c/h   System-call handler and user-memory checks
    │   ├── exception.c   Page faults and CPU exceptions from user code
    │   ├── pagedir.c/h   Page-directory helpers (map/unmap/lookup pages)
    │   ├── gdt.c, tss.c  x86 segment / task-state setup
    │   └── DESIGNDOC     Lab 2 design document (submitted)
    ├── devices/          Hardware drivers (unchanged except timer.c)
    │   └── timer.c       Lab 1 — alarm clock sleep list
    ├── filesys/          Simple file system (unchanged, used as-is by Lab 2)
    ├── lib/              C library for kernel and user programs
    │   ├── kernel/       list.c, hash.c, bitmap.c, console.c
    │   ├── user/         syscall.c (user-side wrappers), entry.c (_start)
    │   └── syscall-nr.h  Syscall numbers (SYS_HALT=0, SYS_EXIT=1, ...)
    ├── examples/         Sample user programs (echo, cat, ls, shell, ...)
    ├── tests/            Test suite (threads/, userprog/, filesys/base/)
    ├── utils/            pintos, pintos-mkdisk, pintos-gdb helper scripts
    └── misc/             Toolchain build scripts and patches
```

**Files we modified** (everything else is upstream):

| File | Lab | Change |
|---|---|---|
| `threads/thread.c` | 1, 2 | priority queue, donation, MLFQS, new `struct thread` fields init |
| `threads/thread.h` | 1, 2 | new fields, `struct child_info`, `struct file_desc` |
| `threads/synch.c` | 1 | priority-ordered waiters, donation on lock acquire/release |
| `threads/fixed-point.h` | 1 | new file |
| `threads/init.c` | 1 | interactive shell when no command line is given |
| `devices/timer.c` | 1 | sleep list, wake-up in timer interrupt |
| `userprog/process.c` | 2 | everything: exec, wait, exit, args, loader |
| `userprog/syscall.c` | 2 | everything: all 13 syscalls |
| `userprog/syscall.h` | 2 | exports `filesys_lock` |
| `userprog/DESIGNDOC` | 2 | design document |

---

## 3. Branches and commit history

```
lab1   ← Lab 1 work (main branch for PRs)
lab2   ← Lab 2 work, built on top of lab1   (current)
master ← untouched upstream
```

Commits, oldest first:

```
ff15c1b  Implement Pintos Lab 1 thread scheduler
18d77c0  Complete Lab 1 scheduler changes
eb07f38  Add Lab 2 process lifecycle state          (struct fields, init)
fde65d6  Implement Lab 2 user memory and system calls
34c52da  Fix Lab 2 kernel stack safety and preemption
7bf601e  Document Lab 2 user programs implementation
363a0d0  Fix child record free race and check argument stack size
5391596  Add Lab 2 design document
```

---

## 4. Toolchain, build, run, test

### 4.1 Requirements

- Linux (or WSL). The `i386-elf-gcc` cross-compiler, `i386-elf-gdb`,
  and `qemu-system-i386`. The scripts in `src/misc/toolchain-build.sh`
  build the cross-compiler if your distro doesn't ship one.
- `src/utils/` must be on your `PATH`:
  ```bash
  export PATH=$PATH:/path/to/pintos-os/src/utils
  ```

### 4.2 Build

Each lab builds from its own directory. The build output goes to a
`build/` subdirectory that is git-ignored.

```bash
# Lab 1
cd src/threads && make

# Lab 2 (this also builds filesys/ and the user examples it needs)
cd src/userprog && make
```

`make clean` removes `build/`.

### 4.3 Run a single thing by hand

```bash
# Lab 1: run one kernel test
cd src/threads/build
pintos --qemu -- -q run alarm-multiple

# Lab 2: make a disk, copy a user program in, run it
cd src/userprog/build
pintos-mkdisk filesys.dsk --filesys-size=2
pintos --qemu -- -f -q                        # format the disk once
pintos --qemu -p ../../examples/echo -a echo -- -q run 'echo hello world'
```

Flag meanings: `-f` format the file system, `-q` quit after the command,
`-p host -a name` copy a host file onto the Pintos disk as `name`,
`run 'cmd'` run that command line as a user program.

### 4.4 Run the test suite

```bash
cd src/threads/build && make check     # 27 Lab 1 tests
cd src/userprog/build && make check    # 80 Lab 2 tests (67 userprog + 13 filesys/base)
```

Run a single test (and see its output):

```bash
make tests/userprog/args-multiple.result
cat tests/userprog/args-multiple.output      # raw kernel output
cat tests/userprog/args-multiple.result      # PASS / FAIL and why
```

Each test is three files in `src/tests/...`: `name.c` (the user program or
kernel test), `name.ck` (Perl script that checks the output), and the
generated `.output` / `.result`.

Current status: **all 80 Lab 2 tests pass** under QEMU.

### 4.5 Debugging

```bash
pintos --qemu --gdb -- -q run alarm-multiple      # terminal 1, waits for gdb
pintos-gdb kernel.o                                # terminal 2
(gdb) target remote localhost:1234
(gdb) break thread_exit
(gdb) continue
```

`src/misc/gdb-macros` adds `dumplist`, `btthread`, `btthreadall` to look at
Pintos lists and thread stacks.

---

## 5. Background you need

Three facts explain most of the code in both labs.

### 5.1 A thread is one 4 kB page

```
   4 kB ┌──────────────────────────┐  ← t->stack starts here, grows down
        │     kernel stack          │
        │          ↓                │
        │                           │
        ├──────────────────────────┤
        │  struct thread            │  tid, status, name, priority, ...
   0 kB └──────────────────────────┘  magic (0xcd6abf4b)
```

`thread_current()` finds the thread by rounding `%esp` down to a page
boundary. If the kernel stack grows past the struct, `magic` is overwritten
and `thread_current()` asserts. **This is why we never put a 4 kB array on
the kernel stack** (see §9).

### 5.2 Thread states and the scheduler

```
            thread_create()
                 │
                 ▼
  ┌────────┐ schedule() ┌─────────┐
  │ READY  │ ─────────▶ │ RUNNING │
  └────────┘ ◀───────── └─────────┘
       ▲   thread_yield()     │
       │                      │ thread_block()  (sema_down, lock_acquire, timer_sleep)
       │ thread_unblock()     ▼
       │               ┌─────────┐
       └────────────── │ BLOCKED │
                       └─────────┘
                                  thread_exit() → DYING → freed by next thread
```

`ready_list` holds READY threads. `schedule()` pops the front and switches
to it. In Lab 1 we keep `ready_list` **sorted by priority** so the front
is always the highest-priority thread.

### 5.3 Interrupts and "interrupt context"

A timer interrupt fires every tick (100 Hz). Its handler runs on the
*current* thread's kernel stack with interrupts off. Inside a handler you
**must not block** (no `sema_down`, no `lock_acquire`). You can call
`thread_unblock()` and `intr_yield_on_return()` (= "yield as soon as the
handler finishes").

Turning interrupts off (`intr_disable()`) is the cheapest way to make a
short critical section atomic on a single CPU. We use it for the sleep list,
ready list, and the shared parent/child record.

---

## 6. Lab 1 — Threads and scheduling

### 6.1 Alarm clock

**Problem.** The original `timer_sleep()` busy-waits: it loops calling
`thread_yield()` until enough ticks pass, burning CPU.

**Solution.** Put the thread to sleep and let the timer interrupt wake it.

- `struct thread` gets `wake_tick` and `sleep_elem`.
- `timer_sleep(ticks)` ([timer.c](src/devices/timer.c)):
  1. compute `wake_tick = now + ticks`,
  2. with interrupts off, insert into `sleep_list` **sorted by wake_tick**,
  3. `thread_block()`.
- `timer_interrupt()` increments `ticks`, then pops every thread at the front
  of `sleep_list` whose `wake_tick <= ticks` and `thread_unblock()`s it. Because
  the list is sorted it stops at the first thread that isn't due yet, so the
  handler does O(k) work for k woken threads, not O(n).
- Finally `thread_check_preempt()` so a newly woken higher-priority thread
  runs immediately after the handler returns.

### 6.2 Priority scheduling

**Rule.** The highest-priority ready thread always runs. Priorities are
0 (`PRI_MIN`) .. 63 (`PRI_MAX`), default 31.

**Implementation.**
- `ready_list` is kept sorted: `thread_unblock()` and `thread_yield()` use
  `list_insert_ordered(..., priority_less, ...)` instead of `list_push_back`.
- Semaphore waiter lists are also sorted by priority (`sema_down`), so
  `sema_up` wakes the highest-priority waiter. Condition variables sort their
  waiters by the priority of the thread inside each semaphore
  (`cond_priority_less`).
- `thread_check_preempt()` compares the front of `ready_list` with the
  current thread and yields if the front is higher. It is called after every
  event that might create a higher-priority ready thread: `thread_create`,
  `sema_up`, `thread_set_priority`, timer wake-ups. It works both inside an
  interrupt handler (`intr_yield_on_return`) and outside (`thread_yield`).

### 6.3 Priority donation

**Problem (priority inversion).** High-priority H waits for a lock held by
low-priority L. Medium-priority M keeps running and L never gets the CPU
to release the lock, so H is stuck behind M.

**Solution.** While H waits, L temporarily inherits H's priority.

Fields: `base_priority` (what the thread asked for), `priority` (effective,
possibly donated), `waiting_lock` (the lock this thread is blocked on).

- `lock_acquire()` ([synch.c:202](src/threads/synch.c#L202)): set
  `waiting_lock = lock`, then `thread_donate(lock)` — walk the chain
  `lock → holder → holder->waiting_lock → its holder → ...` up to 8 levels,
  raising each holder's `priority` to ours if ours is higher. This handles
  **nested** donation (`priority-donate-nest`, `-chain`).
- `lock_release()`: `thread_lock_released(lock)` recomputes the releasing
  thread's priority with `refresh_priority()`, which scans `all_list` for
  threads whose `waiting_lock->holder == me` and takes the max of
  `base_priority` and all remaining donors. This handles **multiple**
  donations on different locks (`priority-donate-multiple`).
- `thread_set_priority()` sets `base_priority` and refreshes; if a donation
  is in effect the effective priority stays high until the lock is released
  (`priority-donate-lower`).
- Donation is skipped entirely when `thread_mlfqs` is on.

### 6.4 MLFQS (multi-level feedback queue scheduler)

Enabled with the kernel option `-o mlfqs`. Priorities are no longer set by
threads; the kernel computes them from how much CPU each thread has used.

Formulas (from the Pintos manual), computed with 17.14 fixed-point
arithmetic in [fixed-point.h](src/threads/fixed-point.h) because the kernel
has no floating point:

```
priority   = PRI_MAX - (recent_cpu / 4) - (nice * 2)        every 4 ticks, all threads
recent_cpu = (2*load_avg)/(2*load_avg + 1) * recent_cpu + nice   once per second
load_avg   = (59/60)*load_avg + (1/60)*ready_threads         once per second
recent_cpu += 1                                               every tick, running thread
```

All of this lives in `thread_tick()` ([thread.c:131](src/threads/thread.c#L131))
and the helpers `update_recent_cpu` / `update_mlfqs_priority`.
`ready_threads` counts the ready list plus the running thread, excluding idle.

### 6.5 Interactive kernel shell

If Pintos boots with no command line, `pintos_init()` calls
`run_interactive_shell()` ([init.c](src/threads/init.c)). It reads keys with
`input_getc()`, handles backspace, splits the line with `strtok_r`, and
supports `echo`, `peek <hex-addr>`, `whoami`, `clear`, `shutdown`, `exit`.
This was an extra for Lab 1 and is unrelated to the test suites.

---

## 7. Lab 2 — User programs

### 7.1 User mode vs kernel mode

Lab 1 code all ran in the kernel at ring 0 with full access to everything.
A user program runs at ring 3: it cannot touch kernel memory
(addresses ≥ `PHYS_BASE` = `0xC0000000`), cannot do I/O, and cannot call
kernel functions directly. The only way in is a **system call**: the program
pushes arguments on its stack and executes `int $0x30`, which the CPU turns
into a trap into `syscall_handler()`.

Each process has its own **page directory** (`thread->pagedir`) mapping its
user addresses (`0` .. `PHYS_BASE`) to physical pages. The kernel half of the
address space is identical in every process. `process_activate()` switches
page directories on every context switch.

One process = one thread. `pid == tid`.

### 7.2 Process creation: exec → load → start

```
Parent                                   Child (new thread)
──────                                   ──────────────────
syscall exec("echo hi")
  copy_string → kernel page
  process_execute("echo hi")
    palloc page, strlcpy cmdline
    malloc child_info (load_sema=0, wait_sema=0)
    list_push_back(&children, child)
    thread_create("echo", start_process, info)
    sema_down(&child->load_sema)  ── blocks ──▶  start_process(info)
                                                   load("echo hi", &eip, &esp)
                                                     pagedir_create()
                                                     filesys_open("echo")
                                                     read ELF header, PT_LOAD segments
                                                     setup_stack(esp, "echo hi")
                                                     file_deny_write(file)
                                                   child->load_success = ok
    ◀──────────────────────────────────────────── sema_up(&child->load_sema)
    if !load_success: release record, return -1   free cmdline page, free info
    return tid                                     jmp intr_exit  → user mode, _start()
```

Key points to say to a TA:

- **The command line is copied** into a kernel page before the child starts,
  because the parent could change its buffer at any time.
- **`exec` blocks until load finishes** so it can return -1 on failure.
  `load_sema` is the hand-off.
- The thread name is the first word, truncated to 15 chars, so the exit
  message is `echo: exit(0)` and not `echo hi: exit(0)`.
- `struct start_info` exists only because `thread_create()` takes a single
  `void *aux`; we need to pass both the cmdline and the child record.

### 7.3 Argument passing and the initial stack

`setup_stack()` ([process.c:601](src/userprog/process.c#L601)) maps one zeroed
page at `PHYS_BASE - 4096` and builds this layout for `echo hello world`
(addresses grow upward, stack grows downward):

```
PHYS_BASE  0xC0000000
           ┌───────────────┐
  bffffffb │ "world\0"     │  ← copied last-first, so argv[0] string is lowest
  bffffff5 │ "hello\0"     │
  bffffff0 │ "echo\0"      │
  bfffffee │ (padding)     │  ← esp rounded down to multiple of 4
  bfffffec │ NULL          │  argv[3]  sentinel
  bfffffe8 │ 0xbffffffb    │  argv[2] → "world"
  bfffffe4 │ 0xbffffff5    │  argv[1] → "hello"
  bfffffe0 │ 0xbffffff0    │  argv[0] → "echo"
  bfffffdc │ 0xbfffffe0    │  argv    (address of argv[0])
  bfffffd8 │ 3             │  argc
  bfffffd4 │ 0             │  fake return address
           └───────────────┘  ← esp handed to the program
```

That is exactly the 80x86 calling convention for `main(int argc, char
**argv)`: `_start()` in `lib/user/entry.c` reads argc/argv off the stack,
calls `main`, then calls `exit(main's return value)`.

Why right-to-left? Pushing `argv[argc-1]` first and `argv[0]` last leaves
`argv[0]` at the lowest address, so the array reads in order when indexed
upward.

Overflow guard: before writing anything, we add up string bytes + 3 bytes of
alignment + `(argc + 4)` words. If that exceeds 4096, `setup_stack` fails and
`exec` returns -1. Arguments are also capped at 128 by the `tokens[]` array.

### 7.4 System calls: the `int 0x30` path

User side ([lib/user/syscall.c](src/lib/user/syscall.c)):
```c
int write (int fd, const void *buf, unsigned size)
{ return syscall3 (SYS_WRITE, fd, buf, size); }   // pushes args, int $0x30
```

Kernel side ([syscall.c:96](src/userprog/syscall.c#L96)):
```
syscall_handler(struct intr_frame *f)
  call = argument(f, 0)            // *(uint32_t*)f->esp, validated
  switch (call)
    case SYS_WRITE:
      fd     = argument(f, 1)
      buffer = argument(f, 2)
      size   = argument(f, 3)
      valid_range(buffer, size) or die
      fd==1 → putbuf(buffer,size)  else file_write(...) under filesys_lock
      f->eax = bytes written       // becomes the return value in user mode
```

`f->esp` is the **user** stack pointer at the moment of the trap; the
arguments sit just above it. `f->eax` is where the CPU restores `%eax` on
`iret`, so writing it sets the syscall's return value.

The 13 syscalls:

| Call | What it does | Error value |
|---|---|---|
| `halt` | `shutdown_power_off()` | — |
| `exit(status)` | store status, `thread_exit()` → prints `name: exit(status)` | — |
| `exec(cmd)` | copy cmd, `process_execute()` | -1 if load fails |
| `wait(pid)` | `process_wait()` | -1 if not child / already waited |
| `create(name, size)` | `filesys_create()` | false |
| `remove(name)` | `filesys_remove()` | false |
| `open(name)` | `filesys_open()`, new fd | -1 |
| `filesize(fd)` | `file_length()` | -1 |
| `read(fd, buf, n)` | fd 0 → keyboard; else `file_read()` | -1 |
| `write(fd, buf, n)` | fd 1 → console; else `file_write()` | -1 |
| `seek(fd, pos)` | `file_seek()` | no-op |
| `tell(fd)` | `file_tell()` | -1 |
| `close(fd)` | `file_close()`, free fd | no-op |

Every file-system call is wrapped in `lock_acquire(&filesys_lock)` /
`lock_release(...)` because the provided file system has no internal locking
and two processes could otherwise corrupt it.

### 7.5 Safe access to user memory

A user program can pass the kernel any pointer: `NULL`, a kernel address, an
unmapped address, or a buffer that runs off the end of a mapped page. The
kernel must **kill that process** (exit status -1) without crashing itself.

We use the "**check first, then dereference**" approach:

```c
valid_range(ptr, size)    // every byte: ptr != NULL
                          //             is_user_vaddr(ptr+i)        (below PHYS_BASE)
                          //             pagedir_get_page(pd, ptr+i) (actually mapped)
argument(f, i)            // valid_range(f->esp + 4*i, 4) then read the word
copy_string(user, kern, n)// byte-by-byte copy, validating each byte, until '\0'
terminate_bad_user()      // exit_status = -1; thread_exit()
```

Strings (file names, exec command lines) are always **copied into a kernel
page** before use, because file-system calls can block and the user could
change the string underneath us.

The alternative method (only check `< PHYS_BASE`, then let the page-fault
handler catch bad addresses with `get_user`/`put_user` assembly) is faster
but needs changes to `exception.c`. We chose the simple one; speed is not
graded.

### 7.6 File descriptors

Each process has `file_descriptors` (a list of `struct file_desc {fd, file,
elem}`) and `next_fd` starting at 2. `open` allocates a `file_desc`, assigns
`next_fd++`, and appends it. All other fd calls use `find_fd(fd)` to look it
up in the *current* thread's list.

- fd 0 = keyboard (`input_getc`), fd 1 = console (`putbuf`). Never handed out.
- fds are **per process**, not global, and **not inherited** by children.
- Opening the same file twice gives two independent `struct file`s with
  separate positions.
- `process_exit` closes every remaining fd.

### 7.7 Parent/child lifetime: `exec`, `wait`, `exit`

This is the hardest part and the one TAs probe most. The problem: a parent
may call `wait(child)` **before or after** the child exits, or never. The
child's exit status must survive the child's `struct thread` being freed, and
the shared record must be freed exactly once.

**Solution: a heap record with two owners.**

```c
struct child_info {
  tid_t tid;
  struct semaphore load_sema;   // exec hand-off (§7.2)
  struct semaphore wait_sema;   // upped once when child exits
  bool load_success, waited;
  bool parent_alive, child_alive;
  int  exit_status;
};
```

The parent holds it in `children`; the child points to it via
`child_record`. `release_child(c, is_parent)` clears the caller's `_alive`
flag and frees the record **only if both are now false**. The flag update and
the test are done with interrupts disabled, so the two sides can't race.

`process_wait(pid)`:
1. Find `pid` in **my** `children` list. Not found → -1. `waited` already → -1.
2. `waited = true; sema_down(&wait_sema)`.
3. Read `exit_status`, remove from list, `release_child(parent)`, return it.

`process_exit()` (child side): write `exit_status` into the record,
`sema_up(&wait_sema)`, `release_child(child)`. Also `release_child(parent)`
on every record in its own `children` list (it is abandoning its kids).

All four orderings:

| Order | What happens |
|---|---|
| P waits, then C exits | P blocks on `wait_sema`; C ups it; P wakes, reads status, frees |
| C exits, then P waits | `wait_sema` is already 1, `sema_down` returns immediately; P frees |
| P exits before C, no wait | P drops its ownership; C keeps running; C's exit frees |
| P exits after C, no wait | C already dropped; P's exit frees |

`wait` twice → -1 (the `waited` flag). `wait` on a grandchild or stranger →
-1 (not in my list). Failed `exec` → child exits normally, parent drops its
side in `process_execute`.

### 7.8 Deny writes to running executables

If someone could modify `echo` on disk while `echo` is running, the running
code would change under it. After a successful `load()` we keep the
`struct file *` in `thread->executable` and call `file_deny_write()`.
`process_exit()` calls `file_allow_write()` and closes it. Closing right
after load would have re-enabled writes too early — that was a real bug
(`rox-simple` test).

### 7.9 Process exit and cleanup

`thread_exit()` → `process_exit()` ([process.c:206](src/userprog/process.c#L206)),
which runs no matter *how* the process died (normal `exit`, bad pointer,
page fault, divide by zero):

1. `printf("%s: exit(%d)\n", name, exit_status)` — only for user processes.
2. Release every child record (parent side).
3. Publish status to own record, `sema_up(wait_sema)`, release (child side).
4. Close every open fd.
5. `file_allow_write` + close the executable.
6. Set `pagedir = NULL`, activate the kernel page directory, destroy ours.

Order matters: step 6 is last so file-system calls above still run with a
valid address space, and `pagedir` is nulled before switching so a timer
interrupt can't switch back into a page directory being destroyed.

Exceptions from user code (`exception.c`): `kill()` prints a message and
calls `thread_exit()`; because `exit_status` defaults to -1, the parent's
`wait` returns -1. Page faults do the same (no VM yet).

---

## 8. Data structures reference

New fields in `struct thread` ([thread.h:107](src/threads/thread.h#L107)):

| Field | Lab | Purpose |
|---|---|---|
| `base_priority` | 1 | Priority set by the thread itself |
| `priority` | 1 | Effective priority (may be donated) |
| `waiting_lock` | 1 | Lock this thread is blocked on (for donation chains) |
| `nice`, `recent_cpu` | 1 | MLFQS inputs |
| `sleep_elem`, `wake_tick` | 1 | Alarm clock |
| `children` | 2 | List of `child_info` for my direct children |
| `child_record` | 2 | My own `child_info`, shared with my parent |
| `exit_status` | 2 | Default -1; set by `exit()` |
| `user_process` | 2 | True after `start_process`; gates the exit message |
| `executable` | 2 | Open file with writes denied |
| `file_descriptors`, `next_fd` | 2 | Per-process fd table |
| `pagedir` | 2 (given) | Page directory |

Globals: `ready_list`, `all_list`, `sleep_list` (timer.c), `load_avg`,
`filesys_lock` (syscall.c).

---

## 9. Bugs we hit and fixed

Worth mentioning to a TA because they show understanding.

1. **Kernel stack overflow** (`34c52da`). An early version had
   `char buf[PGSIZE]` as a local in the syscall handler. 4 kB on a <4 kB
   kernel stack overwrote `struct thread` and tripped the `magic` assertion
   in `thread_current()` — far from the real cause. Fix: `palloc_get_page()`
   for every large temporary buffer.

2. **Child record double-free / use-after-free** (`363a0d0`). Early code
   freed the record in `process_wait` *and* in the child's `process_exit`.
   Fix: the two-owner `parent_alive` / `child_alive` scheme in
   `release_child()` with interrupts off.

3. **Stack-page overflow with many arguments** (`363a0d0`). Nothing stopped
   a 5 kB command line from writing below the stack page. Fix: size check
   before building the stack; `exec` returns -1.

4. **Preemption after `sema_up`** (`34c52da`). A woken higher-priority
   process didn't run until the next timer tick, which broke ordering in
   multi-process tests. Fix: `thread_check_preempt()` at the end of
   `sema_up()`.

5. **Writes to a running executable allowed** (`rox-*` tests). Closing the
   executable right after `load()` re-enabled writes. Fix: keep it open in
   `thread->executable` until `process_exit`.

---

## 10. Known limitations

Be honest if asked; none affect the graded tests.

- `valid_range` checks the page table once **per byte**; once per page would
  be ~4000× fewer lookups for a full-page buffer.
- fd numbers are never reused (`next_fd` only grows); lookup is O(n).
- In `create`/`remove`/`open`, if the user stack is invalid at the *second*
  argument, the name page allocated for the first argument leaks (the process
  is killed, but the page isn't returned).
- `read`/`write` with `size == 0` and `buffer == NULL` kills the process
  rather than returning 0.
- Donation chain depth is capped at 8.
- One global `filesys_lock` serialises all file I/O across all processes.
- No virtual memory: a stack access below the one mapped page is a fatal
  page fault.

---

## 11. TA question bank

**Q: Why copy the command line in `process_execute`?**
The parent's buffer could be freed or modified before the child reads it,
and the child runs on a different thread with no synchronisation.

**Q: Why does `exec` need `load_sema`?**
`exec` must return -1 if loading fails. The parent can't know the result
until the child has tried; the semaphore makes the parent wait for exactly
that moment and nothing more.

**Q: Why not just use the child's `struct thread` to store the exit status?**
The `struct thread` page is freed by the scheduler as soon as the child
dies. If the parent calls `wait` afterwards, it would read freed memory.
The heap `child_info` outlives both.

**Q: How do you prevent the record being freed twice or never?**
Two boolean owner flags; `release_child` flips one and frees only when both
are false, with interrupts disabled so the check-and-free is atomic.

**Q: What stops a user program from passing a kernel address?**
`is_user_vaddr()` rejects anything ≥ `PHYS_BASE`, and `pagedir_get_page()`
rejects unmapped user addresses. Every byte of every buffer is checked
before use.

**Q: Why `strtok_r` and not `strtok`?**
`strtok` keeps state in a static variable shared by all threads; preemption
mid-parse corrupts it. `strtok_r` keeps state in a caller-owned `save_ptr`.

**Q: Why is argv pushed right-to-left?**
So `argv[0]` ends up at the lowest address, matching how C indexes the array
upward. It is the standard 80x86 calling convention.

**Q: What is `f->eax` for?**
The syscall return value. `intr_exit` restores all registers from the
`intr_frame`, so whatever we store in `f->eax` is what the user's `%eax`
contains after `int $0x30`.

**Q: What happens on a page fault in user code?**
`page_fault()` in `exception.c` prints the fault, calls `kill()`, which
calls `thread_exit()`. `exit_status` is still -1, so the parent's `wait`
returns -1. The kernel is not affected.

**Q: Are file descriptors global?**
No. Per process, starting at 2, never inherited. fd 0/1 are reserved for
keyboard/console.

**Q: Why one big `filesys_lock`?**
The given file system (`filesys/`) has no internal synchronisation and we
were told not to modify it. A single lock is the simplest correct choice.
Lab 4 replaces it with finer-grained locking.

**Q: Why disable interrupts instead of using a lock in `release_child`?**
The critical section is three instructions and must also be safe from
`process_exit`, which can run late in a thread's life. Interrupt disabling
is the cheapest, cannot block, and cannot deadlock.

**Q: What does `thread_check_preempt` do and why is it everywhere?**
It yields if the front of the ready list has higher priority than the
current thread. Priority scheduling requires that the highest-priority
ready thread is *always* running, so every event that can make a
higher-priority thread ready must trigger it.

**Q: Explain priority donation with a nested chain.**
H waits on lock A held by M; M waits on lock B held by L. `thread_donate`
walks A→M (raise M to H), then M's `waiting_lock` B→L (raise L to H). L runs,
releases B, its priority is recomputed; M runs, releases A; H runs.

**Q: How is MLFQS priority computed without floating point?**
17.14 fixed point: integers scaled by 2^14. `FP_MUL` uses a 64-bit
intermediate to avoid overflow. See `fixed-point.h`.

---

## 12. Where to look for each file

| I want to see... | File:line |
|---|---|
| Sleep list and wake-up | [timer.c:timer_sleep / timer_interrupt](src/devices/timer.c) |
| Sorted ready list | [thread.c:261](src/threads/thread.c#L261), [thread.c:331](src/threads/thread.c#L331) |
| Donation | [thread.c:568](src/threads/thread.c#L568), [synch.c:202](src/threads/synch.c#L202) |
| MLFQS formulas | [thread.c:131](src/threads/thread.c#L131), [thread.c:595](src/threads/thread.c#L595) |
| Process creation | [process.c:38](src/userprog/process.c#L38) |
| Two-owner record | [process.c:115](src/userprog/process.c#L115) |
| ELF loader | [process.c:354](src/userprog/process.c#L354) |
| Stack layout | [process.c:601](src/userprog/process.c#L601) |
| `wait` | [process.c:177](src/userprog/process.c#L177) |
| Exit cleanup | [process.c:206](src/userprog/process.c#L206) |
| Pointer validation | [syscall.c:39](src/userprog/syscall.c#L39) |
| Syscall switch | [syscall.c:96](src/userprog/syscall.c#L96) |
| Exceptions / page fault | [exception.c:72](src/userprog/exception.c#L72) |
| Design document | [DESIGNDOC](src/userprog/DESIGNDOC) |
| Lab 2 narrative guide | [LAB2_GUIDE.md](LAB2_GUIDE.md) |
