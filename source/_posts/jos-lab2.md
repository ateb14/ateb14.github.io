---
title: jos-lab2
date: 2022-10-08 23:00:41
tags:
- operating system
- jos
categories:
- operating system
---
# LAB2, MEMORY MANAGEMENT

I finish these challenges:

- Use huge pages(4MB) to map the KERNBASE area to physical memories.
- Extend the JOS commands to display the mappings, edit the permissions of any mapping and dump the given virtual or  physical address.

## Physical Page Management

The boot_alloc() allocates memories above KERNBASE in the virtual memory space before the paging system being set up. In the memory initialization function mem_init(), after the size of the physical memory is detected, boot_alloc() is used to allocate two pieces of space where a page for the kernel page directory and some contiguous pages for a bijection to the physical pages are tagged. The bijection is represented as an array of the instances of `struct PageInfo` with the length of the page number of the physical memory, where the $i^{th}$ of it maps to the $i^{th}$ physical page.

After that, page_init() is called to initialized the bijection array. An `struct PageInfo` instance has a pointer field which points to another instance representing a free physical page if the instance is also represeting a free page, or it's null whether it represented an occupied page or it is just the end of the free list, so the free pages can be represented in a linked list(`struct PageInfo *page_free_list`). At the time when the page_init() is called, the physical memory usage status is as follows.

```
+------------------+  <- 0x08000000 (128MB)    <-|-----------------       
|                  |                             |
/\/\/\/\/\/\/\/\/\/\                             |
                                                 | free
/\/\/\/\/\/\/\/\/\/\                             |
|      Unused      |                             | 
|                  |                             |
+------------------+  <- 0x0015e000 (1MB+376KB)<-|----------------
|                  |                             | (in use)
|                  |                             | PageInfo array
| Extended Memory  |                             | One page for the page directory
|                  |                             | Kernel file
+------------------+  <- 0x00100000 (1MB)      <-|----------------
|     BIOS ROM     |                             |
+------------------+  <- 0x000F0000 (960KB)      |
|  16-bit devices, |                             | IO hole, in use
|  expansion ROMs  |														 |
+------------------+  <- 0x000C0000 (768KB)      |   
|   VGA Display    |                             |
+------------------+  <- 0x000A0000 (640KB)    <-|-----------------   
|                  |														 |
|    Base Memory   |                             | free
|                  |  <- 0x00001000            <-|-----------------   
+------------------+  <- 0x00000000            <- Page 0, in use
```

Then it's easy to set up the array and the free list. The operation logic of page_alloc and page_free is quite simple, just pick the head node of the free list when allocating and add the node to be free to the head of the free list when freeing.

## Virtual Memory

### Question 1.

*Assuming that the following JOS kernel code is correct, what type should variable `x` have, `uintptr_t` or `physaddr_t`?*

```c
mystery_t x;
char* value = return_a_pointer();
*value = 10;
x = (mystery_t) value;
```

`uintptr_t`,  for the jos kernel has to access the memory through virtual address.

### Implement the Paging Mechanism

The pgdir_walk() looks for the page table entry given the virtual address and the page directory. If the page table hasn't been created, it will allocate a physical page for the page table if the parameter `create` is set to true or returns null otherwise.

The boot_map_region() simply sets up the mapping between the given virtual address range and the physical address range by modifying the PTEs using pgdir_walk(). There is one thing that needs to pay attention to is that this function is only called to set up the **static** mappings at the boot time, so it won't increase the reference counter in the bijection structure.

The page_lookup(), page_insert() and page_remove() are functions that will be called after the boot time, so the last two of them should modify the reference counter and invalidate the TLB if an entry(mapping) is removed or modified. One thing needed to concern about in the page_insert() is that, before checking whether the mapping has been existing or not, the reference counter of the given physical page should be increased, or it will lead to some troublesome problems if the page_remove() decreases the counter to 0!

## Kernel Address Space

After the paging mechanism is implements, the mapping between the virtual memory space and the physical memory space should be set.

### Question 2.

page directory usage status

| Entry    | Base Virtual Address  | Points to                                                    |
| -------- | --------------------- | ------------------------------------------------------------ |
| 960~1023 | 0xf0000000~0xfffff000 | 256 MB of physical address(though only 128 MB of it is valid) |
| 959      | 0xefff8000(ULIM)      | The 32 KB kernel stack                                       |
| 957      | 0xef400000(UVPT)      | ?? I don't know                                              |
| 956      | 0xef000000(UPAGES)    | The `struct PageInfo` array                                  |

### Question 3.

*We have placed the kernel and user environment in the same address space. Why will user programs not be able to read or write the kernel's memory? What specific mechanisms protect the kernel memory?*

The protection bits in the PDEs and the PTEs specify the protection levels of the mapping, which the MMU will check when it receives a memory accessing request. Thus a request from the user mode to access the kernel's memory which is set to supervisor protection level will fail the protection check by the MMU and thus raises an exception.

### Question 4.

*What is the maximum amount of physical memory that this operating system can support? Why?*

128 MB, for the i386_detect_memory() has detected 0x8000 physical pages.

### Question 5.

