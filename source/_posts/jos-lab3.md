---
title: jos-lab3
date: 2022-10-25 19:53:28
tags:
- operating system
- jos
categories:
- operating system
---

# LAB3, USER ENVIRONMENTS

I finished these challenges:

- Generate the IDT and the handlers gracefully with macros.
- Implement the step-in and continue commands in debug mode.

## Part A: User Environments and Exception Handling

### Exercise 2. Creating and Running Environments

- In `env_setup_vm`, we can simply copy kernel's page directory to that of the new environmen first,  for the virtual spaces above UTOP for all the environments and the kernel are the same except at UVPT.
- Function `load_icode` loads the ELF file of the environment embedded in kernel's ELF file, which is loaded in the memory at boot time. By reading the program headers of the ELF file, the loader will load the segments of the environment into physical memory and map them to proper places  in VA space assigned by the linker. It's noteworthy that the size of a segment in the ELF file may be less than the space it takes up when loaded into environment's VA space due to the existence of bss segment, so the loader needs to fill this extra spaces with zero.
- Function `env_run` makes a context switch to the user environment by loading the page directory into CR3 register and popping the trap frame to proper registers. Here is what will instruction iret do that I found in the X86 documents.

*"The IRET instruction pops the return instruction pointer, return code segment selector, and EFLAGS image from the stack to the EIP, CS, and EFLAGS registers, respectively, and then resumes execution of the interrupted program or procedure. If the return is to another privilege level, the IRET instruction also pops the stack pointer and SS from the stack, before resuming program execution."																																					--x86doc*[1] 

### Exercise 4. Setting Up the IDT && Challenge 1

To set up the IDT, entries of the handlers should be set in `kern/trapentry.S`, which will be stored in the IDT later in `trap.init`. There are two kinds of trap handlers, the first of which handles interrupts with error code and the second one handles interrupts without error code by pushing a zero to the exception stack in place of the error code. So the purpose of having an individual handler for each interrupt is to distinguish between these two types of interrupts(Answer to **question 1**). All the handlers need to execute certain instrcutions located in `_alltrap` that push proper registers to the stack in the form of the trap frame, push the address of this trap frame as a parameter and call function `trap` to handle the interrupts or exceptions.

To generate the handlers and the IDT gracefully(**Challenge 1**),  I define macros that will generate the entries of the handlers or generate a table of the address of the entries given two different parameters. Here is the script .

```assembly
#define GEN_TRAPHANDLER_TEXT(num) \
	TRAPHANDLER(handler##num, num);

#define GEN_TRAPHANDLER_NOEC_TEXT(num) \
	TRAPHANDLER_NOEC(handler##num, num);

#define GEN_TRAPHANDLER_DATA(num) \
	.long handler##num;

#define GEN_TRAPHANDLER_NOEC_DATA(num) \
	.long handler##num;

#define TRAP_GENERATOR(type) \
	GEN_TRAPHANDLER_NOEC_##type(0); \
	GEN_TRAPHANDLER_NOEC_##type(1); \
	...
	GEN_TRAPHANDLER_NOEC_##type(19); \ // totally 20 handlers
	
.text
TRAP_GENERATOR(TEXT)
GEN_TRAPHANDLER_NOEC_TEXT(48) # T_SYSCALL

.data
	.p2align	2	# force 4 byte alignment
	.globl		handlers	# make it a global variable				
handlers:
	TRAP_GENERATOR(DATA)
	GEN_TRAPHANDLER_NOEC_DATA(48) # T_SYSCALL // handlers[20], for convenience
```

By specifying the directives .text and .data and using a few macros, the entries and a table of their addresses are generated neatly! After that we just need to assign the IDT in `trap_init` with macro SETGATE, the addresses stored in the generated table, a selector of kernel's text(GD_KT) and a privilege level 0(for now).

#### Question 2

*The environment `softint` raises an interrupt with `int $14`, but the processor produces a general proctection fault(trap 13). Why?*

Trap 14 is page fault which has a privilege level 0, so the request to invoke it from the user environment with a RPL of 3 violates the permission check, which leads to a general protection fault. If the kernel allows the user environment to invoke the page fault handler, the kernel will probably mess up everything for the behavior of the user environment is unpredictable, for it can trap into the kernel with a fake page fault whenever it likes!

## Part B: Page Faults, Breakpoints Exceptions, and System Calls

To handle different exceptions, I simply added a switch statement in function trap_dispatch.

### Challenge 2

To single-step one instruction at a time, the TF bit in EFLAGS register should be set, which will make the processor raise a debug interrupt right after an instruction is excecuted every time. Then the kernel will handle the debug interrupt the same as the breakpoint interrupt by calling function monitor. Symmetrically, the continue function can be implemented by clearing the TF bit of EFLAGS. The way to set the EFLAGS register is to modify the EFLAGS field of the trap frame in the memory, and the iret instruction will restore it back to the EFLAGS register when switching back to the environment. Also I check the privilege level of the trap frame to ensure only user environments can modify the TF bit, or the kernel may recursively receive debug interrupts and thus crashes.

### Question 3

*The break point test case will either generate a break point exception or a general protection fault depending on how you initialized the break point entry in the IDT (i.e., your call to `SETGATE` from `trap_init`). Why? How do you need to set it up in order to get the breakpoint exception to work as specified above and what incorrect setup would cause it to trigger a general protection fault?*

If the IDT entry of the breakpoint interrupt is set with a privilege level 0, then the request to invoke a breakpoint interrupt from the user environment will generate a general protection fault. So the privilege level of the breakpoint entry should be set to 3.

### Qustion 4

*What do you think is the point of these mechanisms, particularly in light of what the `user/softint` test program does?*

To prevent the user environments to invoke the interrupts or the exceptions arbitrarily 

### Exercise 7. System calls

To allow the environment to use system calls, the entry of the system call trap in the IDT should be set similar to before, and the dispatcher should call function `syscall` dedicated to handle system calls, with the arguments from the trap frame of the current environment. It's also the dispatcher's duty to store the return value of the system call, if there is any, to the trap frame of the environment. Function `syscall` handles the system call given the system call number respectively.

### Exercise 8. User-mode startup

As we know that there is en entry function that calls the main function of the user program, that is `libmain` in JOS. Function libmain makes the user environment know its environment information by invoking a system call that returns the pointer to its Env structure, for the environment information array is read-only to the users.

### Exercise 9. Page faults and memory protection

When the user environment is still active but it traps into the kernel with a breakpoint interrupt or something else, the kenel monitor can display the backtrace normally. However, a page fault will happen when we use backtrace command after the destroying of the user environment.The reason why it happens is due to the fact that, memory mappings of the user environment defined in the page tables of the user environment don't reside in kernel's page tables, and the page directory loaded in register CR3 is no longer that of the user envrionment's, so the kernel will encounter a page fault when it traces into the address space part that is owned by the environment individually.

# References

[1] <https://asmdude.github.io/x86doc/html/IRET_IRETD.html>
