---
title: 'Writeup for The Structure of the "THE"-Multiprogramming System'
date: 2022-10-02 19:36:01
tags:
- operating system
- paper reading
categories:
- operating system

---
# Writeup for *The Structure of the "THE"-Multiprogramming System*(1968)

## What is this paper talking about?

This is a definition paper, as well as a progress report. In this paper, Dijkstra introduced the first operating system that enables multiple users to share one computer and its peripheral hardware devices. He made an illustration of the general structure of the system his team designed, which is abstract as six hierarchies, as well as some experiences of the philosophy of developing a complex system like this.

## What is the motivation? 

Before then, only one user could use a computer at the same time, i.e. one control flow of a program would progress continuously until its termination, which was inconvenient in different ways. Dijkstra tried to solve the problem, to design a multiprogramming system allowing multiple users to share the computer in the university.

## General ideas

Dijkstra designed a hierarchical system to perform a abstraction of the resources, in which the lower hierarchy would hide and take care of the resources and manipulations below to provide abstract interfaces to the higher hierarchy. There are six hierarchies in total.

- Level 0: It provides the abstract of the sequential processes by handling and monopolizing the clock interrupt. It allocates the precessor and performs mutual synchronization among all the sequential processes. Thus, the actual processor is inmaterial beyond this hierarchy.
- Level 1: It provides the abstract of the segments, which map to the physical storage pages, core pages and drum pages, by handling the drum interrupt. Thus, processes in the higher hierarchy just need to record the identifiers of the segments instead of caring about where the pages are.
- Level 2: It provides the abstract of the message interpreter, which creates a virtual console for every process, through which the the conversations between any of the higher level process and the operator can be carried out.
- Level 3: It provides the abstract of processes associated with I/O, above which the actual peripherals are considered as the logical communication units working in the higher levels.
- Level 4: It provides the abstract of the independent-user programs, which was not implemented yet at that time.
- Level 5: The operator.

Dijkstra also introduced the concept of semaphores in the appendix , the P&V manipulations and indivisible actions, through which synchronization is performed.

## Benefits over the SOTA

This is the first multiprogramming system of course, so it reached the "state of the art" as soon as it was published, so I just talk about some of its benefits,

First, the hierarchy structure provides a clear abstract of the relations of the recourses, the higher level just uses the interfaces the lower one provides, which clarifies and divides the responsibility of each level, as well as hides the complex work that the higher level doesn't need to care about.

Second, the hierarchy structure eases the testing work greatly, for the "relevant states" would be messy and the possible amount would be enormous if all the processes handle all the work. In the hierarchy system, testing could be performed level by level, only after the lower level be completely tested will the next level be verified, and the number of relevant states in a single stage is so small that they can all be tested, as Dijkstra illustrated in the paper.

Third, the hierarchy structure proved to be robust, and the cooperation between all the processes are harmonious thanks to the hierarchy structure. Dijkstra didn't mention about the details, but the general idea is that the  hierarchical structure ensures the finite length of the "call-path" from the high level to the low level.

## Drawbacks of this system

As the first multiprogramming system in the world, it has several drawbacks.

- It didn't provide an interface through which independent users can communicate with each other, as Dijkstra sayed that, they only share the configuation and a procedure library.
- The dispatch system is naive. As far as the paper illustrated in the appendix, the process awake after the V operation is undefined, which may causes something like hunger. This might be a potential problem.

## Conclusion

This paper contributed greatly to the development of the technology of computers, as it introduced the **first** multiprogamming system in the world. Although we might take many concepts in the paper for granted for they may be universal nowadays, the step from zero to one is never an easy thing to do. This paper provides the general idea of how the operating system dealing with multipramming, allocating and synchronizing the resources, which is worthwile reading even after more than 60 years. Also, the hierarchy structure and the design philosophy behind it, that we should construct and varify level by level, both inspire me.

## Reference(Source)

The Structure of the "THE"-Multiprogramming System, Edsger W. Dijkstra, Technological University, Eindhoven, The Netherlands

