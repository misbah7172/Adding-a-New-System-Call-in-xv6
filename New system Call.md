# Guide to Adding a New System Call in xv6-riscv

This document provides a step-by-step guide to implementing a new system call in the xv6 operating system.

## Overview

A system call is a mechanism that allows user programs to request services from the kernel. Adding a new system call involves modifications to both kernel-space and user-space code.

## Architecture

```
User Program (user/myprogram.c)
       ↓
User Library (user/user.h, user/usys.S)
       ↓
System Call Interface (kernel/syscall.c)
       ↓
System Call Implementation (kernel/sysproc.c or kernel/sysfile.c)
```

---

## Step-by-Step Implementation

### Step 1: Define System Call Number

**File:** `kernel/syscall.h`

Add a unique number for your system call at the end of the list.

```c
#define SYS_mysyscall  24  // Replace 24 with next available number
```

**Rules:**
- Each syscall must have a unique number
- Use the next sequential number after the last defined syscall
- Current xv6 has syscalls numbered 1-23

---

### Step 2: Declare Function Prototype

**File:** `kernel/syscall.c`

Add the external function declaration in the list of prototypes (around line 105).

```c
extern uint64 sys_mysyscall(void);
System call handlers in xv6 (functions starting with sys_) don't receive arguments the normal way.
When a user program calls a syscall, arguments are placed in CPU registers (RISC-V registers a0, a1, a2, etc.) 
```

---

### Step 3: Register System Call in Dispatch Table

**File:** `kernel/syscall.c`

Add your syscall to the `syscalls[]` array (around line 130).

```c
static uint64 (*syscalls[])(void) = {
  // ... existing syscalls ...
  [SYS_mysyscall]   sys_mysyscall,
};
```

**Note:** The array index must match the syscall number defined in Step 1.

---

### Step 4: Add System Call Name (For Debugging)

**File:** `kernel/syscall.c`

Add the syscall name to the `syscallnames[]` array (around line 155).

```c
static char *syscallnames[] = {
  // ... existing names ...
  [SYS_mysyscall]   "mysyscall",
};
```

This enables debug output showing syscall names in traces.

---

### Step 5: Implement System Call Handler

**File:** `kernel/sysproc.c` (for process-related) or `kernel/sysfile.c` (for file-related)

Implement the actual system call logic:

```c
uint64
sys_mysyscall(void)
{
  // 1. Declare variables
  int arg1;
  uint64 arg2;
  
  // 2. Extract arguments from user space
  argint(0, &arg1);      // First integer argument
  argaddr(1, &arg2);     // Second address argument
  // argstr() for string arguments
  
  // 3. Get current process (if needed)
  struct proc *p = myproc();
  
  // 4. Implement your logic
  // ... your implementation ...
  
  // 5. Return result (0 for success, -1 for error)
  return 0;
}
```

**Argument Extraction Functions:**
- `argint(n, *ip)` - Extract integer argument n
- `argaddr(n, *addr)` - Extract pointer/address argument n
- `argstr(n, buf, max)` - Extract string argument n
- `argfd(n, *pfd, **pf)` - Extract file descriptor argument n

---

### Step 6: Declare User-Space Function

**File:** `user/user.h`

Add the function declaration to make it available to user programs.

```c
int mysyscall(int arg1, void* arg2);  // Match your implementation's signature
```

**Note:** The return type and parameters should match your implementation.

---

### Step 7: Generate System Call Stub

**File:** `user/usys.pl`

Add an entry to generate the assembly stub that transitions from user to kernel mode.

```perl
entry("mysyscall");
```

This Perl script automatically generates the assembly code in `user/usys.S`.

---

### Step 8: (Optional) Create Test User Program

**File:** `user/mysyscall_test.c`

Create a user program to test your syscall:

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  int result;
  
  // Call your syscall
  result = mysyscall(42, 0);
  
  if(result < 0) {
    printf("mysyscall failed\n");
    exit(1);
  }
  
  printf("mysyscall succeeded: %d\n", result);
  exit(0);
}
```

---

### Step 9: (Optional) Add Test Program to Makefile

**File:** `Makefile`

Add your test program to the `UPROGS` list (around line 130):

```makefile
UPROGS=\
	$U/_cat\
	# ... other programs ...
	$U/_mysyscall_test\
```

---

## Complete Example: Simple `getcount` System Call

This example implements a syscall that returns how many times it has been called.

### Kernel Changes

**`kernel/syscall.h`:**
```c
#define SYS_getcount  24
```

**`kernel/proc.h`** (add to `struct proc`):
```c
int syscall_count;  // Counter for getcount syscall
```

**`kernel/proc.c`** (initialize in `allocproc()`):
```c
p->syscall_count = 0;
```

**`kernel/sysproc.c`:**
```c
uint64
sys_getcount(void)
{
  struct proc *p = myproc();
  p->syscall_count++;
  return p->syscall_count;
}
```

**`kernel/syscall.c`:**
```c
// Add prototype
extern uint64 sys_getcount(void);

// Add to syscalls array
[SYS_getcount]   sys_getcount,

