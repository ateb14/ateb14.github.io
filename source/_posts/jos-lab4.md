---
title: jos-lab4
date: 2022-11-15 10:49:18
tags:
- operating system
- jos
categories:
- operating system

---

# LAB4, PREEMPTIVE MULTITASKING

I finished these challenges:

- Implement sfork().
- Implement the fixed-priority scheduler.

## Multiprocessor Support and Cooperative Multitasking

Jos will support "symmetric multiprocessing"(SMP) after this part.

### Multiprocessor Support & Application Processor Bootstrap & Question 1

*Compare `kern/mpentry.S` side by side with `boot/boot.S`. Bearing in mind that `kern/mpentry.S` is compiled and linked to run above `KERNBASE` just like everything else in the kernel, what is the purpose of macro `MPBOOTPHYS`? Why is it necessary in `kern/mpentry.S` but not in `boot/boot.S`? In other words, what could go wrong if it were omitted in `kern/mpentry.S`?*

The macro `MPBOOTPHYS` is used to calculate the physical address of the given symbols in `kern/mpentry.S`. The load address of the application processors'(AP) booting code is specified by the bootstrap processor(BSP), which is irrelevant with the load address set by the linker, thus it's necessary to calculate the physical addresses of all the symbols at runtime.

### Per-CPU State and Initialization

These are the per-cpu states needed to be initialized when booting the APs.

- Per-cpu kernel stack
- Per-cpu kernel TSS and TSS descriptor: The TSS descriptors are contained in the GDT at offset `GD_TSS0`, remember to set a busy flag on it using `ltr(GD_TSS0 + (i << 3))` after setting up the TSS for the current booting ith CPU!
- Per-CPU current environment pointer
- Per-CPU system registers: All registers are private to a CPU, so remember to initialize them for every AP! Recall that I've enabled the huge page flag of cr4 register in lab2 to support the huge page mapping above KERNBASE, so it's necessary to enable it again before we loading the page directory into cr3.

### Locking & Question 2

*It seems that using the big kernel lock guarantees that only one CPU can run the kernel code at a time. Why do we still need separate kernel stacks for each CPU? Describe a scenario in which using a shared kernel stack will go wrong, even with the protection of the big kernel lock.*

For example, when the kernel is trapped from the user mode, the CPU has already pushed some contents of the trap frame onto the kernel stack before the kernel acquires the big kernel lock, so if there are multiple traps from the user mode on different CPUs at the same time, the single kernel stack will be messed up by all these trap frames.

### Round-Robin Scheduling & Question 3,4

*In your implementation of `env_run()` you should have called `lcr3()`. Before and after the call to `lcr3()`, your code makes references (at least it should) to the variable `e`, the argument to `env_run`. Upon loading the `%cr3` register, the addressing context used by the MMU is instantly changed. But a virtual address (namely `e`) has meaning relative to a given address context--the address context specifies the physical address to which the virtual address maps. Why can the pointer `e` be dereferenced both before and after the addressing switch?*

For the mappings in the kernel space, i.e. the addresses above UTOP, are the same for every environment as well as the kernel.

*Whenever the kernel switches from one environment to another, it must ensure the old environment's registers are saved so they can be restored properly later. Why? Where does this happen?*

Because the environments don't have a private space to store their registers when a context switch happens, so the kernel should store them in the environment descriptors(PCB). It happens when a trap is occurred and is in the handler function "trap" in `kern/trap.c`.

```C
// Copy trap frame (which is currently on the stack)
// into 'curenv->env_tf', so that running the environment
// will restart at the trap point.
curenv->env_tf = *tf;
// The trapframe on the stack should be ignored from here on.
tf = &curenv->env_tf;
```

### Challenge: Fixed-priority Scheduler

I add an extra field called `env_priority` in the PCB of every environment, which stands for the priority of an environment and is set to 10000 by default. Then I implement a system call named `sys_set_priority`, through which an environment can change the priority of itself or its children, but the value of the target priority is forced to be equal or less than the current environment's priority. Also I modify the scheduler in `kern/sched.c` that higher-priority environments are always chosen in preference to the lower-priority ones, by simply scanning all the runnable environments.

Here is a simple test program to verify the correctness by assigning ascending priorities to different children, and the parent is always the highest.

```C
void
umain(int argc, char **argv)
{
	cprintf("hello, world\n");
	cprintf("i am environment %08x\n", thisenv->env_id);

	int i = 0;
	envid_t envid[10];
	for(i = 0; i < 10; ++i){
		envid[i] = fork();
		if(envid[i] == 0){
			cprintf("Hello I am child %d with priority %u\n", i, thisenv->env_priority);
			return;
		} 
		int r = sys_set_priority(envid[i], i);
		if(r < 0){
			panic("PRIOR: %e", r);
		}
	}
}
```

The output is as expected, as the children execute in order.

```
[00000000] new env 00001000
hello, world
i am environment 00001000
[00001000] new env 00001001
... ignoring many similar outputs...
[00001000] new env 00001009
[00001000] new env 0000100a
[00001000] exiting gracefully
[00001000] free env 00001000
Hello I am child 9 with priority 9
[0000100a] exiting gracefully
[0000100a] free env 0000100a
Hello I am child 8 with priority 8
[00001009] exiting gracefully
[00001009] free env 00001009
... ignoring many similar outputs...
[00001002] exiting gracefully
[00001002] free env 00001002
Hello I am child 0 with priority 0
[00001001] exiting gracefully
[00001001] free env 00001001
No runnable environments in the system!
```

