---
title: "PostgreSQL：淺入淺出 EXPLAIN（上）"
tldr: "知道資料庫如何執行 SQL 才能進行效能優化"
description: "簡單講解 EXPLAIN 指令以及其中最重要的 SCAN 行為。"
date: 2021-06-29 18:15:20 +0800
draft: false
tags: [postgresql,database,explain,index,scan]
---

### 前言
利用 EXPLAIN 指令對 SQL 進行效能分析相當的重要。除了確認資料庫內的索引（index）是否如預期被使用之外，再加上 ANALYZE ，更可以得到確切的執行流程與時間，由此來分析系統找出瓶頸。本文會先介紹 PostgreSQL 的三種掃描（SCAN）行為以及解說部分 Query Plan，下篇會再介紹各種聯合（JOIN）行為。


### Scan

#### Sequence Scan
會依序掃描所有硬碟中的 pages ，並找出符合查詢條件的資料（rows，行）。雖然聽起來很慢，但是在掃描的過程中會使用**順序 I/O** （Sequence I/O）而不是**隨機 I/O** （Random I/O），來提高速度。順序 I/O 的速度不管是在 HDD 上或是 SSD 上都明顯較快。


通常要避免資料庫進行全表查詢時，會加入索引來加快查詢的速度，此時就適用第二種查詢方式。


#### Index Scan
過程可以分為下列兩個步驟：

1. **到索引產生的 B-Tree 中去找出符合查詢條件的鍵值對（key-value pairs）**，如圖一。

