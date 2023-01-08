---
title: "Backend：處理時間、時區的幾個提醒"
tldr: "隨時搞清楚在資料庫、程式碼、 UI 上的時間是否包含時區"
description: "現代的軟體會有來自世界各地的使用者。在處理時間、時區的時候要格外小心，否則可能會造成使用者誤會、應用程式錯誤。"
date: 2021-08-15T18:10:14+08:00
draft: false
tags: [datetime, python, database, timezone]
---

## 前言
現代的軟體會有來自世界各地的使用者。在處理時間、時區的時候要格外小心，否則可能會造成使用者誤會、應用程式錯誤。本文列舉幾個在開發上需要注意的地方以避免這類的錯誤並提升程式的正確性與安全性。文中以 python 為程式語言做示範。

---

## 時區轉換
時間資料會同時出現在三個地方，包括：資料庫、資料系統（前端、後端、mobile）以及使用者的介面（web、app）。在使用者介面上大部分會將時間轉換至該使用者所在的時區，但是在資料庫與資料系統中會有一些不同的做法。

### 資料庫 vs 時區
在 PostgreSQL 與 MySQL 中存入包含時區的 timestamp 時（MySQL 在 8.0 以後的版本才支援），系統會先將該時間轉換回 UTC 時區再儲存。在讀取資料（SELECT）的時候另外在 SQL 中加入 `AT TIME ZONE` 指令即可將資料庫內的 timestamp 轉為指定的時區。使用 UTC 時區儲存有下列兩個好處：

#### 資料庫移動更彈性
在 MySQL 中的 timestamp 格式是不包含時區資訊的。假如資料庫採用當地時間進行存取，在之後資料庫要進行搬遷、複製（replication）到其他資料中心而且不在相同的時區時，資料會產生錯亂。反之，若統一使用 UTC 時區儲存資料的話，即使資料庫的預設時間更改，也不會影響到原本時間資料的正確性。

#### 解決變動時區的問題，例如夏令時間
在美國與部分歐洲國家中，會在春天時將時間調快一個小時，並在秋天時調回來。以2021年的美國夏令時間為例，會將3月14日凌晨兩點會往後調到凌晨三點，所以當天會有一個小時（02:00 - 03:00）完全消失。在11月7日會將凌晨兩點調回凌晨一點，所以會有一個小時重複出現（01:00 - 02:00），如果只以當地時間來儲存資料，可能會出現誤會。因此，一律儲存為 UTC 時區比較準確。

### 資料系統 vs 時區
在資料系統中進行計算、比較時間先後的時候，通常都統一使用 UTC 時區進行。但是如果資料經過計算後是會呈現在使用者面前，或是該服務會與使用者所在的時區有關，就必須要在計算前先將時間轉換到使用者所在的時區。例如在餐廳訂位系統中，消費者在查詢可訂位的時段時，系統依據餐廳營業的時間加上該餐廳所在的時區等資料計算出所有可訂位的時段，最後回傳給消費者選擇。

#### 確認套件的作用
各個程式語言對於時間、時區的處理都有很完整的支援，在使用前要注意對於時區的轉換。以 python 為例，處理時間的套件就有 `datetime`、`arrow` 等等。筆者習慣使用 `datetime` 處理時間的計算以及 `pytz` 處理時區。各套件對時區轉換的函式有不同的作用，有些會將時間一同轉換過去該時區，有些則只是在時間加上時區資訊，使用時要注意使用情境不要混用。舉例如下：
```python
import datetime
from pytz import timezone

# datetime.datetime.now(tz) 會得到 tz 時區的時間，含時區資料。如果 tz=None ，則得到 UTC 時區時間且時區資料為 None
dt = datetime.datetime.now()
print(dt)                      # 2021-08-15 08:41:06.827710
print(dt.tzinfo)               # None

# datetime.astimezone(tz) 會將 datetime 轉換時間到 tz 時區，並加上時區資訊
tz = timezone("Asia/Taipei")
dt_astimezone = dt.astimezone(tz=tz)
print(dt_astimezone)            # 2021-08-15 16:41:06.827710+08:00
print(dt_astimezone.tzinfo)     # Asia/Taipei

# pytz.timezone.localize(dt) 會將 timezone 的時區資訊加入 dt 中，但不會轉換時間
dt_localize = tz.localize(dt)
print(dt_localize)              # 2021-08-15 08:41:06.827710+08:00
print(dt_localize.tzinfo)       # Asia/Taipei
```

---

## 驗證
### 時間格式
在後端系統中對輸入進行驗證，除了安全上的問題（例如 SQL injection），也是為了確保系統可以正常地運作不會崩潰。以餐廳訂位系統為例，假設系統後端收到 `date` 與 `time` 兩個參數進行驗證，則：

```python
from datetime import datetime
from django.core.exceptions import ValidationError
# 直接使用 django 提供的 exception 套件，可自行設計

def validate_reservation_time(date, time):
    # date example -> "2021/09/01"
    # time example -> "18:00"
    try:
        # 以（年/月/日 時:分）的格式進行驗證，如果有錯就會 raise ValueError
        reservation_time = datetime.strptime(date + " " + time, "%Y/%m/%d %H:%M")
        datetime_now = datetime.now()
        if datetime_now > reservation_time:
            # 當預約時間已經過期
            raise ValidationError("reservation time outdated")
    except ValueError:
        raise ValidationError("invalid reservation time")
```

### 功能正確性
確保所寫的程式正常運作，最簡單的方法就是寫測試（單元測試）。在各語言中幾乎都有完整的套件可以模擬測試中的時間，並且含時區資訊，例如 python 中的 [freezegun](https://github.com/spulec/freezegun)。
```python
from datetime import datetime
from freezegun import freeze_time
from unittest import TestCase

@freeze_time("2021-08-15 21:00 +0800")
def test(TestCase):
    # +8時區21點 = UTC 時區13點
    assert datetime.utcnow() == datetime(year=2021, month=8, day=15, hour=13, minute=0, tzinfo=None)
```


建議在撰寫單元測試的時候，除了一般的 happy path 之外，也要根據實際使用情況測試各種可能出現的 edge cases。撰寫測試除了可以確保程式碼的品質，同時也是成本最低的除錯方式。


---

## 提升效能
與時間設定相關的服務，不一定需要實時運算。例如在各大外送服務的 app 中可以預約外送的時段，間隔為15分鐘。所以在這15分鐘以內，使用者點進來使用該服務都會得到相同的可預約時段。server 端不需要對於每一個 request 都重新計算一次可預約時段，可以將已經計算好的時段放在 redis 之類的 cache 裡面，並將 timeout 時間設定與間隔時間相同。

---

## 結語
近期在專案上正在開發與時間高度相關的功能，這篇文章主要是紀錄過程中的一些心得還有提醒未來的自己該注意的事項。


參考資料：
1. https://dev.mysql.com/doc/refman/8.0/en/time-zone-support.html
2. https://www.postgresql.org/docs/current/datatype-datetime.html
3. https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-ZONECONVERT
4. https://docs.djangoproject.com/en/3.2/topics/i18n/timezones/
