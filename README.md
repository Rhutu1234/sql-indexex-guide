# SQL Indexes

*A deep-dive walkthrough of database indexes — covering the B-tree structure underneath most indexes and precisely why it makes lookups fast, clustered vs. non-clustered indexes, the real write-side cost every index imposes, composite indexes and why column order is a genuine design decision, covering indexes and index-only scans, selectivity as the property that determines whether an index actually helps, reading a query execution plan to know whether an index is genuinely being used, and how to create and manage indexes through EF Core's migrations.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [The Book Index Analogy, Taken Seriously](#1-the-book-index-analogy-taken-seriously)
3. [The B-Tree: The Structure Underneath Most Indexes](#2-the-b-tree-the-structure-underneath-most-indexes)
4. [Why a B-Tree Makes Lookups Fast: The Actual Math](#3-why-a-b-tree-makes-lookups-fast-the-actual-math)
5. [Clustered vs. Non-Clustered Indexes](#4-clustered-vs-non-clustered-indexes)
6. [The Real Cost: Indexes Slow Down Writes](#5-the-real-cost-indexes-slow-down-writes)
7. [Composite Indexes and Column Order](#6-composite-indexes-and-column-order)
8. [Covering Indexes and Index-Only Scans](#7-covering-indexes-and-index-only-scans)
9. [Selectivity: The Property That Determines Whether an Index Helps at All](#8-selectivity-the-property-that-determines-whether-an-index-helps-at-all)
10. [Reading a Query Execution Plan](#9-reading-a-query-execution-plan)
11. [Unique Indexes vs. Unique Constraints](#10-unique-indexes-vs-unique-constraints)
12. [When NOT to Add an Index](#11-when-not-to-add-an-index)
13. [Creating Indexes Through EF Core](#12-creating-indexes-through-ef-core)
14. [Common Pitfalls](#13-common-pitfalls)
15. [Quick Reference Table](#quick-reference-table)
16. [Conclusion](#conclusion)

---

## Introduction

A database index exists for exactly the reason its name suggests — the same reason a book has one — but understanding *precisely* how it delivers that speedup, and precisely what it costs to maintain, is what separates indexing as a genuine, deliberate design discipline from indexing as guesswork ("add an index and see if it helps"). This guide goes deep on the actual data structure most relational database indexes are built from (a B-tree), the real, asymptotic reason that structure makes lookups fast, the meaningful distinction between a clustered and a non-clustered index, and the write-side cost every index imposes that makes "just add more indexes" a genuinely bad default strategy rather than a free performance win.

```plaintext
WITHOUT an index: finding "all orders with CustomerId = 42" means
  scanning EVERY ROW in the table, checking each one — O(n).
WITH an index on CustomerId: the database navigates a B-TREE structure
  (Section 2) directly to the matching rows — O(log n), a fundamentally
  different, dramatically faster growth curve as the table grows.
```

---

## 1. The Book Index Analogy, Taken Seriously

### A book's index lets you skip straight to a page, instead of reading the whole book to find a topic

```plaintext
Without an index: find every mention of "garbage collection" by reading
  the ENTIRE book, page by page.
With an index: look up "garbage collection" in the index (itself
  alphabetically sorted, making IT fast to search), and jump directly to
  the specific pages it lists.
```

This analogy is genuinely accurate, not just a simplified metaphor — a database index really is a separate, auxiliary structure, sorted by the indexed column(s), that lets the database engine locate matching rows without examining every row in the table, exactly the same principle a book's alphabetically-sorted index applies to finding a topic without reading cover to cover.

### Where the analogy needs extending: an index has to be actively MAINTAINED as the underlying data changes

```plaintext
A book's index is fixed once printed — but a DATABASE TABLE changes
  constantly (rows inserted, updated, deleted), and every index on that
  table must be kept accurate and up to date with EVERY such change,
  which is precisely Section 5's real, ongoing cost the book analogy
  doesn't have an equivalent for.
```

---

## 2. The B-Tree: The Structure Underneath Most Indexes

### A balanced, sorted tree structure — not a simple sorted list

```plaintext
A B-tree organizes indexed values into a tree of NODES, each holding
  multiple sorted keys and pointers to CHILD nodes covering sub-ranges
  of values — structured so that starting from the ROOT and following
  pointers downward, based on comparing the search value against each
  node's keys, reaches the target value in a SMALL, BOUNDED number of steps.
```

```plaintext
                    [50]
                 /        \
            [20, 35]      [70, 90]
           /   |   \      /   |   \
       [..] [..] [..]  [..] [..] [..]   ← LEAF nodes, holding the actual indexed values (and row pointers)
```

This is genuinely worth visualizing precisely, not just accepting as a black box — a B-tree is *balanced*, meaning every leaf sits at (approximately) the same depth from the root, which is the structural property Section 3 shows is what actually delivers the logarithmic search time; it's also specifically designed around how disk/page-based storage works, with each node sized to fit efficiently into one storage "page," minimizing the number of separate disk reads a single lookup requires.

### Why leaf nodes, specifically, matter for range queries

```plaintext
Leaf nodes in a B-TREE (specifically, the B+TREE variant most database
  engines actually use) are typically LINKED together in sorted order —
  once you've navigated DOWN to the correct starting leaf, a RANGE query
  ("WHERE Price BETWEEN 10 AND 50") can simply walk SIDEWAYS across
  linked leaves, rather than needing to re-navigate the tree from the
  root for every matching row.
```

This is a genuinely important structural detail — it's precisely why B-tree indexes are so well-suited to range queries (`BETWEEN`, `>`, `<`, `ORDER BY`) and not just exact-match lookups: once positioned at the right starting point, retrieving a contiguous range of sorted values is a cheap, linear walk across linked leaves, not a repeated series of independent tree traversals.

---

## 3. Why a B-Tree Makes Lookups Fast: The Actual Math

### The growth curve that actually matters: O(log n) vs. O(n)

```plaintext
Table scan (no index): to find a matching row among N rows, in the
  WORST case, examine ALL N of them — cost grows LINEARLY with table size.
B-tree lookup: cost grows LOGARITHMICALLY with table size — because
  each step down the tree eliminates a large FRACTION of the remaining
  possibilities (per node), not just one row at a time.
```

### Making this concrete with real numbers

```plaintext
A table with 1,000,000 rows:
  Table scan: up to 1,000,000 row examinations, worst case.
  B-tree lookup (with, say, ~100 keys per node): roughly log₁₀₀(1,000,000)
    ≈ 3 LEVELS to traverse — a handful of comparisons, not a million.
```

This is worth internalizing as a genuinely dramatic, not marginal, difference — the gap between "examine up to a million rows" and "traverse roughly three tree levels" isn't a modest speedup; it's the difference between a query that's noticeably slow at real data volume and one whose performance barely changes as the table grows from a thousand rows to a billion, which is precisely why proper indexing is one of the highest-leverage things you can do for database query performance, directly connecting to this series' High-Volume Transaction Processing guide's own emphasis on avoiding operations whose cost scales badly with volume.

---

## 4. Clustered vs. Non-Clustered Indexes

### Clustered index: the table's actual, physical row storage order

```plaintext
A CLUSTERED index doesn't just point TO the data — it IS the physical
  order the table's rows are stored in, on disk. A table can have AT
  MOST ONE clustered index, precisely because rows can only physically
  be stored in ONE order at a time.
```

Per SQL Server's (and most relational databases') default convention, the clustered index is typically built on the primary key — this means rows are physically stored in primary-key order, and a lookup by primary key is genuinely the fastest possible access pattern, since the B-tree's leaf nodes *are* the actual row data, not a separate pointer to it elsewhere.

### Non-clustered index: a separate structure, pointing BACK to the actual row

```plaintext
A NON-CLUSTERED index is a genuinely SEPARATE B-tree, sorted by whatever
  column(s) it covers — its LEAF nodes don't hold the full row; they
  hold the indexed value PLUS a pointer (the clustered index's key, or a
  raw row locator) back to where the FULL row actually lives.
```

This is the structural reason a non-clustered index lookup, in the general case, involves an extra step (Section 7 covers the exception): the database first navigates the non-clustered index's B-tree to find the matching indexed value, then follows that pointer to the clustered index (or the underlying heap) to retrieve the row's remaining columns — this second step is often called a "key lookup" or "bookmark lookup," and it's a real, additional cost worth knowing about explicitly.

### A table can have many non-clustered indexes, each optimized for a different query pattern

```sql
CREATE INDEX IX_Orders_CustomerId ON Orders(CustomerId);
CREATE INDEX IX_Orders_Status ON Orders(Status);
CREATE INDEX IX_Orders_CreatedAt ON Orders(CreatedAt);
```

Unlike the single clustered index, a table can have as many non-clustered indexes as genuinely needed, each one supporting fast lookups on a different column or combination of columns — the trade-off (Section 5) is that every one of these needs to be independently maintained on every write to the table, which is precisely why "many" doesn't mean "unlimited" in practice.

---

## 5. The Real Cost: Indexes Slow Down Writes

### Every `INSERT`, `UPDATE`, or `DELETE` must update EVERY affected index, not just the table itself

```plaintext
Inserting a new row into a table with FIVE indexes means: insert the row
  into the table's own storage, AND update FIVE separate B-tree
  structures to reflect this new row's presence — each one its OWN
  write, its OWN potential page split/rebalancing (per Section 2's
  balanced-tree structure), its OWN cost.
```

This is the genuinely important, often-underweighted half of the indexing trade-off — an index isn't a free performance win; it's a deliberate trade of write-side cost for read-side speed, and a table with many indexes can see meaningfully slower `INSERT`/`UPDATE`/`DELETE` performance precisely because every one of those operations now has more structures to keep synchronized and correct.

### Why this connects directly to this series' High-Volume Transaction Processing guide's own concerns

```plaintext
Per that guide's Section 6, hot rows and high write-volume tables are
  EXACTLY where this indexing trade-off matters most — a table
  sustaining thousands of writes per second pays the FULL index-maintenance
  cost on EVERY one of those writes, which is precisely why that guide's
  own emphasis on minimizing unnecessary work on the hot write path
  extends directly to being deliberate, not liberal, about which indexes
  a high-throughput table actually carries.
```

This is worth stating as a direct, genuine connection rather than a coincidental parallel — a write-heavy table in a high-volume transaction processing system is precisely the case where over-indexing (adding indexes "just in case," without confirming they're actually used by real queries) does the most real, measurable damage, since every unnecessary index is pure write-side overhead with no corresponding read-side benefit.

---

## 6. Composite Indexes and Column Order

### An index spanning multiple columns, where the ORDER of those columns is a genuine, consequential decision

```sql
CREATE INDEX IX_Orders_CustomerId_Status ON Orders(CustomerId, Status);
```

This index is genuinely, structurally different depending on which column comes first — its B-tree is sorted primarily by `CustomerId`, and *within* each `CustomerId`'s group of rows, secondarily by `Status`. This is precisely analogous to a phone book sorted by last name, then first name within each last name — you can efficiently find "everyone named Smith," and within that, "everyone named Smith whose first name is John," but you cannot efficiently find "everyone named John" without scanning the whole structure, since first name isn't the primary sort key.

### The leftmost-prefix rule: which queries a composite index can actually accelerate

```sql
-- ✅ Uses the index efficiently — CustomerId is the LEADING column
SELECT * FROM Orders WHERE CustomerId = 42;
SELECT * FROM Orders WHERE CustomerId = 42 AND Status = 'Shipped';

-- ❌ CANNOT use this index efficiently — Status alone is NOT the leading column
SELECT * FROM Orders WHERE Status = 'Shipped';
```

This is the single most important, most commonly misunderstood consequence of composite index column order — an index on `(CustomerId, Status)` genuinely cannot efficiently serve a query filtering only on `Status`, because the B-tree's sort order is governed by `CustomerId` first; searching for a specific `Status` value across all customers would require examining every `CustomerId` group, defeating the index's whole purpose. This is why column order in a composite index needs to be chosen deliberately, based on the *actual query patterns* the application genuinely uses, not just "the columns that seem related."

### Ordering by selectivity and query frequency, together

```plaintext
General guidance: put the column MOST OFTEN filtered on ALONE, or with
  the HIGHEST selectivity (Section 8), FIRST — but the real, correct
  answer always depends on the ACTUAL queries your application runs,
  not a universal rule applied blindly.
```

---

## 7. Covering Indexes and Index-Only Scans

### The extra cost Section 4 identified: a key lookup back to the full row

```plaintext
A non-clustered index on CustomerId, used to satisfy
  "SELECT OrderDate, Total FROM Orders WHERE CustomerId = 42," finds the
  matching rows' LOCATIONS efficiently via the index — but OrderDate and
  Total aren't PART of that index, so the database still needs a
  separate KEY LOOKUP back to the full row for every match, to retrieve
  those additional columns.
```

### A covering index: including the additional columns the query needs, directly in the index itself

```sql
CREATE INDEX IX_Orders_CustomerId_Covering
    ON Orders(CustomerId)
    INCLUDE (OrderDate, Total); -- these columns are STORED in the index's leaf nodes too,
                                   --  even though the index isn't SORTED by them
```

This is a genuinely powerful, deliberate optimization — by including `OrderDate` and `Total` directly in the index's leaf-level storage (via `INCLUDE`, distinct from being part of the sort key itself), the database can satisfy the entire query — find matching rows *and* retrieve every column it needs — from the index alone, with **no key lookup back to the table at all**. This is called an "index-only scan" or "covering index scan," and it's often meaningfully faster than an equivalent non-covering index, precisely because it eliminates the second, separate lookup step entirely.

### The trade-off: a covering index is LARGER, and costs more to maintain (Section 5), in exchange for eliminating key lookups

```plaintext
Worth including additional columns via INCLUDE deliberately, for
  queries that are GENUINELY frequent and performance-critical — not
  reflexively adding every column a table has to every index, which
  would defeat the entire size/maintenance trade-off this technique exists to manage.
```

---

## 8. Selectivity: The Property That Determines Whether an Index Helps at All

### Selectivity: what fraction of a table's rows a given value actually matches

```plaintext
HIGH selectivity: a column where most VALUES match very FEW rows —
  a genuinely unique or near-unique column (an email address, an order
  number) — an index here is HIGHLY effective, since it eliminates the
  vast majority of rows immediately.
LOW selectivity: a column where a given value matches a LARGE FRACTION
  of the table — a "IsActive" boolean column, say, where 95% of rows
  might share the SAME value — an index here is often GENUINELY USELESS,
  or even counterproductive.
```

### Why low selectivity can make an index actively worse than no index at all

```plaintext
If a WHERE clause matches 40% of a table's rows, the database's query
  optimizer will frequently (and correctly) CHOOSE to ignore an available
  index entirely and perform a plain table scan instead — because
  navigating the index's B-tree, THEN performing a KEY LOOKUP (Section 7)
  for EACH of those many matching rows, is genuinely MORE expensive than
  just reading the table sequentially once.
```

This is a genuinely important, counter-intuitive fact worth understanding precisely — having an index on a column doesn't guarantee the database will actually *use* it for a given query; the query optimizer makes a cost-based decision, and for a low-selectivity condition, a full table scan frequently *is* the objectively faster choice, which is exactly why Section 11 covers when adding an index in the first place is the wrong call.

---

## 9. Reading a Query Execution Plan

### The tool that tells you, definitively, whether an index is actually being used

```sql
-- SQL Server:
SET SHOWPLAN_ALL ON; -- or use the graphical execution plan in SSMS/Azure Data Studio
-- PostgreSQL:
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

This is worth treating as genuinely essential, not optional, for any serious indexing decision — rather than guessing whether an index helps, an execution plan shows precisely what the query optimizer actually decided to do: which indexes (if any) it used, whether it performed a table scan, and the estimated (or, with `ANALYZE`, actual) cost of each step.

### The key things to look for in a plan

```plaintext
"Index Seek" (SQL Server) / "Index Scan" with a condition (Postgres):
  the index WAS used, navigating directly to matching rows — GOOD, the
  expected, fast outcome for a well-indexed, selective query.
"Table Scan" / "Seq Scan": the ENTIRE table was read sequentially — the
  index (if one exists) was NOT used, either because none existed for
  this query, or because the optimizer determined (per Section 8) that
  using it would genuinely be slower.
"Key Lookup" / "Bookmark Lookup": an INDEX was used to find matching
  rows, but an ADDITIONAL step was needed to retrieve columns not
  covered by that index (Section 7) — worth knowing whether this is
  happening FREQUENTLY enough to justify a covering index instead.
```

Learning to recognize these specific terms in an execution plan is the actual, concrete skill underneath "know whether your indexing strategy is working" — everything this guide covers conceptually (selectivity, covering indexes, composite index column order) shows up directly, observably, in what a plan reports for a specific, real query.

---

## 10. Unique Indexes vs. Unique Constraints

### A unique constraint is enforced BY a unique index, underneath

```sql
ALTER TABLE Products ADD CONSTRAINT UQ_Products_Sku UNIQUE (Sku);
-- underneath, this CREATES a unique index on Sku — the CONSTRAINT and the INDEX are, mechanically,
-- close to the same thing in most database engines
```

Worth knowing this relationship precisely — a `UNIQUE` constraint isn't a separate, independent enforcement mechanism from indexing; the database implements uniqueness enforcement *by* building a unique index (one that simply rejects any insert/update that would produce a duplicate key) on the constrained column(s), which means a unique constraint gets the same B-tree-based lookup speed benefit as any other index, as a genuine side effect of the uniqueness guarantee.

### A unique index can exist without a formally-declared constraint, for the same effect

```sql
CREATE UNIQUE INDEX IX_Products_Sku ON Products(Sku); -- functionally equivalent to the constraint above,
                                                          -- for most PRACTICAL purposes
```

---

## 11. When NOT to Add an Index

### Low-selectivity columns (Section 8), unless part of a well-designed composite index

```plaintext
A standalone index on a boolean "IsDeleted" flag, where 99% of rows
  share the same value, is almost never useful on its own — the
  optimizer will likely ignore it (Section 8), and it still pays the
  full write-side maintenance cost (Section 5) for essentially no benefit.
```

### Tables that are written to far more often than they're read, especially with many existing indexes already

```plaintext
Per Section 5's write-cost discussion: a write-heavy, rarely-queried
  table (an audit log, an event stream) genuinely benefits LEAST from
  additional indexes and is HURT MOST by their maintenance cost — worth
  deliberately minimizing indexes on tables shaped this way.
```

### Columns that are genuinely never used in a `WHERE`, `JOIN`, or `ORDER BY` clause

```plaintext
An index only helps queries that actually FILTER, JOIN, or SORT on the
  indexed column(s) — indexing a column purely because it "seems
  important," without a concrete, real query pattern that would actually
  benefit, is pure overhead with no corresponding gain.
```

### The general discipline: index based on measured, real query patterns, not speculation

```plaintext
The correct workflow is nearly always: identify a SPECIFIC, genuinely
  slow query (via execution plans, Section 9, or query performance
  monitoring), determine whether an index would actually help it (given
  Section 8's selectivity and Section 6's column-order considerations),
  add it deliberately, and VERIFY via the execution plan that it's
  actually being used — not adding indexes speculatively and hoping
  they help.
```

---

## 12. Creating Indexes Through EF Core

### Fluent API index configuration, in `OnModelCreating`

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Order>()
        .HasIndex(o => o.CustomerId); // a simple, single-column index

    modelBuilder.Entity<Order>()
        .HasIndex(o => new { o.CustomerId, o.Status }) // a COMPOSITE index — column ORDER matches
                                                          //  the anonymous type's property order (Section 6)
        .HasDatabaseName("IX_Orders_CustomerId_Status"); // an explicit, meaningful name
}
```

This is the direct, concrete implementation of everything Sections 4-8 cover, expressed through EF Core's own configuration API rather than raw SQL — `HasIndex` with an anonymous type listing multiple properties produces exactly the composite index Section 6 describes, with column order in the anonymous type directly determining the index's actual sort order.

### A unique index, and a covering index, via the Fluent API

```csharp
modelBuilder.Entity<Product>()
    .HasIndex(p => p.Sku)
    .IsUnique(); // per Section 10 — enforces uniqueness AND provides the index performance benefit

modelBuilder.Entity<Order>()
    .HasIndex(o => o.CustomerId)
    .IncludeProperties(o => new { o.OrderDate, o.Total }); // per Section 7's covering index
```

### Migrations: how an EF Core index configuration actually reaches the database

```plaintext
Per this series' EF Core guide's Section 11: adding or changing a
  HasIndex configuration in OnModelCreating, then running
  "dotnet ef migrations add AddOrderIndexes," generates the CREATE INDEX
  (or DROP/re-CREATE) statements needed — the index becomes part of the
  same code-first, model-as-source-of-truth workflow that guide covers
  for the rest of the schema.
```

This is worth closing on as the direct link between this guide's SQL-level concepts and this series' EF Core guide's own workflow — indexes aren't a separate, out-of-band database concern when working with EF Core; they're configured in the same C# model, versioned through the same migrations, and deployed through the same controlled process that guide's Section 11 describes for schema evolution generally.

---

## 13. Common Pitfalls

| Pitfall | Why it hurts | Better approach |
|---|---|---|
| Assuming an index automatically speeds up every query on that column | Low-selectivity conditions are frequently ignored by the query optimizer in favor of a table scan, which can genuinely be faster | Check selectivity (Section 8) and verify actual usage via an execution plan (Section 9) before assuming an index helps |
| Adding many indexes to a write-heavy table "just in case" | Every index adds real, ongoing write-side maintenance cost, directly slowing down every `INSERT`/`UPDATE`/`DELETE` | Index deliberately, based on measured, real query patterns — especially minimize indexes on high-write tables (Section 5, Section 11) |
| Choosing composite index column order arbitrarily | A query filtering only on a non-leading column can't use the index efficiently at all, per the leftmost-prefix rule | Order composite index columns based on actual query patterns and selectivity, leading with what's most often filtered alone (Section 6) |
| Not knowing whether a query is actually using an available index | Guessing rather than verifying leads to wasted indexing effort or missed opportunities for genuine optimization | Use `EXPLAIN`/execution plans (Section 9) to confirm, rather than assume, what the query optimizer is actually doing |
| Confusing a unique constraint and a unique index as separate, independent mechanisms | Leads to redundant or conflicting index creation, or confusion about which one to add for a given uniqueness requirement | Understand a unique constraint is implemented BY a unique index underneath (Section 10) — they're mechanically nearly the same thing |
| Ignoring key lookups shown in an execution plan for a frequently-run, performance-critical query | A non-covering index still pays an extra retrieval step per matching row, which a covering index would eliminate | Add `INCLUDE`d columns to create a covering index for genuinely hot, frequent queries where key lookups show up repeatedly (Section 7) |
| Indexing a column that's never actually used in a `WHERE`/`JOIN`/`ORDER BY` | Pure write-side overhead with no corresponding read-side benefit, since nothing ever queries by it | Only index columns genuinely used in real query filtering, joining, or sorting patterns (Section 11) |
| Managing indexes outside of EF Core's model/migration workflow for an EF Core-managed database | Creates drift between the C# model (the intended source of truth) and the actual database schema | Configure indexes via the Fluent API and let migrations manage them, keeping the model and schema consistent (Section 12) |

---

## Quick Reference Table

| Concept | SQL / EF Core | Purpose |
|---|---|---|
| Clustered index | (default, usually the primary key) | The table's own physical row storage order — at most one per table |
| Non-clustered index | `CREATE INDEX ...` / `.HasIndex(...)` | A separate B-tree structure pointing back to full rows |
| Composite index | `CREATE INDEX ... ON t(a, b)` / `.HasIndex(x => new { x.A, x.B })` | Multi-column index; leading column governs which queries benefit |
| Covering index | `INCLUDE (...)` / `.IncludeProperties(...)` | Eliminates key lookups by storing extra columns directly in the index |
| Unique index | `CREATE UNIQUE INDEX` / `.IsUnique()` | Enforces uniqueness while providing standard index performance benefits |
| Execution plan | `EXPLAIN ANALYZE` / `SET SHOWPLAN_ALL ON` | Confirms whether and how an index is actually being used by a query |
| Selectivity | (a property of the data, not a syntax) | Determines whether an index is genuinely useful for a given column |

---

## Conclusion

An index's speedup comes from a real, well-understood data structure — a balanced B-tree — turning a linear, examine-every-row search into a logarithmic one, and that's a genuinely dramatic difference at real table sizes, not a marginal optimization. But that speedup is never free: every index is a deliberate trade of write-side maintenance cost for read-side speed, which is exactly why "add an index and see if it helps" is the wrong default mental model — the right one is understanding selectivity, composite column order, and covering-index trade-offs well enough to add indexes deliberately, for real, measured query patterns, and then confirming via an execution plan that the database is actually using them the way you intended.

This connects directly to this series' EF Core guide's migration workflow and this series' High-Volume Transaction Processing guide's own emphasis on minimizing unnecessary cost on a hot write path — indexing isn't a separate, purely-database-administrator concern once you're working in a code-first EF Core application; it's part of the same model, the same migrations, and the same deliberate, measured design discipline this whole series applies to every other performance-sensitive decision.

---

*Found this useful? Feel free to star the repo, open an issue with corrections, or share the composite-index-column-order-was-backwards-and-nothing-used-it incident that made the leftmost-prefix rule click far better than any B-tree diagram ever could.*
