---
layout: post
title:  "Variant Indexes in Data Warehouses"
date:   2022-12-28 09:09:55 -0500
categories: dbms indexes
usemathjax: true
---

In this post I'll be discussing the paper [Improved Query Performance with Variant Indexes](https://pages.cs.wisc.edu/~nil/764/DADS/36_improved-query-performance-with.pdf) by Pat O'Neil and Dallan Quass.  [Pat O'Neil](https://en.wikipedia.org/wiki/Patrick_O%27Neil) is a real database legend.  One of the things that he is known for is for co-inventing the [LSM-tree](https://www.cs.umb.edu/~poneil/lsmtree.pdf).  Hopefully I will discuss LSM-trees in a future post.

# Background

First, a little background information.

Different DBMSs are optimized for different workloads.  Some systems, for example, are optimized to support the actual operations of a business.  Let's imagine a banking application.  Users have to be able to view their balance and other account information, and also make deposits and withdrawals.  In order to service these requests, the application has to issue reads and writes to specific rows of its tables.  This kind of fine-grained access with many reads and writes to the database is what online transaction processing (OLTP) systems must be able to process efficiently.  PostgreSQL and MySQL are examples of OLTP systems.


This is in contrast with online analytical processing (OLAP) type queries, which are commonly used in Data Warehouses.  An example of an OLAP query is "what is the total dollar sales that were made for all Pepsi products at all stores on the East coast?" Data Warehouses are usually periodically updated in a batch fashion.

These are two very different environments. In one, the operations of the business (application) depend on efficient fine-grained access to the database. In the other, users depend on efficient aggregations and analytics than scan a potentially vast quantity of data (and perform almost exclusively read operations).

This paper focuses on the second environment.  It explains how Data Warehouses can exploit their "read-mostly" environment to use different types of indexes to speed up queries.  These techniques don't help much in the OLTP case, because these indexes need to be synchronously maintained on every insert, which results in a huge write amplification.  Furthermore, they don't benefit point queries that much (except for the Value-List index, which is really a normal index).

There's one last thing I want to mention.  When evaluating the cost of an algorithm, we will count how many disk pages the algorithm reads into memory.  These algorithms attempt to minimize the number of I/Os that are required to service a SUM request of a column.  What these algorithms want to avoid is spending an I/O, only to get a few column values from it.  If we're going to spend an I/O, we want that page to be chock full of relevant data.

Ok, now on to the paper.

# Section 2: Variant Indexes

## Value-List Indexes

First, the paper defines "Value-List" indexes.  A Value-List index is a bunch of key-value pairs, where the key is the column being indexed, and the value is a list of Row IDs (RID) where the corresponding Row has that value.  A RID is a reference to a row, not the row itself.  You can think of it like the address of a row.  It specifies where on disk the row is stored.

Here's an example of a Value-List index (suppose we have a table with an index on state of residence):

```
Alabama: [2, 97, 1002]
Alaska: [4, 8]
Arizona: [1010, 3001]
...
Maryland: [10, 34, 95, 154]
...
Wisconsin: [204, 4532]
Wyoming: [74, 89]
```

Each number here is a reference to a row in the table.

The paper also briefly mentions B-trees.  I won't explain B-trees here (maybe I will in a future post).  B-trees are how we efficiently lookup a particular list in a Value-List index.  The key-value pairs are stored in the leaf level pages of a B-tree.  That way, instead of scanning the RID-Lists one by one, we can traverse a tree to find the correct RID-list.


The paper now suggests an alternate way of expressing these RID-lists called Bitmaps.  First, the paper discusses row numbers.  A row number is an integer in the range `[1,M]` where given a row number, we can efficiently fetch the disk page on which the row is stored. `M` here is the max row number, and there can be no more than `M` rows in the table. I am a little unclear from the paper why a row number can't be used as a RID (maybe it can be?).  In the below discussion, I will use row number and RID synonymously.  Let's also assume that `M` is actually the number of rows stored (not the case when rows can have variable size).

I think bitmaps are best explained with an example so here's one:

`Maryland: 0100100001000...00`

In this example, rows with row numbers 2, 5 and 10 have Maryland for their state value.  The idea is that we store a single bit for every row in the table, and set the bit to 1 for each row with column value "Maryland" and 0 for all others.

Here's the tradeoff: when a column can take only a few distinct values (let's call the number of distinct values the "cardinality") bitmaps are much more storage efficient. Being storage efficient is good, because it means we need to spend fewer I/Os to read the index into memory.

