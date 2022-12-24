---
layout: post
title:  "Variant Indexes in Data Warehouses"
date:   2022-12-23 09:09:55 -0500
categories: dbms indexes
---
This is my first post!

In this post I'll be discussing the paper [Improved Query Performance with Variant Indexes](https://pages.cs.wisc.edu/~nil/764/DADS/36_improved-query-performance-with.pdf) by Pat O'Neil and Dallan Quass.  [Pat O'Neil](https://en.wikipedia.org/wiki/Patrick_O%27Neil) is a real database legend.  One of the things that he is known for is for co-inventing the [LSM-tree](https://www.cs.umb.edu/~poneil/lsmtree.pdf).  Hopefully I will discuss LSM-trees in a future post.

## Background

First, a little background information.

Different DBMSs are optimized for different workloads.  For example, some systems are optimized to support the actual operations of a business.  Let's imagine a banking application.  Users have to be able to view their balance and other account information, and also make deposits and withdrawals.  In order to service these requests, the application has to issue reads and writes to specific rows of its tables.  This kind of fine-grained access with many reads and writes to the database is what online transaction processing (OLTP) systems must be able to process efficiently.  PostgreSQL and MySQL are examples of OLTP systems.


This is in contrast with online analytical processing (OLAP) type queries, which are commonly used in Data Warehouses.  An example of an OLAP query is "what is the total dollar sales that were made for particular product brand at all stores on the East coast?" Data Warehouses are usually periodically updated in a batch fashion.

These are two very different environments. In one, the operations of the business (application) depend on efficient fine-grained access to the database. In the other, users depend on efficient aggregations and analytics than scan a potentially vast quantity of data (and perform almost exclusively read operations).

This paper focuses on the second environment.  It explains how Data Warehouses can exploit their "read-mostly" environment to use different types of indexes to speed up queries.

Ok, now on to the paper.

## Section 2

Section 2.1 explains that database indexes are typically implemented as B-trees (it also points out that in database documentation, a B-tree usually refers to a B+-tree! I'm very happy that it points this out, because it is something that I've been confused about).  I won't discuss B-trees themselves here (the paper doesn't either) but maybe I will in a future post.

Instead, the paper discusses what is stored in the leaf pages.  At the leaf level of the B-tree are key-value pairs, where the key is value of the indexed column and the value is a list of all Row IDs (RID) that have that column value.  A RID is a reference to a row, note the row itself.  You can think of it like the address of a row.  It specifies where on disk the row is stored.  The paper calls these "Value-List" indexes.  Here's an example of a particular key-value pair stored in the leaf of a B-tree (suppose we have a table with an index on state of residence):

`Maryland: [10, 34, 95, 154, 204]`

Each number here is a reference to a row that has Maryland as its value in its state column.

The paper now suggests an alternate way of expressing these RID-lists called Bitmaps.  First, the paper discusses row numbers.  A row number is an integer in the range `[1,M]` where given a row number, we can efficiently fetch the disk page on which the row is stored. `M` here is the max row number, and there can be no more than `M` rows in the table. I am a little unclear from the paper why a row number can't be used as a RID (maybe it can be?).  In the below discussion, I will use row number and RID synonymously.  Let's also assume that `M` is actually the number of rows stored (not the case when rows can have variable size).

I think bitmaps are best explained with an example so here's one:

`Maryland: 0100100001000...00`

In this example, rows with row numbers 2, 5 and 10 have Maryland for their state value.  The idea is that we store a single bit for every row in the table, and set the bit to 1 for each row number where the row satisfies the condition, and 0 for other rows.

Here's the tradeoff: when a column can take only a few distinct values (let's call the number of distinct values the "cardinality") bitmaps are much more storage efficient.

Here's why: let's assume we have a column indicating whether an individual is married. Every individual is either a married or not.  Therefore, we need two bitmaps to store this information.  This gives us `2M` bits that we need to store in total.  Assuming 32 bits in a row number (RID), a RID list index would require `32M` bits for the same index.  A RID-list index on any column requires `32M` bits because there are `M` rows and a RID requires 32 bits, and each RID appears once.  So the storage savings is huge.

Now let's take the state example from above.  Assuming we are only storing US residents, every individual lives in exactly one state.  And there are 50 states, we need to store 50 different bitmaps (one for each state).  In this case the uncompressed size of a bitmap index is slightly worse than RID-lists (`32M` bits vs `50M`). The "density" of a bitmap is `1/50` because on average, for a particular bitmap, `1/50` of the bits are set to 1.  However, since the density of this bitmap is low, the bitmaps will be very compressable (there will be many "runs" of 0s and 1s).  So bitmaps still may be smaller.  (I think they would be because most states are pretty rural, and the bitmaps associated with these states would be very compressable.)

Another benefit of bitmaps is they allow bitwise operations on sets of rows.  If you have two bitmaps describing two set of rows satisfying different predicates, you can take the AND of these predicates by doing a bitwise operation, likewise for OR and NOT [^1].



[^1] I am neglecting to mention the existence bitmap (EBM) required for computing NOT.





