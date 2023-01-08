---
title: "PostgreSQL：淺入淺出 EXPLAIN（下）"
tldr: "JOIN 不一定會很慢，如果有好好處理的話"
description: "EXPLAIN 指令中另一個重要的 JOIN 行為。"
date: 2022-02-12T19:50:15+08:00
draft: false
tags: [postgresql,database,explain,join]
---

## 前言
在一個 query 中如果會關聯到兩個以上的 relation（表），就需要進行 JOIN。在 PostgreSQL 中有三種 JOIN 行為（大多數的關聯式資料庫也分為這三種）：Nested Loop Join、Merge Join、Hash Join。

## JOIN
先簡單加入一些資料當作起手式。假設在一個購物網站中，要關聯使用者資料與訂單資料。首先是使用者（user）資料：

| u_id     | name   | phone       |
| -------- | ------ | ----------- |
| 1        | Peven  | 0912345678  |
| 2        | Jack   | 0922345678  |
| 3        | Ben    | 0932345678  |

再來是訂單（order）資料：

| o_id     | amount   | payment_method | customer_id |
| -------- | -------- | -------------- | ----------- |
| 1        | 100      | CASH           | 1           |
| 2        | 200      | CREDIT_CARD    | 2           |
| 3        | 300      | CASH           | 3           |
| 4        | 150      | DEBIT_CARD     | 1           |
| 5        | 150      | CREDIT_CARD    | 3           |

並且下 query：

```sql
SELECT * FROM user JOIN order ON user.u_id = order.customer_id;
```
以下提到的 `left relation` 與 `right relation` 是 query planner 在 runtime 決定的，不一定是指 user 或是 order。

### Nested Loop Join
對於 left relation 中的每一個 row，都會取其 join attributes（在範例中就是 user.u_id 或是 order.customer_id）去掃描（SCAN）right relation，找到符合條件的 rows 再組合資料，如果對 SCAN 不熟悉的話可以參考[上一篇文章](https://pevenc12.github.io/posts/postgresql-explain-explained/)。這個方法看似很慢，但是如果在 SCAN 過程中有使用到 Index Scan 的話，整體效率會好上不少。
另外這個方法有幾個特色：
1. 如果有其中一個 relation 很小的時候，query planner 會傾向使用這個方法。如果 right relation 很小（筆數少）的話，對其進行大量掃描的成本就比較低。
2. 當 JOIN 條件中使用了非等號的判斷式，query planner 就只會選擇使用 Nested Loop Join。

### Merge Join
使用的前提是兩個 relation 必須先依照 join attributes 排好。例如範例中，user 已經依照 u_id 完成排序了，但是 order 需要依照 customer_id 重新排序如下：

| o_id     | amount   | payment_method | customer_id |
| -------- | -------- | -------------- | ----------- |
| 1        | 100      | CASH           | 1           |
| 4        | 150      | DEBIT_CARD     | 1           |
| 2        | 200      | CREDIT_CARD    | 2           |
| 3        | 300      | CASH           | 3           |
| 5        | 150      | CREDIT_CARD    | 3           |

如此一來，就可以平行地處理兩個 relation 將資料組合，而且兩個每一個 row 只會被掃過一次。
這個方法最大的問題就是對兩個 relation 都需要進行排序。排序資料是非常耗時的工作。

### Hash Join

#### 步驟一
對 right relation 進行掃描之後，將結果組成 hash table 放進記憶體中。其中 key 為 join attributes。如下圖：（假設 user 為 right relation）

| key(u_id) | value                                               |
| --------- | --------------------------------------------------- |
| 1         | {"u_id": 1, "name": "Peven", "phone": "0912345678"} |
| 2         | {"u_id": 2, "name": "Jack", "phone": "0922345678"}  |
| 3         | {"u_id": 3, "name": "Ben", "phone": "0932345678"}   |

#### 步驟二
接著掃描 left relation，將得到 rows 逐一透過 join attributes 去 hash table 找出所需的資料，例如：（假設 order 為 left relation）對於 o_id = 1 的訂單資料，其 customer_id = 1，所以就會去 hash table 找到 key = 1 的使用者資料。

這個方法會選擇其中一個 relation 放進記憶體，通常 query planner 會選擇將比較小的 relation 放進記憶體避免記憶體不足。可以透過下列兩個方法幫 relation 瘦身：
1. query 增加更多條件（WHERE）。如果在步驟一中對 right relation 的掃描過程增加更多條件，減少符合的 rows 數量，就能減少記憶體使用量。
2. query 減少欄位。欄位減少的話，資料量也會減少。例如：如果只需要 order 與對應到的 user 手機號碼的話可以將 query 改為：
    ```sql
    SELECT user.phone, order
    FROM user JOIN order ON user.u_id = order.customer_id;
    ```
    如此在過程中建出的 hash table 就會如下，資料量就會減少。
    | key(u_id) | value                              |
    | --------- | ---------------------------------- |
    | 1         | {"u_id": 1, "phone": "0912345678"} |
    | 2         | {"u_id": 2, "phone": "0922345678"} |
    | 3         | {"u_id": 3, "phone": "0932345678"} |
    

## 結論
在看 EXPLAIN 結果的時候，除了 SCAN 要特別注意之外，也可以觀察一下 JOIN 的行為是否符合預期。除了這兩者之外，其他的行為例如 SORT 等都是滿淺顯易懂的，就不再另外介紹了。

## 參考資料
1. https://use-the-index-luke.com/sql/join
2. https://www.postgresql.org/docs/current/planner-optimizer.html
3. https://stackoverflow.com/questions/49023821/nested-join-vs-merge-join-vs-hash-join-in-postgresql