### System Calls for Environment Creation

In this section we should implement several primitives as some system calls, remember to check the permission and the relationship when one user environment tries to modify the status of a destination environment .

- sys_exofork: Create another environment and block it. Remember to clear the eax register in the trap frame of the newly created environment.
- sys_env_set_status: Block or unblock a specified environment.
- sys_page_alloc, sys_page_map, sys_page_unmap: These are the primitives to manipulate the address space of a specified environment, so we should carefully handle all the possible errors!

## Copy-on-Write Fork

### Invoking the User Page Fault Handler & User-mode Page Fault Entrypoint

The page fault handler defined by the user(the page fault upcall) by which the user environment will handle the page fault on its exception stack, so the kernel only needs to make sure that the address of the upcall is valid and the user exception stack is allocated and is sufficient to push a user trap frame on. Depending on whether the incoming trap is from a previous upcall or not, the kernel will allocate a word on the exception stack in addition to the fixed space for a user trap frame if it's from a upcall, which is useful for the user handler to restore the context when a recursive upcall hanppens. After that, the kernel will set the trap frame(not the user trap frame!) of the current environment, where the eip is set to the entry point of the user page fault handler and the esp is set to the stack pointer of the user exception stack.

The entry point of the user page fault handler is defined in `lib/pfentry.S`, which will call the C-language page fault  handler and return to the instruction that triggers page fault afterwards. This is perhaps the most tricky part in this lab, for there might be a transition from the exception stack to the normal stack and we should treat it equally as the recursive upcall situation, as well that we should restore all the general register.

To solve this problem, we should push the trap-time eip to the **trap-time** stack, and use a `ret` instruction to restore executions after we have restored the general registers and the eflag properly. However, we can't use the `push` instruction directly for sure, instead, we should read the trap-time esp from the user trap frame and manually "push" the trap-time eip by subtracting a word from the trap-time esp and writing the eip to the space it points to. Recall that the kernel will leave an extra word on the exception stack so it won't violate the structure of the user trap frame when there is a recursive upcall. After that, we use `popal` and `popf` to restore the general registers and the eflag — we do it after the "push" operation because we have to use the general registers to do some calculations. Finally, we load the esp with the value of the trap-time esp and use a `ret` instruction to restore executions.

### Implementing Copy-on-Write Fork

In this section we should implement a **user-level** copy-on-write fork based on the system calls above, and here are the functions we should implement.

- duppage:  It duplicates a page table entry and moves it into the child's page tables, and if the page pointed by this entry is writable or copy-on-write, the page table entries of both the parent and the child should be updated to copy-on-write.
- pgfault: This is the alternative page fault handler specified by the user, which will allocate a new page and copy the old page to it when there is an access to a copy-on-write page. And then it will add a new mapping to the environment that triggers this page fault in place of the old mapping and mark it as writable.
- fork: It forks a child environment using `sys_exofork`. For the child, it simply resets the `thisenv` variable to the child. While for the parent, it uses duppage to construct the mappings of the child below the user exception stack and allocates a writable exception stack for the child — this is necessary for the page fault handler that deals with a copy-on-write page will run on a exception stack. Then the parent sets the page fault handler of the child and unblocks it.

### Challenge: sfork()

The way to implement the shared-memory sfork() is similar to that of fork(), I just simply copy the page table entries of the parent environment except for the exception stack and the normal stack. Then I allocate a writable page for the exception stack and mark the normal stack as copy-on-write for the child environment.

The previous `thisenv` pointer in the user library is no longer compatible with sfork(), for it is statically lay in the .text area of the program, which will be sharable and thus leads to conflict between different environments. Then I manage to solve this problem by making it dynamic with the help of a macro:

```c
#define thisenv (&((struct Env *)UENVS)[ENVX(sys_getenvid())])
```

So for now, every time an environment tries to get access to its PCB, it will make a system call to fetch it. I gain the correctness at the cost of some efficiency loss.

## Preemptive Multitasking and Inter-Process Communication (IPC)

### Clock Interrupts and Preemption

To receive and handle the hardware interrupts, the entries after IRQ_OFFSET in the IDT should be initialized just like what I've done in lab3, and the FL_IF flag should be enabled when a user environment is running. For simplicity, this flag is disabled when the kernel is executing. The kernel will handle the clock interrupts by simply calling `sched_yield` to transfer the control flow to another environment if there is any.

### Inter-Process Communication (IPC)

Two environments are able to communicate with each other by a sender function and a receiver function defined by the kernel. For the receiver, it marks itself as ready to receive a message and blocks itself waiting for a message. For the sender,  if the receiver is in the receivable state, it will store the message in the PCB of the receiver environment, mark the receiver as unable to receive a new message and awake it. Or if the receiver is unable to receive a message, the sender simply returns an error code. The message delivered from the sender discussed above contains a word, an address and a permission status. If the address is below UTOP, and the receiver also specifies an address below UTOP, the sender will also map the address of the receiver's to the physical page to which the address of the sender's points, with the permission status sent from the sender, as a result the receiver will share this physical page with the sender.

In the view of the user environments, the receiver simply make the receiver system call and waits for a sender, while the sender will repeatedly make the sender system call until the target environment receives the message or it encounters a unrecoverable failure.