To understand why, let's imagine column for marital status. Every individual is either married or not.  Therefore, we need two bitmaps to store this information.  This gives us `2M` bits that we need to store in total.  Assuming 32 bits in a row number (RID), a RID list index would require `32M` bits for the same index.  That's because every row number has to appear in some RID-list (this is true for an index on any column).

Now let's take the state example from above.  Assuming we are only storing US residents, every individual lives in exactly one state.  And there are 50 states, we need to store 50 different bitmaps (one for each state).  In this case the uncompressed size of a bitmap index is slightly worse than RID-lists (`32M` bits vs `50M`). The "density" of a bitmap is `1/50` because on average, for a particular bitmap, `1/50` of the bits are set to 1.  On the other hand, since the density of this bitmap is low, the bitmaps will be very compressable (there will be many "runs" of 0s and 1s).

Another benefit of bitmaps is they allow bitwise operations on sets of rows.  If you have two bitmaps describing two set of rows satisfying different predicates, you can take the AND of these predicates by doing a bitwise operation, likewise for OR and NOT.

These bitwise operations are much faster than looping over two RID lists.

We can also compute COUNT on a bitmap.  We want simply count the number of 1s in the bitmap.  This is also easily parallelized by partitioning the bitmap and computing counts on each segment.

Now note that a bitmap most likely cannot fit onto a single 4KB disk page (or even a larger 32KB page). Assuming 8,000,000 rows, there are 8,000,000 bits = 1MB in a bitmap.  So this will require 250 4KB pages.  This requires us to partition the bitmap, with each partition corresponding to a row segment of the database table.

Now, Value-Lists can have a combination of RID-lists and bitmaps.  For dense enough keys (greater than $\frac{1}{32}$), we can use a bitmap and otherwise use a RID-list.  We can even make it so that some fragments use a bitmap and others use a RID-list (within a single key-value pair).


## Projection Indexes

The paper now describes projection indexes.  A projection index is a copy of all values of a column stored contiguously on disk.  This is efficient for the case that all column values need to be retrieved for a particular foundset, but the entire rows are not needed.  For example, assuming a 4 byte column field, then 1000 values will fit on a single 4KB page.  If the 4 byte column fields are part of a 200 byte rows, then only 20 rows fit on a 4KB page.  And to read those same 1000 values, at least 50 pages need to be read.


## Bit-Sliced Indexes

The paper now describes bit-sliced indexes.  This is best explained with an example also.  Suppose we have the following table named SALES which has a row for every sale.  And that table has a column dollar_sales for the dollar amount of the sale.  I will also write the integer dollar amount in base 2 (suppose this number represents the number of pennies).

<img src="https://raw.githubusercontent.com/awitten1/awitten1.github.io/master/images/bit-sliced-index.png" alt="drawing" width="300"/>

In this example, we have 4 rows in our table and each has a dollar_sales value.  To form a bit-sliced index, we compute the the bitmaps indicated by the colorful vertical boxes above.  Now we have a bunch of bitmaps expressing the following predicate: the i<sup>th</sup> bit of this field is a 1.  So to compute the column value of the row with row number i, we can scan the i<sup>th</sup> bit of each of the bitmaps.

