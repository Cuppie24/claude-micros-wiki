# MySQL: nested per-group counts in a parent/child tree — carry the counts up in one recursive CTE, not "node × leaf" matching

Found 2026-10-24 in CWATIS (`global_db.StockGoodsGroups_Get`, goods tree `global_db.rfr_goods_tree` with `ParentId` + a materialized path `ParentIds`).

## Symptom

A procedure returns a tree of groups with "items in this group including all nested groups". It's correct and instant on test data, but slow on the
real catalog: 1.7 s on ~1,100 tree nodes / ~1,300 lowest groups / 16k items. No error and only one query, so nothing looks like an N+1. The cost is
hidden inside the SQL.

## Root cause

The first version matched every tree node against every lowest group by string path:

```sql
... FROM tree t
    JOIN cards c
    JOIN rfr_goods_tree leaf ON leaf.MGoodGroupId = c.GroupId AND FIND_IN_SET(t.MGoodGroupId, leaf.ParentIds)
GROUP BY t.MGoodGroupId
```

That is a nested loop of (tree nodes × lowest groups) string scans, quadratic in catalog size. `FIND_IN_SET` can't use an index. It's the SQL
equivalent of N+1: one query, but per-node work over the whole leaf set.

## Fix

Push each lowest group's count up its ancestor chain in the recursive CTE itself, then sum per group:

```sql
WITH RECURSIVE cards AS (SELECT g.ParentId AS GroupId, COUNT(*) AS Cnt FROM items ... GROUP BY g.ParentId),
up AS (SELECT grp.Id, grp.ParentId, c.Cnt, 1 AS lvl
       FROM tree_table grp JOIN cards c ON c.GroupId = grp.Id
       UNION ALL                                   -- ALL: contributions of different leaves to one ancestor must not collapse
       SELECT p.Id, p.ParentId, u.Cnt, u.lvl + 1
       FROM tree_table p JOIN up u ON p.Id = u.ParentId
       WHERE u.lvl < 50)                            -- guard: a cyclic ParentId would otherwise recurse until the depth limit error
SELECT Id, ParentId, SUM(Cnt) AS NestedCount FROM up GROUP BY Id, ParentId;
```

Rows = lowest groups × depth, joined on the primary key only. Same result, 1744 ms → 106 ms (measured on the full catalog as if every goods had an item).

## Checklist

- "Count including descendants" over a tree: carry the value up (UNION ALL + SUM), don't match each node against each leaf.
- `UNION` (distinct) in that CTE silently merges equal `(ancestor, count)` rows from different leaves → wrong (too small) totals. It must be `UNION ALL`.
- `FIND_IN_SET(x, path)` is fine for filtering one fixed id over a bounded set (e.g. "items under group X" over one warehouse's items). As a
  join condition between two growing sets it's a hidden quadratic.
- One procedure call per screen is not proof of "no N+1": look at what the SQL does per row.
