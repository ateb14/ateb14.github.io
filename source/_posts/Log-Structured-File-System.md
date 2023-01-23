---
title: Log-Structured File System
date: 2022-11-23 15:13:45
tags:
- operating system
- paper reading
categories:
- operating system
---

# Writeup for *The Design and Implementation of a Log-Structured File System*(1991)

## What is this paper talking about?

A log-structured file system, Sprite LFS, which is similar to the write-ahead logging mechanism used in database systems is introduced in this paper, where the full file system is considered as a log and the writing operations are taken as appending contents to the log.

## What is the motivation? 

The larger scale of the main memory makes more files be cached in them, increasing the efficiency of the read requests for the disk. As a result, the write requests will dominate the disk traffic. Traditional file systems spend a lot of time on seeking the metadata for the write requests, which lowers the utilization of the disk bandwidth dramatically especially for frequently writing small files. Furthermore, traditional file systems tend to write the metadata stuctures synchronously even the blocks are written asynchronously, which bounds the CPU utilization. The log-structured file system aims at solving these problems by buffering a sequence of write requests in the file cache and writing all of them sequentially in a single I/O, including both the metadata and the file data.

## General ideas

The new data scheduled to be flushed to the disk are cached in main memory until their amount exceeds a curtain threshold, after which the large collection of data is written to the disk in a single I/O to make full use of the disk's bandwidth. Or we can say that, it collects the scattered small files and writes them to the disk together periodically. The write operations are append-only, which means that the positions of the inodes aren't fixed, and thus they are recorded in an inode map.

To guarantee enough spaces for a write operation, the log-structured file system combines both threading and copying. Organized by fixed-size segments which contain certain number of blocks, any given segment is always written sequentially from its beginning to its end, and all live data must be copied out of a segment before the segment can be rewritten. The log is threaded on a segment-by-segment basis, and the file system applies a reclamation policy that compacts the contents by segments, in which the contents of the live blocks are written out to the head of the log.

The cost for a group of segments to be cleaned is highly relative to its utilization, i.e., the more blocks are alive, the higher the cost is. Whereas, the lower utilization indicates the lower performance, so the log-structured file system tries to make a tradeoff between cost and performance by seperating the disk into two classes, the first of which are nearly full, while the second has an extremely low utilization on which the cleaner works on.

Like the database management system, the log-structured file system employs a checkpoint mechanism in the log to perform crash recovery, and it tries to recover the operations happened before an inconsistent checkpoint using some special logs.

## Benefits

- The buffering and append-only writing mechanism reduces the time to seek the metadata, which increases the utilization of disk bandwidth dramatically.
- Cleaning overheads are low with a simple but useful policy based on cost and benefit, and there is no cleaning overhead at all for very large files that are created and deleted in their entirety.
- Crash recovery is quick since the positions of the last disk operations are at the end of the log, which means that it's unnecessary to scan all the metadata structures to restore consistency.

## Drawbacks

- Since the files to be writen to the disk are cached in a buffer and are appended to the end of the log, the file system is unable to reorganize them to achieve some logical locality, which indicates that a potential performance loss will occur when the read pattern doesn't fit in the temporal locality that the log creates.
- The log-structured file system is actually not compatible with the commonly used file systems, which makes it hard to grow widespread.

## Conclusion

The prediction made behind the log-structured file system that the scale of the main memory will make the disk traffic be dominated by write requests are proved to be correct nowadays, which highlights the idea of buffering the small file write requests. What's more, the cleaning mechanism is quite similar to the physical attributes of the SSD, where a block must be erased before being written again, so I suppose that the general ideas of the log-structured file system are also useful in SSD hardware design. In conclusion, the idea of this file system is still inspiring today from my perspective.

