---
layout: post
title: 淺談 Python 的特殊方法 (Special Method Names) (1)
categories: 教學
tags:
  - python
  - tutorial
---

摘要：本文介紹以下幾個 Special Method Names:

* `__init__()`
* `__repr__()` (與 `__str__()`)
* `__add__()` 與 `__ge__()`
* `__getitem__()` (與 `__setitem__()`)
* `__iter__()` 與 `yield` 保留字

---

初學 Python 一陣子的朋友，想必都被 Python 語言的簡潔、優雅、與超快的開發速度所吸引，很開心地在寫生活或工作所需的大小程式了吧！本系列文章的目標，是透過範例介紹 Python 的特殊方法 (special method names) (就是那一大堆 `__xxx__()`、雙底線開頭與結尾的函式啦)，讓你的 Python 功力從初階銜接到中階。知道如何使用這些特殊方法，可以讓你的程式碼更簡潔、優雅、且容易維護，也可以漸漸看懂開源專案或是高手們的程式碼在寫什麼。

## 會自行換算的貨幣

讓我們從一個假想的貨幣問題開始吧！假設你有許多不同國家的紙鈔，幣值與匯率的換算總是有點麻煩，能不能讓不同國家的紙鈔「自己知道要怎麼換算」呢？我們可以從一個表示貨幣的類別 (class) 開始。貨幣至少要有國名及幣值：

```
# currency.py
class Currency:
    def __init__(self, symbol, amount):
        self.symbol = symbol
        self.amount = amount
```

`__init__()` 是撰寫 Python 物件導向程式時第一個碰到、也最常使用的特殊方法，其功用是作為建構式 (constructor)，在初始化 (initialize) class 的實例 (instance) 時被呼叫。首先，創建一個 Currency 實體並印出其資訊：

```
>>> from currency import Currency
>>> c1 = Currency('USD', 10)
>>> print(c1)
<currency.Currency object at 0x0000015C432AE0F0>
```

印出創建的 10 美元紙鈔時，顯示的是 Python 中表示物件的標準輸出，並不夠直覺，我們希望 `print(c1)` 時，能夠看到 `USD $10` 之類的資訊，該怎麼做呢？可以使用 `__repr__()` 方法：

```
class Currency:
    # ... 之前定義的 __init__() 方法
    def __repr__(self):
        return '{} ${:.2f}'.format(self.symbol, self.amount)

>>> c1 = Currency('USD', 10)
>>> c1
USD $10.00
>>> print(c1)
USD $10.00
```

`__repr__()` 方法可以覆寫「物件的標準輸出」，其回傳值必須是字串，我們可以在回傳時使用易讀的格式。或許有些朋友也看過 `__str__()` 方法，兩者的差異是：`__str__()` 會在使用 `str()`、`format()` 或 `print()` 時特別被呼叫，若該物件本身沒有定義 `__str__()`，會改呼叫 `__repr__()` 方法。因此若沒有特意要區別不同輸出格式，只要定義 `__repr__()` 即可。

接著，只要在類別定義內提供匯率資訊與換算方法，就能讓貨幣自行換算成他國貨幣：

```
class Currency:
    rates = {
        'USD': 1,
        'NTD': 30
    }
    # ... 之前定義的 __init__() 與 __repr__() 方法
    def convert(self, symbol):
        new_amount = (self.amount * self.rates[symbol]) / self.rates[self.symbol]
        return Currency(symbol, new_amount)       
    
>>> c1 = Currency('USD', 10)
>>> c1 = c1.convert('NTD')
>>> c1
NTD $300.00
```

一個 Currency 物件的 `convert()` 函式可以把自己轉成其他國家的貨幣。有了換算方法，不同貨幣就能加總與比較大小了，只要定義 `__add__()` 及 `__ge__()`即可：

```
class Currency:
    # ... 之前定義的 dict 與方法們
    def __add__(self, other):
        new_amount = self.amount + other.convert(self.symbol).amount
        return Currency(self.symbol, new_amount)
    def __gt__(self, other):  # "gt" 代表 "greater than"
       return self.amount > other.convert(self.symbol).amount

>>> c1 = Currency('USD', 10)
>>> c2 = Currency('NTD', 300)
>>> c1 + c2
USD $20.00
>>> c2 + c1
NTD $600.00
>>> c3 = Currency('NTD', 500)
>>> c3 > c1
True
>>> c1 > c3
False
```

