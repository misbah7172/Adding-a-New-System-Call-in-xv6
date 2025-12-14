# Adding a New System Call in xv6

This document describes the **standard and correct procedure** to add a new system call in the xv6 operating system. The same workflow can be followed for **any system call**, including the `trace` system call used in Task-1.

---

## Overview

Adding a system call in xv6 requires coordinated changes in **user space** and **kernel space**. The goal is to allow a user program to invoke a function that safely executes inside the kernel.

---

## Step-by-Step Procedure

### 1. Declare the System Call in User Space

Add the function prototype in `user/user.h` so that user programs can access the system call.

```c
int trace(int);
```

> Replace the function name and parameters as required for other system calls.

---

### 2. Add the User-Space Stub Entry

In `user/usys.pl`, add an entry for the new system call.

```perl
entry("trace");
```

This automatically generates the required assembly stub in `usys.S` when `make` is executed.

---

### 3. Register the User Program in the Makefile

If the system call has an associated user program (e.g., `trace.c`), add it to `UPROGS` in the `Makefile`.

```make
$U/_trace \
```

This ensures the program is compiled and available in the xv6 shell.

---

### 4. Define a System Call Number

In `kernel/syscall.h`, assign a unique system call number.

```c
#define SYS_trace 22
```

Ensure the number does not conflict with existing system calls.

---

### 5. Declare the Kernel Handler Function

In `kernel/syscall.c`, declare the kernel-side handler.

```c
extern uint64 sys_trace(void);
```

---

### 6. Register the System Call in the Syscall Table

In `kernel/syscall.c`, add the system call to the `syscalls` array.

```c
[SYS_trace] sys_trace,
```

This maps the syscall number to its implementation.

---

### 7. Implement the System Call Logic

Define the system call function, typically in `kernel/sysproc.c`.

```c
uint64
sys_trace(void)
{
  int sys_num;
  argint(0, &sys_num);          // Fetch argument from user space
  myproc()->trace_num = sys_num;
  return 0;
}
```

This function performs the core kernel-side operation.

---

### 8. Extend the Process Structure (If Required)

If the system call requires storing process-specific data, add a new field to `struct proc` in `kernel/proc.h`.

```c
int trace_num;
```

---

### 9. Initialize the New Process Field

Initialize the newly added field when a process is created (in `allocproc()` in `kernel/proc.c`).

```c
p->trace_num = 0;
```

This prevents unintended behavior across processes.

---

### 10. Modify the Syscall Dispatcher (Task-Specific)

For tasks like `trace`, modify the `syscall()` function in `kernel/syscall.c` to intercept and log system calls.

```c
if (p->trace_num == num) {
  printf("pid: %d, syscall: %s, return value: %d\n",
         p->pid, syscall_names[num], ret);
}
```

This ensures tracing occurs only for the calling process.

---

### 11. Add Syscall Name Mapping (Optional but Recommended)

To print readable syscall names, define a syscall name array in `kernel/syscall.c`.

```c
static char *syscall_names[] = {
  [SYS_fork]  "fork",
  [SYS_read]  "read",
  [SYS_trace] "trace",
};
```

---

## Summary

The general workflow for adding a system call in xv6 is:

1. Declare the system call in `user.h`
2. Add an entry in `usys.pl`
3. Register the user program in `Makefile`
4. Assign a syscall number
5. Declare the kernel handler
6. Register the handler in the syscall table
7. Implement the system call logic
8. Extend `struct proc` if needed
9. Initialize new fields
10. Modify `syscall()` for task-specific behavior
11. Add syscall name mapping if output formatting is required

Following this structured approach ensures correct integration of any new system call from **user space to kernel space** in xv6.