This is in one sense a different way to view projection indexes.  Bit-sliced indexes store all the same bits as projection indexes, just in a different order.  With projection indexes a single column value is stored contiguously on disk.  With Bit-sliced indexes, all the 0th bits for all column values are stored contiguously.  Same for the 1st bits, 2nd bits, and etc.  In theory this requires the same amount of storage as the projection indexes.  However, with bit-sliced indexes, we don't need to store bitmaps that are all 0.  In addition, bit-sliced indexes can potentially achieve much higher compression ratios.  For example, consider the dollar_sales example.  Let's suppose the column represents the number of pennies in a sale.  The vast majority of sales will require fewer than 20 bits (because most sales are less than $$2^20 \approx 1,000,000 = 10,000\$ $$).  Therefore, assuming 32 bit fields, the higher order bitmaps will be almost entirely 0s.  This can yield very high compression ratios.

We now have established 3 types of indexes: Value-List indexes using RID-lists and bitmaps, Projection Indexes, and Bitsliced Indexes.

# Section 3: Evaluating Cost

This section walks us through a few potential query plans for a query of the form:

`SELECT SUM(dollar_sales) FROM SALES WHERE condition;`

We make the following assumptions:
* 100 million rows in the table
* 200 bytes per row => 20 rows per 4KB page
* Foundset satisfying `condition` has already been computed and is represented by bitmap `B`<sub>`f`</sub>.  This is an important assumption and depends on existing Value-List indexes (which themselves are indexed with B-trees).

I won't be discussing the CPU cost in much detail here.

## Query Plan 1: read rows in foundset directly to calculate the aggregation.

We want to compute how many disk pages we need to read in (how many I/Os we need to potentially do).  The calculation goes like this:


```
100 million rows
20 rows in a page
=> 100 million / 20 = 5 million pages in the table.
 ```

What is the probability that a particular page stores a row we need? Well, what is the probability that a particular page does not store a row we need?  That is the probability that none of the `2 million` rows in a foundset is on this page. That is $$(1 - 1/5,000,000)^{2,000,000} = (1 - \frac{(2,000,000/5,000,000)}{2,000,000})^{2,000,000} \approx e^{-2/5}$$

Which means the probability that a particular page stores a row we need is:

$$(1 - e^{-2/5})$$

What's the expected number of pages we need to read in?  Using linearity of expectation and indicator random variables it is:

$$5,000,000(1 - e^{-2/5}) = 1,648,400$$ disk pages

That's a lot of pages!  Assuming a modern AWS gp3 disk (which is an SSD) a reasonable number of IOPS is 3000.  And, AWS assumes 16KB pages.  So, this gives us a total of `(1,648,400 pages / 4)/ 3000 IOPS = 137 seconds`

(I am assuming a much higher performance disk than what the paper assumes.)

Not terrible, but a little annoying to wait over two minutes for the query to return.


## Query Plan 2: Use Projection Index.

Assuming the dollar_sales field is 4 bytes

=> 1000 dollar_sales fields per page => `100 million / 1000 = 100,000 pages` for all fields.

Assuming we read all pages, that gives us a response time of: `(100,000 pages / 4) / 3000 IOPS = 20 seconds`.  Much better than plan 1!


## Query Plan 3:  Use Value-List index on dollar_sales.

Assume `10,000` unique dollar_sales values.

We need to read in each RID-list/bitmap!  Assuming they are all RID-lists, we need to read in `100 million rows * 4 bytes per RID = 400 million bytes => 100,000 pages`.  For each RID-list/bitmap, we need to compute the cardinality of its intersection with the foundset.  Then multiply its cardinality with the corresponding value and add it to a total.

This has the same I/O count as query plan 2, but is much more CPU intensive.

One tradoff to note: this will have a smaller I/O cost than plans 2 or 4 if the column field is large.  In this paper the assumption is the field is 4 bytes and a RID is also 4 bytes, so it just so happens that they are the same.


## Query Plan 4: Using a Bit-Sliced index.

In query plan 4, the paper makes an assumption that looks unsafe to me.  It assumes that you only need 21 bitmaps for the bit-sliced index.  The idea is that you don't need to store bitmaps that are all 0.  That makes sense, but it's still not clear to me that for 32 bit fields, we would have 11 empty bitmaps.  I think the assumption is that after 2<sup>20</sup> ~ 1,000,000 pennies = $10,000 there are no more sales at all. I don't think that is a safe assumption.

