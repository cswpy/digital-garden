---
title : Query Cost Estimation Example
notetype : feed
date : 2021-04-11
time: 17:45
mermaid: true
---
# Query Cost Estimation Example

Consider the following tables 

**Sailors**

|  sid  | sname  | rating |   age   |
|:-----:|:------:|:------:|:-------:|
| `INT` | `TEXT` | `INT`(0-9)  | `FLOAT` |

Sailor relation has 500 pages, each with 80 tuples. The `rating` column can only take an integer between 0 to 9.

**Reserves**

|  sid  |  bid  |  day   | rname  |
|:-----:|:-----:|:------:|:------:|
| `INT` | `INT`(1-100) | `DATE` | `TEXT` |

Reserves relation has 1000 pages, each with 100 tuples. The `bid` column has 100 unique IDs.

Additionally, each page is 4000 bytes large, and we have 5 pages in buffer pool.

We will estimate different physical and relational equivalencies of the following query

```sql
SELECT S.sname from Reserves R, Sailors S
WHERE R.sid = S.sid AND R.bid = 100 AND S.rating > 5;
```


## Join using Page Nested Loop Join
<div class='mermaid'>
flowchart BT;
S --500--> J["Join (sid)"];
R --1000--> J["Join (sid)"];
J --> Sr["Selection (rating ≥ 5)"];
Sr --> Sb["Selection (bid = 100)"];
Sb --> Ps["Projection (sname)"];
</div>
We start from the left child, that is a SeqScan on relation S. In PNLJ, for each page in S, we scan the inner relation R. So the cost is $500+500\times1000=500500$.

## Push Down Selection
``` mermaid
flowchart BT

Sr -- 250 --> J
R -- 1000 --> J["Join (sid)"]
S -- 500 --> Sr["Selection (rating ≥ 5)"]
J --> Sb["Selection (bid = 100)"]
Sb --> Ps["Projection (sname)"]

```
Here is when things get a bit confusing. Again, we start from the left child, calling a SeqScan on relation S and filter the results on-the-fly. This incurs a one-time cost of reading S (500 pages), page by page, into the buffer. Once we have a full page of S that satisfies the condition $rating\geq 5$ we bring it up to the join operator, which in turn calls a SeqScan on relation R, which incurs a cost of 1000. For selecting sailors with ratings larger or equal to 5, we have a *reduction factor* of 0.5, assuming a uniform distribution of ratings among the 10 possible values. This means that after the selection, we would have $500*0.5=250$ pages, each of which will call a SeqScan on relation R. This means the total I/O cost is $500+250*1000=250500$.

## Push Down All Selection
``` mermaid
flowchart BT

Sr -- 250 --> J
R -- 1000 --> Sb["Selection (bid = 100)"]
Sb -- 10 --> J["Join (sid)"]
S -- 500 --> Sr["Selection (rating ≥ 5)"]
J -->  Ps["Projection (sname)"]

```
Now we push down both selection to on-the-fly during the SeqScan of both relations. Firstly, we read in 500 pages from outer relation S one by one, and brings up a full page of selected tuples to the join operator sequentially. The join operator then calls the right child, doing a SeqScan on relation R, coupled with a on-the-fly filtering for `bid`. By the same reasoning, we could assume that the selection would produce 10 pages of desired tuples with a reduction factor of 0.1. For each call on the right child, we incur a read cost of 1000, even though the result is only 10 pages long. This gives us an estimate of $500+250*1000=250500$. Notice here that by pushing down another selection we did not reduce cost any further. This is because we never save the intermediate result in the right child, so we have to read R entirely every time the left child calls.