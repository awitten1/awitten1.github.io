---
layout: post
title:  "Variant Indexes in Data Warehouses"
date:   2022-12-23 09:09:55 -0500
categories: dbms-indexes
---
This is my first post!

In this post I'll be discussing the paper [Improved Query Performance with Variant Indexes](https://pages.cs.wisc.edu/~nil/764/DADS/36_improved-query-performance-with.pdf) by Pat O'Neil and Dallan Quass.  [Pat O'Neil](https://en.wikipedia.org/wiki/Patrick_O%27Neil) is a real database legend.  One of the things that he is known for is for co-inventing the [LSM-tree](https://www.cs.umb.edu/~poneil/lsmtree.pdf).  Hopefully I will discuss LSM-trees in a future post.

First, a little background information.

Systems such as Postgres and MySQL are optimized to handle online transaction processing (OLTP) workloads. In these workloads there are many read and write queries to the system, and most queries are fine-grained.  This is the workload that an application depends on in order to function properly.  For example, imagine a banking application.  If I want to view my bank account information, the web server will have to retrieve from its DBMS informatiom related to me, which will require it to read specific rows from its tables (this is what I mean by fine-grained).  If I want to make a deposit, the system will have to perform updates to specific rows. This workload requires a system that can efficiently perform fine-grained queries and a relatively high percent of writes.

This is in contrast with online analytical processing (OLAP) workloads, which data warehouses are optimized to handle efficiently.  These systems are optimized for an environment where the vast majority of queries are read-only, and updates are done only periodically, in a batch fashion.  In addition, most of these analytical queries usually need to read a huge number of rows, but only a few columns.

