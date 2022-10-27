---
title: 'Writeup for The Multics Virtual Memory: Concepts and Design'
date: 2022-10-18 09:46:28
tags:
- operating system
- paper reading
categories:
- operating system
---

# Writeup for *The Multics Virtual Memory: Concepts and Design(1970)*

## What is this paper talking about?

This is a definition paper which talks about the design and implementation of the virtual memory mechanism in Multics(Multiplexed Information and Computing Service). The Multics system is a segmented system where equal-size pages compose of every segment and segments are packages of information that are directly addressed and have a set of  access attributes that are checked by hardware whenever they are referenced by any user. Each segment owns **at most** a page table and each process is able to know about and connect to several segments

## What is the motivation?

Previous systems that used only paging, only segementaion or the combination of both wern't able to meet the requirements to support the **rapid** but **controlled** sharing of information stored on-line, which requires not only the software design but also the hardware support. Aiming at solving this problem, the Multics system provides a segmentation and paging mechanism that satisfies two design goals:

- All on-line information, i.e. information stored in the core memory, is addressed directly by a processor and hence referenced directly by any  user. (rapid, shared)
- All the references to all on-line information are under control according to their and the information's privilege levels. (controlled)

## Benefits over the "state-of-the-art"

- All on-line information can be referenced directly by any user as long as it passes the privilege check. Information is both sharable and controllable in the Multics system compared with the previous systems.
- The segmentation abstract is fully managed by the supervisor and the number of the segment descriptors is sufficiently large enough, so the Multics user doesn't have to manage the segment descriptors and it can just directly operate on the original information that retains its identity and attributes instead of the copies of the information.
- The combination of the paging and segmentation mechanism makes it needless to store all the information of a segment on the core memory for only the needed pages will reside in the memory, which allows a segment to be large enough compared to the size of a page.
- The policy applied in the Multics system that tries to only keep the segments in use be active, only keep the pages in use in the core memory and make a process know about the segments only when they are needed by this process fully exploits the physical memory.

## Drawbacks 

- The privilege checks are only performed on the segmentation level, i.e. the page table entries don't specify the access levels, which makes it not flexible enough to manage the access levels on the address space.

## Conclusions

The segmentation and paging mechanism described in this paper is actually really close to the memory manage systems nowadays, for the idea that a system should allow sharable but controllable information referencing is still valuable in today's opinion and is widely applied in modern systems. 

Although the paging mechanism is quite naive, where only the virtual addresses, physical addresses and the valid tags are stored in an entry and no privilege checks will be performed in page level, the idea that the access levels should be specified by the supervisor and checked by the hardware is already used in the segmenation mechanism, which definitely inspired the design of the following memory management systems that are more complete in both software and hareware levels.

Actually, the general ideas of the memory management system in this paper are quite complete even compared with the modern systems.

