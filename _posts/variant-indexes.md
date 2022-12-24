---
layout: post
title:  "Variant Indexes in Data Warehouses"
date:   2022-12-23 09:09:55 -0500
categories: dbms-indexes
---
This is my first post!

In this post I'll be discussing the paper [Improved Query Performance with Variant Indexes](https://pages.cs.wisc.edu/~nil/764/DADS/36_improved-query-performance-with.pdf) by Pat O'Neil and Dallan Quass.  [Pat O'Neil](https://en.wikipedia.org/wiki/Patrick_O%27Neil) is a real database legend.  One of the things that he is known for is for co-inventing the [LSM-tree](https://www.cs.umb.edu/~poneil/lsmtree.pdf).  Hopefully I will discuss LSM-trees in a future post.

First, a little background information.

Different DBMSs are optimized for different workloads.  For example, some systems are optimized to support the actual operations of a business.  Let's imagine a banking application.  Users have to be able to view their balance and other account information, and also make deposits and withdrawals.  In order to service these requests, the application has to issue reads and writes to specific rows of its tables.  This kind of fine-grained access with many reads and writes to the databse is what OLTP systems are optimized for.  PostgreSQL and MySQL are examples of OLTP systems.


This is in contrast with online analytical processing (OLAP) type queries, which are commonly used in Data Warehouses.  An example of an OLAP query is "what is the total dollar sales that were made for particular product brand at all stores on the East coast?" Data Warehouses are usually periodically updated in a batch fashion.

These are two very different environments. In one, the operations of the business (application) depend on efficient fine-grained access to the database. In the other, users depend on efficient aggregations and analytics over a potentially vast quantity of data (and perform almost exclusively read operations).

This paper focuses on the second environment.  It explains how Data Warehouses can exploit their "read-mostly" environment to use different types of indexes to speed up queries.

