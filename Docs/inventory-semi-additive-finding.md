# Finding: the inventory measures were summing across snapshots

`fact_inventory` holds 31 monthly snapshots — 18,600 rows of month × product × location.
It is **semi-additive**: it adds up across product and location, but adding it across time
counts the same physical stock 31 times.

Four measures were doing exactly that. Each was reproduced from source with a plain sum
over all snapshots, and each matched to the cent:

| Measure | Additive result | Correct, latest snapshot |
|---|---|---|
| Inventory Value | 369,250,030.97 | **12,338,823.59** |
| Units On Hand | 1,162,362 | **38,943** |
| Products Below Reorder Point | 905 | **22** |
| Products Below Safety Stock | 156 | **2** |

The exception counts are the giveaway. 905 products below reorder point against a
catalogue of 100 products is impossible — it exceeds the catalogue nine times over.

There is a scale check too. $369M of inventory against $56.8M of three-year revenue implies
6.5 years of stock on hand. The corrected $12.3M against a ~$23M annual run rate is about
six months, which is plausible for this business.

## The pattern

Every inventory measure now resolves to the latest snapshot **visible in the current filter
context**, so a date slicer still works:

```dax
Latest Snapshot = MAX ( fact_inventory[MonthStart] )

Inventory Value =
VAR LastSnap = [Latest Snapshot]
RETURN
    CALCULATE (
        SUM ( fact_inventory[InventoryValue] ),
        fact_inventory[MonthStart] = LastSnap
    )
```

The breach counts use the same collapse, then count distinct products:

```dax
Products Below Reorder Point =
VAR LastSnap = [Latest Snapshot]
VAR Breaches =
    FILTER (
        CALCULATETABLE ( fact_inventory, fact_inventory[MonthStart] = LastSnap ),
        fact_inventory[UnitsOnHand] < fact_inventory[ReorderPoint]
    )
RETURN
    COUNTROWS ( SUMMARIZE ( Breaches, fact_inventory[ProductKey] ) )
```

## It is demonstrable in the report

Page 3 carries a Year slicer. Set it to **2026** and the inventory measures resolve to the
July 2026 snapshot: $12.3M, 38,943 units, 22 products below reorder. Set it to **2025** and
they resolve to December 2025: $10.4M, 32,550 units, 38 below reorder.

The snapshot follows the filter rather than accumulating, which is the whole point.

## Why it matters beyond the number

Nothing about the additive version throws an error. It returns a plausible-looking figure
that happens to be thirty times too large. The only way to catch it is to know that a
snapshot fact is semi-additive and to sanity-check the result against something real — in
this case, the size of the product catalogue.