> 圖一。這張表已經以 so 欄位建立索引。以 SELECT ... FROM ... WHERE so > 250; 這句 SQL 為例。圖中的 page number 與 offset （0~7）為了方便理解而做了簡化。
![b-tree-example-1.png](https://d.img.vision/pevenc12/b-tree-example-1.png)

+ 在 B-Tree 中，資料儲存的格式為鍵值對。其中鍵（key）為建立索引時所使用的欄位資料。鍵經過排序之後，儲存在 B-Tree 中，因此可以快速地被找到。而值（key）則為該筆完整資料所在的位置，類似一個指標，指向某個 page （包含 page number 以及 offset）。

2. 拿到鍵值對後，**透過 page number 與 offset 去找出該 page 與上面的完整資料**（rows）。

假如有多個查詢條件，並且都有建立 B-Tree 的話，**就會反覆進行步驟一與步驟二**。其中因為步驟二使用的是**隨機 I/O** ，如果執行太多次就會造成效能低落甚至不如使用 Sequence Scan 。

上述兩個方法利用的分別是利用了順序 I/O 以及索引帶來的優勢，如果嘗試將兩者結合，就會產生第三種查詢方式。


#### Bitmap Scan
以下的步驟是簡化版本，詳細的實作方式可以參考各資料庫的文件與程式碼：

1. 到索引產生的 B-Tree 中去找出符合查詢條件的鍵值對，如圖一。
2. **利用鍵值對中的的 page number 建立一個雜湊表**（Hash table, key = page number）。如圖二。在雜湊表中， page number 對應到的 bitmap ，其內容為：以該 page number 為基礎之下， offset 值為多少時，所找到的 page 符合查詢條件。

> 圖二。其中的 page number 以及 offset 值對應圖一的 B-Tree。
![hash-table-example-and-explain-2.png](https://d.img.vision/pevenc12/hash-table-example-and-explain-2.png)

3. 再利用步驟二產生的 bitmap 去找出所有符合條件的 page 。

因為雜湊表使用了 page number 作為 key ，**所以從硬碟讀取時可以依照 page number 的順序讀出**，**並且跳過沒有任何 offset 值的部分 pages** ，達到**順序 I/O** 。如此一來就能避免 Index Scan 所使用的隨機 I/O 。另外值得注意的是，在多個查詢條件之下（並且都已經建立索引）使用 Bitmap Scan ，會將產生的 bitmap 進行 OR 或是 AND 運算，找出同時符合（AND）或至少符合一個條件（OR）的 page 。

#### 小結

上述三種查詢方法各有優缺點與不同的適用情境。即使在已經建立索引的情況下，資料庫仍然有可能因為**總筆數太少**或是**相同索引值太多**，而選擇 Sequence Scan 而非其他兩者。
大致上來說，在利用索引查詢時，如果查詢出的總筆數較少，就可以使用 Index Scan ，而如果總筆數較多，就會使用 Bitmap Scan 來避免隨機 I/O 。而如果筆數非常多的時候，則直接使用 Sequence scan ，省去查詢 B-Tree 與建立 bitmap 的時間。這邊提到的多與少只是一個相對的概念，在不同的資料庫中會有不同的決策過程。


---


### EXPLAIN 範例
在 PostgreSQL 下了 EXPLAIN + SQL 指令之後，資料庫裡的 planner 會分析該 SQL 、資料庫內的資料、已建立的索引，產生 QUERY PLAN ：預計使用哪些方式來查詢資料。QUERY PLAN 是一個由 plan node 組合而成的樹狀結構。 plan node 可能會有多個子 plan node ，而 plan node 的行為包含了掃描（前文提到的三種 Scan）、運算（AND、OR）、過濾（Filter）、排序（Sort）或是迴圈（Nested Loop）。

1. 先來看最簡單的例子，從 tenk1 這個 table 取出所有資料。這個 QUERY PLAN 由一個 plan node 組成。
```sql
EXPLAIN SELECT * FROM tenk1;

                         QUERY PLAN
-------------------------------------------------------------
 Seq Scan on tenk1  (cost=0.00..458.00 rows=10000 width=244)
```
+ 因為 SQL 中並沒有出現 WHERE 子句，所以資料庫執行 Sequence Scan 搜尋整張表。其中的 cost 的單位是可自行定義的，預設值是將**讀取一個 disk page 所花費的時間訂為 1.0**。第五行得到的四個數字所代表的涵意依序為：
    + 0.00：**預估執行此 plan node 前需要的 total cost** 。因為這份 QUERY PLAN 只有一個 plan node ，所以起始的時間是 0.00。
    + 458.00：**預估完成此 plan node 的 total cost** 。
    + 10000：**預估**此 plan node 完成後會回傳的總行數（rows）。
    + 244：平均每一行資料（row）的大小，單位是 bytes 。


2. 在第二個例子中，增加一個 WHERE 條件在已經建立索引的欄位 unique1 上。因為查詢到的行數很少，所以使用 Index Scan 。
```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 = 42;

                                 QUERY PLAN
-----------------------------------------------------------------------------
 Index Scan using tenk1_unique1 on tenk1  (cost=0.29..8.30 rows=1 width=244)
   Index Cond: (unique1 = 42)
```
+ 利用索引找到符合 WHERE 子句的行數只有一行，只要在硬碟上搜尋 page 時執行一次**隨機 I/O** 即可找到該 page ，成本較低。


3. 承上例，在第三個例子中， WHERE 子句查詢到的行數較多，使用 Bitmap Scan。
```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100;

                                  QUERY PLAN
------------------------------------------------------------------------------
 Bitmap Heap Scan on tenk1  (cost=5.07..229.20 rows=101 width=244)
   Recheck Cond: (unique1 < 100)
   ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.04 rows=101 width=0)
         Index Cond: (unique1 < 100)
```
+ planner 使用 Bitmap Scan 。其中的兩個步驟： Bitmap Index Scan 和 Bitmap Heap Scan 分別對應到前文介紹的流程。 Bitmap Index Scan 利用索引搜尋後建立 bitmap ，而 Bitmap Heap Scan 利用 bitmap 去找出 page 。
+ 另外也能從 cost 的時間得知兩個 plan node 的發生順序：先執行 Bitmap Index Scan 再執行 Bitmap Heap Scan 。


4. 承上例，在第四個例子中， 再多新增一個 WHERE 條件在已建立索引的欄位 unique2 上。
```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100 AND unique2 > 9000;

                                     QUERY PLAN
-------------------------------------------------------------------------------------
 Bitmap Heap Scan on tenk1  (cost=25.08..60.21 rows=10 width=244)
   Recheck Cond: ((unique1 < 100) AND (unique2 > 9000))
   ->  BitmapAnd  (cost=25.08..25.08 rows=10 width=0)
         ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.04 rows=101 width=0)
               Index Cond: (unique1 < 100)
         ->  Bitmap Index Scan on tenk1_unique2  (cost=0.00..19.78 rows=999 width=0)
               Index Cond: (unique2 > 9000)
```
+ 首先分別對兩個索引進行 Bitmap Index Scan 得到兩組 bitmap ，再將兩組 bitmap 進行 AND 運算。得到最後的 bitmap 後再進行 Bitmap Heap Scan 找出 page 。兩組 Bitmap Index Scan 執行時對應到不同的 B-Tree ，因此可以同時進行，加速 QUERY PLAN 。


5. 在接下來的範例中，包含了跨表連接（JOIN），其中 unique1 與 unique2 皆已建立索引。
```sql
EXPLAIN SELECT *
FROM tenk1 t1, tenk2 t2
WHERE t1.unique1 < 10 AND t1.unique2 = t2.unique2;

                                      QUERY PLAN
--------------------------------------------------------------------------------------
 Nested Loop  (cost=4.65..118.62 rows=10 width=488)
   ->  Bitmap Heap Scan on tenk1 t1  (cost=4.36..39.47 rows=10 width=244)
         Recheck Cond: (unique1 < 10)
         ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..4.36 rows=10 width=0)
               Index Cond: (unique1 < 10)
   ->  Index Scan using tenk2_unique2 on tenk2 t2  (cost=0.29..7.91 rows=1 width=244)
         Index Cond: (unique2 = t1.unique2)
```
+ Nested Loop 為迴圈，會重複執行子 plan node ，次數要從子 plan node 上觀察。
+ 這個範例中， Nested Loop 包含兩個子 plan node ，第一個是在 tenk1 上進行 Bitmap Scan 。第二個 plan node 是在 tenk2 上進行 Index Scan ，類似執行下列的 SQL：
    ```sql
    SELECT ... WHERE t2.unique2 = constant
    ```
+ 整體的步驟如下：
    + 對 tenk1 進行 Bitmap Scan（t1.unique1 < 10）得到 rows_A（共 10 行）
        + 基於 rows_A 中的每一行 row_A ，對 tenk2 進行 Index Scan（t1.unique2 = t2.unique2），得到對應的 rows_B（可能多行）
        + 將 row_A 與所有 rows_B 連接後回傳

+ Bitmap Scan 中回傳的 rows 數量為 10， total cost 為 39.47 ，而 Index Scan 執行了十次， total cost 為 7.91 * 10。再加上 CPU 在連接（JOIN）上的耗時，就會接近 Nested Loop 裡的 total cost 。


6. 如果更改查詢條件，  planner 也會決定不一樣的連接方式。
```sql
EXPLAIN SELECT *
FROM tenk1 t1, tenk2 t2
WHERE t1.unique1 < 10 AND t2.unique2 < 10 AND t1.hundred < t2.hundred;

                                         QUERY PLAN
---------------------------------------------------------------------------------------------
 Nested Loop  (cost=4.65..49.46 rows=33 width=488)
   Join Filter: (t1.hundred < t2.hundred)
   ->  Bitmap Heap Scan on tenk1 t1  (cost=4.36..39.47 rows=10 width=244)
         Recheck Cond: (unique1 < 10)
         ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..4.36 rows=10 width=0)
               Index Cond: (unique1 < 10)
   ->  Materialize  (cost=0.29..8.51 rows=10 width=244)
         ->  Index Scan using tenk2_unique2 on tenk2 t2  (cost=0.29..8.46 rows=10 width=244)
               Index Cond: (unique2 < 10)
```
+ 在範例六中， Materialize node 會將完成 Index Scan 的資料暫存在記憶體中。接下來在 Nested Loop 中再次需要資料時會從記憶體中讀取而不是到硬碟中。
+ 同範例五，在最後的步驟要進行連接，但這次要再另外通過一個 Filter（t1.hundred < t2.hundred）。


7. 最後來看看加上 ANALYZE 之後的 QUERY PLAN。加上ANALYZE 後，資料庫會實際執行 SQL ，並且紀錄**實際**完成的時間以及**實際**回傳的 rows 數量。
```sql
EXPLAIN ANALYZE SELECT *
FROM tenk1 t1, tenk2 t2
WHERE t1.unique1 < 10 AND t1.unique2 = t2.unique2;

                                                           QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=4.65..118.62 rows=10 width=488) (actual time=0.128..0.377 rows=10 loops=1)
   ->  Bitmap Heap Scan on tenk1 t1  (cost=4.36..39.47 rows=10 width=244) (actual time=0.057..0.121 rows=10 loops=1)
         Recheck Cond: (unique1 < 10)
         ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..4.36 rows=10 width=0) (actual time=0.024..0.024 rows=10 loops=1)
               Index Cond: (unique1 < 10)
   ->  Index Scan using tenk2_unique2 on tenk2 t2  (cost=0.29..7.91 rows=1 width=244) (actual time=0.021..0.022 rows=1 loops=10)
         Index Cond: (unique2 = t1.unique2)
 Planning time: 0.181 ms
 Execution time: 0.501 ms
```
+ 此處 actual time 的單位是毫秒，與 cost 的單位不同。
+ 實際執行之後，可能會和原本 EXPLAIN 所得的結果不同。


---

### 結語
本篇文章介紹了 PostgreSQL 中的幾種掃描（Scan）行為，藉以看懂 EXPLAIN 生成的決策。資料庫可以說是後端開發的靈魂。後端開發常使用框架、套件中的 ORM 來協助開發。在開發過程中可以把生成的 SQL 語法拿出來檢查，除了可以提前檢查出問題之外，也可以拿到資料庫中直接使用 EXPLAIN 指令檢查行為是否和預期中一樣。


### 參考資料
1. https://condusiv.com/sequential-io-always-outperforms-random-io-on-hard-disk-drives-or-ssds/
2. https://www.postgresql.org/docs/13/using-explain.html
3. https://rajeevrastogi.blogspot.com/2018/02/bitmap-scan-in-postgresql.html?showComment=1518410565792#c4647352762092142586
4. https://www.cybertec-postgresql.com/en/postgresql-indexing-index-scan-vs-bitmap-scan-vs-sequential-scan-basics/