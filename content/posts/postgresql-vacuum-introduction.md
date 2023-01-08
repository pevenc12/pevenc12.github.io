---
title: "PostgreSQL：垃圾回收機制 VACUUM"
tldr: "垃圾回收操作不當，甚至會卡死整個 service"
description: "介紹 PostgreSQL 的垃圾回收機制以及衍生出的問題"
date: 2021-11-30T22:03:55+08:00
draft: false
tags: [vacuum, postgresql, XID]
---

前陣子在工作上遇到了資料庫佔用太多空間所以要清理的情境，便順手研究了一下 PostgreSQL 內的垃圾回收機制—— VACUUM 。


## 為什麼需要垃圾回收
在討論為什麼要進行垃圾回收之前，要先介紹一下 PostgreSQL 內部實作的 MVCC（Multi Version Concurrency Control）機制：

資料庫內部有一個 counter ，負責發給每個 transaction 一個 transaction ID（XID），每次發完之後會將 counter 加一。在 transaction 內完成 create 、 update 、 delete 等行為後，會將該 XID 更新到每一筆有動到的 tuple 裡，如此讓資料也有版號。當其他 transaction 要觸碰這些被更動過的 tuple 時，就會拿著自己的 XID 去跟 tuple 上的 XID 做比較，如果自己的版號較新，就能完整讀到所有資料，反之如果自己的版號較舊，就會讀不到。

由此可知，被刪除的 tuple 不會馬上從資料庫內消失，而是會被註記於某個 XID 以後就再也讀不到。因此，資料庫必須進行垃圾回收才能釋放空間並重複利用。

## VACUUM 機制

### 回收儲存空間
在 PostgreSQL 中，有兩種回收的行為： `VACUUM` 與 `VACUUM FULL` ，前者會將同一個 table 的空間回收並供該 table 繼續使用，並且在過程中並不會有使用到任何 lock 。後者則是會新建 table 並且將 tuples 重新寫入，過程中會使用 exclusive lock ，多餘的空間會釋出並還給作業系統。

### 更新 Query Planner 使用的數據
PostgreSQL 會依據各個表內的數據、資料型態（例如資料的離散程度、是否建立 Index 等等）產出最佳化的 Query Plan 。在 `VACUUM` 的過程中，會呼叫 `ANALYZE` 指令，更新這些數據，確保產出的 Query Plan 仍然適合目前的資料。想要瞭解更多有關 Query Plan ，可以參考我的前幾篇文章[1]。

### 更新 Visibility Map
Visibility Map 用來記錄一個 page 是否能被全部正在進行中的 transaction 給看到，即該 page 裡所有 tuples 的 XID 都足夠舊。如果是的話，則下一次 VACUMM 掃到這個 page 的時候就可以直接略過。

Visibility Map 的另一個用途是，當 transaction 進行 Index-Only Scan[2]時，可以用來確認搜尋結果中是否都包含正確的 XID ，意即是可以被該 transaction 所讀到的 index 。如此一來，就不需要循著 index 找出原 tuple 才能知道 XID 。

### 避免 XID Wraparound Problem
前文提到的 counter ，長度是 32 bit ，上限到 40 億左右，管控不好的話就會溢位然後從 0 開始，導致前後順序大亂，這就是所謂的 XID wraparound problem 。 PostgreSQL 如果發現快要達到上限的時候，會直接停下來不接受更多的寫入，嘗試多消化掉一些 transaction 。幾乎會造成服務中斷。

前陣子 Notion 進行了 DB Sharding[3] 的動機也是由於 VACUUM 跑得不順利，擔心會出現 XID Wraparound Problem 。知名的監控服務 Sentry 也曾經遇到這個問題[4]。想像中這類型的服務會有大量的寫入需求，如此一來就會就要更頻繁的。

## 結語
在 application 越長越大的過程，資料庫佔的空間也會快速成長，再加上 XID 衍生的問題，垃圾回收的機制變得越來越重要。提前發現未來的坑就能先避開。


## Reference
1. POSTGRESQL： 淺入淺出 EXPLAIN https://pevenc12.github.io/posts/postgresql-explain-explained/
2. Index-Only Scans and Covering Indexes https://www.postgresql.org/docs/current/indexes-index-only-scans.html
3. Herding elephants: Lessons learned from sharding Postgres at Notion https://www.notion.so/blog/sharding-postgres-at-notion
4. Transaction ID Wraparound in Postgres https://blog.sentry.io/2015/07/23/transaction-id-wraparound-in-postgres