What is true, however, is that the compression ratio for the higher order bitmaps will be insane.  They will compress to almost nothing, because the assumption is that the vast majority of sales are <= $100, and therefore the higher order bitmaps will be almost entirely 0s.

Therefore it's a little hard to estimate the I/O cost for this plan.  Not considering compression, it's the same as plan 2.  But given the compression, I think there would be significant I/O savings. (And actually, the higher order bitmaps could use a RID-list.)

Maybe down to ~75,000 bytes? I'm not exactly sure.

## Clustering

The projection index and the bit-sliced index benefit from clustering.  If the foundset is clustered on a fraction f of the disk, then with those indexes you only need to do that ratio of the above I/O.

## Other aggregation functions

The paper now goes on to discuss other column aggregate functions, and which index is the most useful for which.  None are needed for COUNT, since we already have the foundset.  Value-List is best for MAX and MIN.  Value-List is best for percentiles (surprisingly, the bit-sliced index can also be used in a mysterious way for this).

# Section 4: Evaluting Range queries

```
SELECT column-list FROM T WHERE C-range AND <condition>
```

where C-range is a range predicate: `{C > c1, C >= c1, C = c1, C < c2, C <= c2, C between c1 and c2}`

And the condition is already in a foundset B<sub>f</sub>.

Using a projection index: for each row in B<sub>f</sub> look in projection index and test the range.

Using a Value-List index:  for each value in range in the Value-List index, compute the union of the lets (using bitwise OR).  Then AND that with B<sub>f</sub>.

Using a Bit-Sliced index: Interestingly, this can be done efficiently.  It's pretty confusing.  I think it deserves a short blog post on its own.

# Section 5

First, some more background information.  A common schema in OLAP databases is the star schema.

<img src="https://raw.githubusercontent.com/awitten1/awitten1.github.io/master/images/star-schema.png" alt="drawing" width="300"/>

The SALES table is the so-called "fact table."  A fact table represents a set of observations, in this case sales.  The referenced tables (by a foreign key) are "dimension tables."

An example OLAP query is

```
SELECT P.brand, T.week, C.city, SUM(S.dollar_sales)
 FROM SALES S, PRODUCT P, CUSTOMER C, TIME T
 WHERE S.day = T.day and S.cid = C.cid
and S.pid = P.pid and P.brand = :brandvar
and T.week >= :datevar and C.state in
('Maine', 'New Hampshire', 'Vermont',
 'Massachusetts', 'Connecticut', 'Rhode Island')
 GROUP BY P.brand, T.week, C.city;
```

A query such as the above one can be precomputed in a summary table for all possible values of T.day, C.cid, P.pid.  That is called the "OLAP cube" or "Datacube."

This incurs a large price on updates.  Furthermore, the size of the summary table grows with the product of the number of values in the dimension tables, which makes it hard to precompute everything.  Also, some fields may be non-dimensional, like temperature, and that negates the benefit of the summary table (these numeric fields drawn from a continuous space should almost never have dimension tables).

In short, you can't precompute everything, and we still need strategies for efficiently computing these types of queries.


## Join Indexes
```
"A Join Index is an index on one table for a quantity that involves a column value of a different table through a commonly encountered join."
```

These are regular indexes, except that they are on a "virtual column" that is brought in through a join.  For example, in the SALES table above we can have an index on brand, even though that column is in a dimensional table.

This index can be a Projection, Value-List, or Bit-Sliced index.  Having enough join indexes, may remove the need for an explicit join altogether.

## Groupset Aggregates

In a query like the above one, once we have a foundset satisfying the WHERE clause, we must then partition the set into groups (groupsets) and compute the aggregation for each groupset.

Using projection indexes: for each row of the foundset, read from each projection index (one for each group by dimension), accumulating in an array cell corresponding to that groupset.

Using Value-List indexes:

For all combinations of values of columns we are grouping on, compute the intersection of the bitmaps.  Intersect that with the foundset.  Compute the agg of that intersection.


[^1] I am neglecting to mention the existence bitmap (EBM) required for computing NOT.





