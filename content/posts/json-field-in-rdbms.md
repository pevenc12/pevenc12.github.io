---
title: "JSON：試著舒服地在關聯式資料庫中使用"
tldr: "開發快樂與維護麻煩的大對決"
description: "說明、Benchmark、舉例如何好好使用 JSON"
date: 2023-05-21T22:45:12+08:00
draft: false
tags: [postgresql,database,python,json,django]
---

## 前言
在網路服務中，client-side 與 server-side 常利用 JSON 格式溝通，好用而且易懂。但是在資料庫中，特別是關聯式資料庫（以下稱 RDB），JSON 格式就相當微妙，好用易懂似乎並不是優點，反而會變成缺點。這就是工程上常見的 trade-off。

本文會先整理一下 JSON Field 於 server-side 開發時要注意的事項，接著介紹如何使用比較容易避開雷點。文中的例子都是使用 Python 程式語言與 PostgreSQL 資料庫。


## 資料類型
以 PostgreSQL 為例，在 RFC7159 規範下，裡面能放的資料格式包含：

| JSON primitive type | PostgreSQL type |
| ------------------- | --------------- |
| string              | text            |
| number              | numeric         |
| boolean             | boolean         |
| null                | (none)          |

其中 SQL NULL 和 JSON null 是不一樣的概念，無法劃上等號。在程式實作中，存入資料時要注意存入的資料會變成什麼形狀。例如使用 Python 語言與 DjangoJSONEncoder 時，會把 `datetime.date` 轉為 YYYY-MM-DD 格式的 string、把 `Decimal` 轉為 string 等等。如果今天有不同的系統都會接上同一個資料庫，要特別小心各個系統對於資料的轉換可能會不同。


## 比較
工程上，優缺點這件事都是相對於需求而言的，以下會根據 JSON 或 JSON Field 的特點進行討論。

### SCHEMA 彈性高
相較於 RDB 其他格式的欄位，JSON Field 明顯在 schema 上有很大的彈性，只要是 key-value pair 就可以塞進去 JSON Field。但這樣是好事還是壞事呢？

如果儲存的資料，是經常變動的、規格複雜、不確定的，那用 JSON Field 來存是適合的。舉例來說，有一個 Transaction 的 Table 要紀錄用戶的付款資料，因為系統支援各種不同的支付方式，所以有一個 payment_detail 來存不同支付方式產生的資料，例如系統有串接 stripe 且用戶付款選擇信用卡支付，可能記錄如下：

```json
{
    "gateway": "stripe",
    "payment_method": "credit_card",
    "transaction_uuid": "e148e5ce-8af6-41b6-a503-97be406de81d",
}
```

另外如果選擇 Line Pay，可能會記錄：

```json
{
    "gateway": "line_pay",
    "transactionId": "2016010112345678910",
    "orderId": "test_order_#1",
}
```

因為各家 payment gateway 回傳的資料沒有既定格式，甚至同一個 payment gateway 也有可能因為 API 異動而回傳不同其他資料。

Schema 高彈性當然也有顯而易見的壞處，那就是未來維護的人即使看過 Schema 也沒辦法一眼看出 Table 裡倒底存了哪些東西、用什麼格式存？如果大量使用，可以想見會留下不少債。另外，只要有 JSON Field 存在，就違反了 1NF。



### 效能
效能永遠是重點，直接測試。在這個 schema
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    color VARCHAR(30),
    properties JSONB
);
```

之下，簡單測試一下 query 的效能。塞入一萬筆的假資料，並且在 properties 裡面都放了 "color" 的 key-value pair。分別針對 color 欄位以期 properties 欄位進行 query，得到下列結果：

```
postgres=# EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE color = 'green';
                                                QUERY PLAN                                                
----------------------------------------------------------------------------------------------------------
 Seq Scan on users  (cost=0.00..2752.00 rows=6793 width=90) (actual time=0.048..15.136 rows=6744 loops=1)
   Filter: ((color)::text = 'green'::text)
   Rows Removed by Filter: 93256
   Buffers: shared hit=1502
 Planning Time: 0.078 ms
 Execution Time: 15.686 ms
(6 rows)

postgres=# EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE properties->>'color' = 'green';
                                               QUERY PLAN                                                
---------------------------------------------------------------------------------------------------------
 Seq Scan on users  (cost=0.00..3002.00 rows=500 width=90) (actual time=0.026..28.693 rows=6744 loops=1)
   Filter: ((properties ->> 'color'::text) = 'green'::text)
   Rows Removed by Filter: 93256
   Buffers: shared hit=1502
 Planning Time: 0.083 ms
 Execution Time: 29.179 ms
(6 rows)
```

在兩者都是 Seq Scan 的前提下，執行時間大約差了一倍。接著來替 color 與 properties.color 加上 index 後，再來嘗試一次 query。

```
postgres=# EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE color = 'green';
                                                       QUERY PLAN                                                        