*How much space overhead is there for managing memory, if we actually had the maximum amount of physical memory? How is this overhead broken down?*

- The `struct PageInfo` array costs 64 pages.
- The page directory costs 1 page.
- If all the page tables are in use, they will cost 1024 pages.

So the maximum amount of physical memories used to manage memory is $64+1+1024=1089$ pages, that is 4356 KB or 4.25 MB. However, as the two-level paging system is implemented, only the page table is needed to record at least one mapping will it be allocated in the physical memory. For now, there are only 67 page tables in use as the `info pg` shows, that means there are only 528 KB memory used to memory management.

### Question 6.

*Revisit the page table setup in `kern/entry.S` and `kern/entrypgdir.c`. Immediately after we turn on paging, EIP is still a low number (a little over 1MB). At what point do we transition to running at an EIP above KERNBASE? What makes it possible for us to continue executing at a low EIP between when we enable paging and when we begin running at an EIP above KERNBASE? Why is this transition necessary?*

After the instructions:

```assembly
mov	$relocated, %eax
jmp	*%eax
```

The entry_pgdir points to a page directory which is manually set in the `kern/entrypgdir.c` maps virtual addresses in [0, 4MB) and [KERNBASE, KERNBASE + 4MB) to physical addresses [0, 4MB) and is loaded to cr3 register before the paging mode is on, so after the paging is enabled the program will execute at a low EIP that is interpreted as a virtual memory. Since the mapping between [0, 4MB) and [0, 4MB) is set, the program is still able to excecute at a low EIP before the jmp instruction change the EIP above KERNBASE, otherwise the MMU can't interpret the request below 4MB. This is why this transition is necessary.

## Challenge: Huge Pages

According to the intel manuals, after the PTE_PS bit in the PDE is enabled, the MMU will interpret this PDE as an direct index towards a 4MB contiguous area in the physical memory instead of indexing the page table if the PSE bit in cr4 register is enabled.

So I implement function boot_map_region_big_page() in `kern/pmap.c` for the KERNBASE mapping, which will set the mappings in the page directory and enable the PTE_PS bit in the PDEs. After the huge pages mappings are set up and before the address of the page directory being loaded into the cr3 register, the PSE bit in cr4 is enabled. These enable the huge page mode and set up the mappings.

The setting up of the huge page mode will fail the check about the KERNBASE area, for the check_va2pa() function can't interpret the huge pages correctly, so it is also modified to translate the huge pages.

## Challenge: Debug Commands

- Command that shows mappings: `sm <vm_left> <optional: vm_right>/<optional: +page_offset> <optional: r>(round down to page size)`

This command will display the mappings of the given virtual address, it can interpret the huge pages correctly.

```
K> sm 0xffffa000 +3 # 3 more mappings
VA:0xffffa000        PA:0x0fffa000           W:1 U:0
VA:0xffffb000        PA:0x0fffb000           W:1 U:0
VA:0xffffc000        PA:0x0fffc000           W:1 U:0
VA:0xffffd000        PA:0x0fffd000           W:1 U:0
K> sm 0xef3fffff 0xef400a00 r # specify the right boundary and round down to PGSIZE
VA:0xef3ff000        PA:0x0051d000           W:0 U:1
VA:0xef400000        PA:----------           W:- U:-
```

Just look into the PD to find the entry and display it.

- Command that modify the permissions of any given mapping: 

```
K> spp
usage: spp <vm_addr> <mode> <permission1> <permission2>...
modes: clear, cover, add, delete
K> sm 0xf0000000
VA:0xf0000000        PA:0x00000000           W:1 U:0
K> spp 0xf0000000 add U
K> sm 0xf0000000
VA:0xf0000000        PA:0x00000000           W:1 U:1
K> spp 0xf0000000 clear
K> sm 0xf0000000
VA:0xf0000000        PA:0x00000000           W:0 U:0
K> spp 0xf0000000 cover W
K> sm 0xf0000000
VA:0xf0000000        PA:0x00000000           W:1 U:0
K> spp 0xf0000000 delete W
K> sm 0xf0000000
VA:0xf0000000        PA:0x00000000           W:0 U:0
```

Just look into the PD to find the entry and modify it.

- Command that dumps a given virtual address or a physical address

```
K> dump
usage: dump <-p/-v> <addr> <optional: +<$byte_offset>> <optinal: d<$decode_range(1,2,4 or 8)>>
K> dump -v 0xf0000000 +3
VA:0xf0000000        PA:0x00000000           Val:0x53
VA:0xf0000001        PA:0x00000001           Val:0xff
VA:0xf0000002        PA:0x00000002           Val:0x0
VA:0xf0000003        PA:0x00000003           Val:0xf0
K> dump -p 0x0 +3
PA:0x00000000        Val:0x53
PA:0x00000001        Val:0xff
PA:0x00000002        Val:0x0
PA:0x00000003        Val:0xf0
K> dump -v 0xf7ffffff +1
VA:0xf7ffffff        PA:0x07ffffff           Val:0x0
VA:0xf8000000        PA:0x08000000           Val:PA out of range
```

For the physical address, just dereference it with PADDR, while for the virtual address, it is needed to find the associated PDE and PTE.
