title: Python 的時間還有時區處理
date: 2015-07-08 13:46:12
tags: python
---

身為一個程式設計師，很容易就會碰到時間處理的問題，如果你的服務是跨國的應用，也會需要瞭解怎麼去換算時區。
舉個例子來說，David 在美西時間的 20:00 (GMT-8) 發表了一則留言，而他在台灣的女朋友 Mary 看到的時間應該是 12:00 (GMT+8)。

## Python的時間處理 ##

python 的時間處理其實頗簡單

### 利用 timestamp 產生 python datetime 物件 ###

```python
import time
import datetime
# timestamp
t = time.time()
# datetime物件
dt = datetime.datetime.fromtimestamp(t)
```

### 利用字串產生 python datetime 物件 ###

```
dt = datetime.datetime.strptime(‘2015-07-08 11:20:28', '%Y-%m-%d %H:%M:%S')
```

### 利用 python datetime 物件產生字串 ###

```
datetime.datetime.fromtimestamp(t).strftime('%Y-%m-%d %H:%M:%S')
# Output -> '2015-07-08 11:20:28'
```

### 利用 python datetime 物件產生 timestamp ###

```
time.mktime(dt.timetuple()) + 1e-6 * dt.microsecond
```

## Python的時區處理 ##

### 更改時區 ###

利用pytz來更改時區，以下我們先拿來得出美西時間的2015-07-03 20:25

```
import pytz
us = pytz.timezone('US/Pacific')
dt = datetime.datetime.strptime('2015-07-03 20:25', '%Y-%m-%d %H:%M').replace(tzinfo=us)
# 物件為 datetime.datetime(2015, 7, 3, 20, 25, tzinfo=<DstTzInfo 'US/Pacific' PST-1 day, 16:00:00 STD>)
```

### 也可以利用asttimezone更改時區 ###

```
dt.astimezone(pytz.utc)
# 物件為 datetime.datetime(2015, 7, 4, 4, 25, tzinfo=<UTC>)
```

在處理時區問題時，我們會推薦要儲存 UTC 的時間(timestamp)，等到 User 存取資料時在去換時區， 而儲存時間也是一樣，要先把 User 傳上來的資料轉成 UTC 的時間，這時我們要注意在 python 中， datetime 物件的 timetuple 其實是沒有帶時區的資訊，`time.mktime`是利用你本地端電腦的時區 轉出timestamp，而`calendar.timegm`是不會自動幫你加時區的，這也是很多人沒搞懂的地方。

```
dt = datetime.datetime.strptime('2015-07-03 20:25', '%Y-%m-%d %H:%M')
time.mktime(dt.timetuple())
# 1435926300 => 2015-07-03 12:25 GMT
calendar.timegm(dt.timetuple())
# 1435955100 => 2015-07-03 20:25 GMT
```

下面這個例子，我們先更換時區，在產生出timestamp，可以拿來跟樓上的例子比較

```
dt = datetime.datetime.strptime('2015-07-03 20:25', '%Y-%m-%d %H:%M').replace(tzinfo=us)
time.mktime(dt.timetuple())
# 1435926300 => 2015-07-03 12:25 GMT
calendar.timegm(dt.timetuple())
# 1435955100 => 2015-07-03 20:25 GMT
```

tamp 還是跟先前一樣， 就是因為 timetuple 不會有 timezone 資訊，所以我們要利用`utctimetuple`， 讓他對應到正確的 UTC 時間的 timestamp。

```
dt = datetime.datetime.strptime('2015-07-03 20:25', '%Y-%m-%d %H:%M').replace(tzinfo=us)
time.mktime(dt.utctimetuple())
# 1435955100 => 2015-07-03 20:25 GMT
calendar.timegm(dt.utctimetuple())
# 1435983900 => 2015-07-04 04:25 GMT
```

從上面的過程中，我們最後可以得出，在轉換時區後，要取得正確的 timestamp， 必須要將`calendar.timegm`和`utctimetuple`一起服用，或是利用`dt.astimezone(pytz.utc)` 把時區先轉成UTC時間，才能得到我們真正想要的。