-------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on users  (cost=76.94..1663.85 rows=6793 width=90) (actual time=0.672..3.106 rows=6744 loops=1)
   Recheck Cond: ((color)::text = 'green'::text)
   Heap Blocks: exact=1486
   Buffers: shared hit=1486 read=8
   ->  Bitmap Index Scan on idx_color  (cost=0.00..75.24 rows=6793 width=0) (actual time=0.424..0.425 rows=6744 loops=1)
         Index Cond: ((color)::text = 'green'::text)
         Buffers: shared read=8
 Planning:
   Buffers: shared hit=31 read=2 dirtied=3
 Planning Time: 0.365 ms
 Execution Time: 3.517 ms
(11 rows)

postgres=# EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE properties->>'color' = 'green';
                                                            QUERY PLAN                                                            
----------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on users  (cost=8.17..1043.85 rows=500 width=90) (actual time=1.188..4.778 rows=6744 loops=1)
   Recheck Cond: ((properties ->> 'color'::text) = 'green'::text)
   Heap Blocks: exact=1486
   Buffers: shared hit=1486 read=8
   ->  Bitmap Index Scan on idx_properties_color  (cost=0.00..8.04 rows=500 width=0) (actual time=0.871..0.872 rows=6744 loops=1)
         Index Cond: ((properties ->> 'color'::text) = 'green'::text)
         Buffers: shared read=8
 Planning:
   Buffers: shared hit=2
 Planning Time: 0.188 ms
 Execution Time: 5.482 ms
```

兩者讀取的效能都有顯著上升，但 JSON Field 明顯還是較慢。
總結來說，如果對於讀取效能有較高的要求，或是很常基於某個欄位進行 query，就可以考慮直接把欄位加進去 table 而不是只放在 JSON Field 裡。


## 格式化處理
站在 server-side 或是後端開發的角度來說，如果預期 JSON Field 格式是固定的或是有固定有幾種不同的 schema，總是期望從資料庫拉回來的值都可以自動完成格式化，不要每一次存取都要多一道手續處理格式化的工作。
在 Python 中可以利用 dataclass 來寫 getter/setter function 方便存取。以 Django 中的 model 為例，如果要記錄消費者付款資料，並且 payment gateway 來源有兩者：Stripe 與 Paypal，在 model 的部分可以改寫：

```python
from django.db import models
from django.contrib.postgres.fields import JSONField
from dataclasses import dataclass, asdict
from typing import Optional, Union

@dataclass
class StripeGateway:
    # 放 stripe 交易過程的資料
    stripe_version: str
    account_id: str # 只是舉例

    @classmethod
    def from_dict(cls, data: dict) -> 'StripeGateway':
        # 利用 dataclass 輕鬆初始化
        return cls(**data)

@dataclass
class PaypalGateway:
    # 放 paypal 交易過程的資料
    paypal_environment: str
    client_id: str # 只是舉例

    @classmethod
    def from_dict(cls, data: dict) -> 'PaypalGateway':
        # 利用 dataclass 輕鬆初始化
        return cls(**data)


class Payment(models.Model):
    amount = models.DecimalField(max_digits=5, decimal_places=2)
    _gateway_detail = JSONField(db_column='gateway_detail')

    @property
    def gateway_detail(self) -> Optional[Union[StripeGateway, PaypalGateway]]:
        if self._gateway_detail:
            gateway_type = self._gateway_detail.get('type')
            gateway_data = self._gateway_detail.get('data')

            if gateway_type == 'stripe': # 利用 type 來分類
                return StripeGateway.from_dict(gateway_data)
            elif gateway_type == 'paypal':
                return PaypalGateway.from_dict(gateway_data)
        return None

    @gateway_detail.setter
    def gateway_detail(self, gateway: Union[StripeGateway, PaypalGateway]) -> None:
        gateway_type = 'stripe' if isinstance(gateway, StripeGateway) else 'paypal'
        self._gateway_detail = {
            'type': gateway_type, # 利用 type 來分類
            'data': asdict(gateway) # data 才存真正的資料們
        }
```

在存入/取出時，操作如下：

```python
from myapp.models import Payment, StripeGateway, PaypalGateway

# Create a payment with Stripe
stripe_gateway = StripeGateway(stripe_version='2020-08-27', account_id='fake_account_id')
payment_with_stripe = Payment(amount=200.00, gateway_detail=stripe_gateway)
payment_with_stripe.save()

# Create a payment with Paypal
paypal_gateway = PaypalGateway(paypal_environment='sandbox', client_id='fake_client_id')
payment_with_paypal = Payment(amount=200.00, gateway_detail=paypal_gateway)
payment_with_paypal.save()

print(payment_with_stripe.gateway_detail.stripe_version)
# output: 2020-08-27

print(payment_with_paypal.gateway_detail.paypal_environment)
# output: sandbox
```

如此一來，寫 code 的過程就更輕鬆了一些。


## 結論
對開發者來說，RDB 開始完善支援 JSON Field 是一件舒服的事情。在開發與維護之間做取捨，是一個無止境的工程問題，本篇文記錄下自己的用法，給未來的自己一個參考依據。