// Add to syscallnames array
[SYS_getcount]   "getcount",
```

### User Space Changes

**`user/user.h`:**
```c
int getcount(void);
```

**`user/usys.pl`:**
```perl
entry("getcount");
```

**`user/getcount_test.c`:**
```c
#include "kernel/types.h"
#include "user/user.h"

int main(void) {
  for(int i = 0; i < 5; i++) {
    printf("Call count: %d\n", getcount());
  }
  exit(0);
}
```

---

## Building and Testing

### 1. Build the Kernel
```bash
make clean
make qemu
```

### 2. Test Your System Call
```bash
$ mysyscall_test
```

### 3. Debug Output
If you encounter errors, check:
- Compiler errors: Review syntax in kernel files
- Link errors: Ensure all functions are properly declared
- Runtime errors: Add `printf()` statements in kernel code

---

## Common Patterns

### Pattern 1: Process Information Syscall
Returns information about the current process.

```c
uint64
sys_getpid(void)
{
  return myproc()->pid;
}
```

### Pattern 2: Syscall with Arguments
Takes parameters from user space.

```c
uint64
sys_kill(void)
{
  int pid;
  argint(0, &pid);
  return kill(pid);
}
```

### Pattern 3: Syscall Modifying Process State
Changes process attributes.

```c
uint64
sys_setpriority(void)
{
  int priority;
  argint(0, &priority);
  
  struct proc *p = myproc();
  p->priority = priority;
  
  return 0;
}
```

### Pattern 4: Syscall with Pointer Arguments
Safely copies data between user and kernel space.

```c
uint64
sys_read(void)
{
  int fd;
  uint64 addr;
  int n;
  
  argint(0, &fd);
  argaddr(1, &addr);
  argint(2, &n);
  
  // Use copyin() or copyout() to safely transfer data
  return fileread(fd, addr, n);
}
```

---

## Best Practices

### 1. **Validate Arguments**
Always validate user-provided arguments to prevent security issues.

```c
if(priority < 0 || priority > MAX_PRIORITY) {
  return -1;  // Invalid argument
}
```

### 2. **Use Proper Error Codes**
Return `-1` on error, non-negative values on success (following Unix convention).

### 3. **Add Locking When Needed**
If your syscall accesses shared data structures, use appropriate locks.

```c
acquire(&ptable.lock);
// ... critical section ...
release(&ptable.lock);
```

### 4. **Memory Safety**
When dealing with user pointers:
- Use `copyin()` to read from user space
- Use `copyout()` to write to user space
- Never directly dereference user pointers in kernel mode

```c
// Bad - unsafe!
char *userstr = (char*)addr;

// Good - safe!
char kbuf[128];
if(copyin(p->pagetable, kbuf, addr, sizeof(kbuf)) < 0)
  return -1;
```

### 5. **Keep It Simple**
System calls should be minimal and focused. Complex logic should be in separate kernel functions.

---

## Debugging Tips

### 1. **Add Debug Prints**
```c
printf("sys_mysyscall: arg1=%d, arg2=%p\n", arg1, arg2);
```

### 2. **Check Return Values**
```c
if(argint(0, &arg1) < 0)
  return -1;  // Argument extraction failed
```

### 3. **Use GDB**
Start xv6 with debugging:
```bash
make qemu-gdb
```
In another terminal:
```bash
gdb-multiarch kernel/kernel
(gdb) target remote localhost:26000
(gdb) b sys_mysyscall
(gdb) c
```

### 4. **Check System Call Tracing**
Use the `trace` syscall to monitor your new syscall:
```bash
$ trace 32 mysyscall_test  # 32 = 2^5 if SYS_mysyscall is 5
```

---

## Checklist

Before considering your syscall complete, verify:

- [ ] Syscall number defined in `kernel/syscall.h`
- [ ] Function prototype added in `kernel/syscall.c`
- [ ] Function registered in `syscalls[]` array
- [ ] Function name added to `syscallnames[]` array
- [ ] Implementation added in `kernel/sysproc.c` or `kernel/sysfile.c`
- [ ] User-space declaration in `user/user.h`
- [ ] Entry added in `user/usys.pl`
- [ ] Test program created (optional but recommended)
- [ ] Test program added to `Makefile` (if created)
- [ ] Code compiles without errors
- [ ] System call works as expected
- [ ] Error cases handled properly
- [ ] Memory safety verified

---

## References

- [xv6 Book](https://pdos.csail.mit.edu/6.828/2023/xv6/book-riscv-rev3.pdf) - Chapter 2: Operating System Organization
- Original xv6 repository: https://github.com/mit-pdos/xv6-riscv
- RISC-V System Calls: Uses the `ecall` instruction to trap into kernel mode

---

## Summary

Adding a system call requires coordinated changes across multiple files:

1. **Define** the syscall number (`syscall.h`)
2. **Declare** the function (`syscall.c`)
3. **Register** it in dispatch tables (`syscall.c`)
4. **Implement** the handler (`sysproc.c` or `sysfile.c`)
5. **Export** to user space (`user.h`, `usys.pl`)
6. **Test** with a user program

Follow this guide systematically, and your system call will integrate seamlessly with xv6!
