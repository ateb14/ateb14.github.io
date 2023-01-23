---
title: LegoOS
date: 2022-12-10 15:25:25
tags:
- operating system
- paper reading
categories:
- operating system

---

# Writeup for *LegoOS: A Disseminated, Distributed OS for Hardware Resource Disaggregation*

## What is this paper talking about?

This is a definition paper which introduces a new OS design paradigm, LegoOS, which decouples the process, memory and storage components in a system, where any of which communicates with others only through network and the failure of a single hardware won't affect others.

## What is the motivation? 

Previous servers, especially monolithic servers, suffer from its lack of flexibility as different types of hardwares are clustered together through local bus, resulting in several key limitations:

- Inefficient resource utilization resulted from the constraint that CPU and memory for a job has to be allocated from the same physical machine.
- Poor hardware elasticity to configure the hardware components.
- Coarse failure domain which will fail the whole sever when only a single hardware component fails.
- Bad support for heterogeneity which makes it hard to add more new hardware devices.

With the rapid growth of network speed, it's possible to separate different hardware components and connect them with network instead of local bus. These hardware components thus compose the whole system like Lego.

## General ideas

The whole system is composed of three kinds of hardware components: process(pComponent), memory(mComponent) and storage(sComponent), each of which owns a monitor that manage the component and communicates with others through network. As a result the functionalities of the system are split. 

A monitor runs on its specific hardware component, thus it can use its own implementation to manage the hardware component that it runs on. So to deploy a new hardware device, its developers only need to implement the monitor of the device and simply attach it to the network of the system.

The components of the system are not coherent, and only the coherence that the hardware provides within a component is guaranteed. This is because maintaining coherence between different components will cause high network bandwidth consumption. However, the applications running on the kernel are still able to implement their desired level of coherence.

The kernel also globally manages resources, but only occasionally for coarse-grained decisions to mininize performance and scalability bottleneck, while the monitors of the components will make their own fine-grained decisions.

To handle component failures, the kernel adds redundancy for recovery.

## Benefits

- The hardware devices are loosely coupled, which makes it much more simple to add new hardware devices and configure the hardware components.
- The recource utilization can be improved by managing the load balance of different hardware components.
- A failure of a single component won't affect the whole system, which makes the system more reliable.
- The management policy of a hardware component can be customized, leading to better flexibility of the hardware components.

## Drawbacks

- The resources of a single application are possible to be scattered into different components, which will not only create a locality loss, but also increase the possiblity for a application to suffer from hardware failures.
- The newwork speed is a bottleneck of LegoOS.

## Conclusions

LegoOS demonstrates a new OS paradigm that shows us the possibility to build a highly-scalable and loosely-coupled system based on network. With the development of the network technology, this paradigm is promising.