利用以上特殊方法定義物件的加法與比大小的行為，就叫做運算子重載 (operator overloading)，聰明如你一定想到了：既然「加法」與「大於」可以定義，那減法、乘法、相等、小於...等運算子也可以定義囉？當然，完整的列表可以參考官方文件：[emulating numeric types](https://docs.python.org/3/reference/datamodel.html#emulating-numeric-types) 跟 [rich comparison methods](https://docs.python.org/3/reference/datamodel.html#object.__lt__)。

## 聰明的錢包

有了貨幣，再來創建一個聰明的錢包吧！錢包裡面當然要有貨幣，也要有能夠把錢放入錢包的方法：

```
class Wallet:
    def __init__(self):
        self.currencies = []
    def put(self, money):
        self.currencies.append(money)

wallet = Wallet()
wallet.put(Currency('USD', 10))
wallet.put(Currency('USD', 100))
wallet.put(Currency('NTD', 300))
```

如果想知道「錢包裡有多少美元」，當然可以這樣寫：

```
>>> for c in wallet.currencies:
...     if c.symbol == 'USD':
...         amount += c.amount
...
>>> print(amount)
110
```

但是，有沒有一個簡潔的作法，可以用讓我們用 `wallet['USD']` 這種方式取得錢包內所有美元 (或其他國家的貨幣)呢？有的！`__getitem__()` 就是用來定義 `[]` 運算子的行為：

```
class Wallet:
    # ... 之前定義的方法們
    def __getitem__(self, symbol):
        amount = 0
        for c in self.currencies:
            if c.symbol == symbol:
                amount += c.amount
        return amount

>>> wallet['USD']
110
>>> wallet['NTD']
300
```

與 `__getitem__()` 相對的地，`__setitem__()` 方法是定義「透過 `[]` 指定 value 給 key」的行為，如 `wallet['NTD'] = 300` 時會發生什麼事，不過此處 `__setitem__()` 並沒有適合這個錢包物件的語義，所以省略。

最後，讓我們把錢包裡的貨幣一一列出吧！最直覺的寫法是這樣：

```
>>> for c in wallet.currencies:
...     print(c)
...
USD $10.00
NTD $300.00
USD $100.00
```

但是我們也可以用 `__iter__()` 方法定義巡覽 (iterate) 物件時的行為：

```
class Wallet:
    # ... 之前定義的方法們
    def __iter__(self):
        for c in self.currencies:
            yield c

>>> for c in wallet:
...     print(c)
...
USD $10.00
NTD $300.00
USD $100.00
```

在 `__iter__()` 方法中用到了 `yield` 關鍵字，這大概是這篇文章最困難的部份。簡單來說，`yield` 跟 `return` 關鍵字很類似，函式遇到 `return` 跟 `yield` 時都會回傳一個值，但不同之處在於：函式遇到 `return` 時會結束，遇到 `yield` 時不會結束，而只會暫時把執行權還給呼叫者，下次被呼叫時，會接續之前的執行狀態，直到又遇到 `yield` 為止 (若沒遇到 `yield` 則結束函式)[註]。

本文透過範例介紹了 Python 的一些特殊方法。下次寫程式時可以試著使用它們，讓你的程式碼更優雅且容易維護。

## 補充資料

* [openhome.cc: 特殊方法名稱](https://openhome.cc/Gossip/Python/SpecialMethodNames.html)
* [Python 官方文件: Special method names](https://docs.python.org/3/reference/datamodel.html#special-method-names)


[註] 事實上，含有 `yield` 關鍵字的函式是一個 [generator](https://docs.python.org/3/library/stdtypes.html#generator-types)，它實作了 Python 的 [iterator protocol](https://docs.python.org/3/library/stdtypes.html#typeiter)，讓物件自動支援巡覽所需的 `__next__()` 方法。例如：

```
>>> w = wallet.__iter__()
>>> w.__next__()
USD $10.00
>>> w.__next__()
NTD $300.00
>>> w.__next__()
USD $100.00
>>> w.__next__()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```

不了解以上程式碼的意思沒關係，不影響本文範例的使用。關於 `yield` 的用法與原理我們將另文說明。