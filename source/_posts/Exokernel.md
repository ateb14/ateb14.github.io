---
title: Exokernel
date: 2022-11-09 23:33:11
tags:
- operating system
- paper reading
categories:
- operating system
---

# Write up for *Exokernel: An Operating System Architecture for Application-Level Resource Management*(1995)

## What is this paper talking about?

This is a definition paper that introduces a new kind of kernel, exokernel, which is quite different from the traditional kernels for that it exports almost all the physical resources to the untrusted user libraries in a secure way, by which the physical recourses are protected and are revokable. User library systems can implement, for example, virtual memory mechanism and user-defined exception handlers based on their demands.

## What is the motivation?

The traditional operating systems manage the physical resources and provides high-level abstractions for user programs, which are usually built-in privileged programs that fully manipulate all the physical resources and are shard by all users, and therefore making them impossible to get changed by user programs. Although these high-level abstractions separate the physical resouces from users' view and thus makes them easier to manage and much safer for use, they leave no space for users to make some adjustments to make full use of the physical resources, which will bring down the performance due to both the cost of abstractions and the low flexibility for users to exploit the physical resources directly. Therefore, exokernel manages to lower these costs of abstractions by exporting the physical resources to user libraries in a secure way.

## Geneal Ideas

The design of exokernel comes from a simple observation that low-level primitives can be implemented in more efficient ways, as well as that low-level primitives are much more extensible for the higher-level abstractions to be implemented on. Exokernel manages to grant the library systems the maximum freedom to manipulate the physical resources, while protecting them from each other for which errors in a library system are ought not to affects other systems, by separating protection from management.

In order to achieve the separation, exokernel is implemented with a secure bindings mechanism which tracks the ownership of resources and perform protections on binding points and all usages, a visible revocation protocol allowing library systems to participate in the resource revocations, and an abort protocol through which exokernel breaks secure bindings owned by uncooperative library systems by force.

User library systems, ExOS demonstrated in this paper for example, are able to fully exploit the physical resources as what they want to, for almost all the necessary hardware information is delivered to them from exokernel. 

## Benefits

- The low-level primitives are much more efficient than that of the traditional kernels, for they only require several instructions to be performed, as shown in the experiment results displayed in this paper that the cost for exokernel to dispatch an exception is relatively minor to that of Ultrix, a mature monolithic OS.
- Abstractions of the traditional kernels can be implemented efficiently on the user library systems, one possible reason for which is that kernel transitions are eliminated because abstractions are implemented at application level, and another reason may be that application-level abstractions can be implemented in ways that are not supported by the traditional kernels.
- Special-purpose implementations of abstractions are accessible to user programs by changing or modifying the library systems used in exokernel. As we know that, traditional kernels that implement high-level abstractions need to make a trade-off among various cases, so the abstractions are quite general, i.e. they focus on the average performance of multiple cases rather than that of some specific cases that are clear to the user programs and the user programs may suffer from the cost of the general abstractions. Therefore, the extensibility of exokernel allows user programs to gain from specific abstractions of library systems that best fit to them.

## Drawbacks

- The abstractions are implemented at user level, which is quite contradictory to the design philosophy of the traditional kernels and thus is unfamiliar to most programmers, making it difficult to develop and maintain.
- It's hard to maintain the compatibility between different library systems, as they are not proposed as a whole system, which means that different library systems may have distinct purposes and they may conflict with each other.

## Conclusions

Abstractions definitely raise costs, but only the user programs are clear about what kind of abstractions they want for the best performances, so exokernel transfer the power of building abstractions to user libraries which enables users to select the right abstractions they need to lower the cost of abstractions as much as possible and make full use of the physical resources. This philosophy is widely used in different domain, for example, domain-specific-architectures in computer architecture. If we just want the machine to simply execute our specific tasks at all its exertions instead of taking all the general tasks into consideration, the design logic of exokernel is worthwhile being taken account of.
