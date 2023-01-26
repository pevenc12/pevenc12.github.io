---
title: "不要過度 Indexing，看看這個反例"
tldr: "不知道要不要加 Index，就先不要"
description: "舉例說明過度 Indexing 會有不良影響"
date: 2022-05-03T23:20:11+08:00
draft: false
tags: [postgresql,database,index,scan]
---

本文的舉例取自 [PostgreSQL Optimizer Methodology](https://youtu.be/XA3SBgcZwtE) 內容主要在 PostgreSQL 的 Query Planner 如何最佳化以及其遇到的痛點，值得一看。

文中提到 PostgreSQL 對於 row-count estimate 有改善的空間時，提到了以下範例：

```sql
SELECT * FROM foo WHERE a = 1 ORDER BY b LIMIT 1;
```

假設資料庫已經對 a 和 b 都分別做 indexing 的話，Query Planner 可能會產出以下兩種 Plan：
1. 利用 index scan 去找出 a = 1 的 tuples，再一一比較找出含最小值 b 的 tuple
2. 利用 index scan 依照 b 的大小順序一一把 tuples 拉出來看，再找含 a = 1 的 tuple

如果這個 table 只有幾個 tuples 的 a 值等於 1 的話，那就很適合使用第一個方法，只需要幾次的 random access 即可找到所求。但是，如果有大量的 tuples 的 a 值為 1 的話，會因為 random access 的次數過多而導致效能變差，反而方法二可能會好一些。

從這個例子可以看出 row-count estimate 是在做決策的時候非常重要的因素。事實上，即使在這個例子中，假設 Query Planner 對於 column a 和 column b 的數值分佈都掌握得非常精準，也有可能發生一種情形：所有 a = 1 的 tuples 都有非常大的 b 值，假設他們的 b 值剛好是最大的 10%。則在方法二中，必須用 index scan 掃過 90% 不符要求的資料，花費大量時間、資源在 random access 上，才能找到所求。

如果在這個 case 中，把 b 的 index 刪掉，讓 Query Planner 不產出方法二，反而是更好的做法。這證明了不要亂加 index，真的有需要再加就好。

加入不必的 index，除了會混淆 Query Planner，也會讓資料在寫入時必須要維護更多 B-Tree，進而影響效能。最好還是根據需求決定 index。

