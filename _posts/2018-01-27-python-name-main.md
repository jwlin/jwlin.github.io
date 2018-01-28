---
layout: post
title: Python - if __name__ == '__main__' 涵義
categories: Programming
tags:
  - Programming
  - Python
---

相信許多人初學 Python 時，常會在範例程式內看到類似的段落：

```python
if __name__ == '__main__':
    main()  # 或是任何你想執行的函式
```

對於 `__name__ == '__main__'` 到底是什麼意思，一直存有疑問；在[我的爬蟲課程](https://hahow.in/cr/python-web-crawler)的問題討論區中，也不時看到同樣的問題。如果你上網搜尋之後還是似懂非懂，這篇文章會嘗試用初學者的角度再說明一次。

首先，如果你永遠都只執行一個 Python 檔，而不引用別的 Python 檔案的話，那你完全不需要了解這是什麼。例如你寫了一個很棒的函式 `cool.py`：

```python
# cool.py

def cool_func():
    print('cool_func(): Super Cool!')

print('Call it locally')
cool_func()
```

然後你永遠都是直接執行它，像這樣：

```bash
>> python cool.py

Call it locally
cool_func(): Super Cool!
```

這完全沒有問題。問題會出在當你想要在別的檔案中使用你在 `cool.py` 中定義的函式時，例如你在相同目錄下有一個 `other.py`：

```python
# other.py

from cool import cool_func

print('Call it remotely')
cool_func()
```

當你執行 `other.py` 時，你應該預期只會看到 `Call it remotely` 與 `cool_func(): Super Cool!` 兩段輸出，但實際上你看到的是：

```bash
>> python other.py

Call it locally
cool_func(): Super Cool!
Call it remotely
cool_func(): Super Cool!
```

看到問題了嗎？`cool.py` 中的主程式在被引用的時候也被執行了，原因在於：

1. 當 Python 檔案（模組、module）被引用的時候，檔案內的每一行都會被 Python 直譯器讀取並執行（所以 `cool.py`內的程式碼會被執行）
2. Python 直譯器執行程式碼時，有一些內建、隱含的變數，`__name__`就是其中之一，其意義是「模組名稱」。如果該檔案是被引用，其值會是模組名稱；但若該檔案是(透過命令列)直接執行，其值會是 `__main__`；。

所以如果我們在 `cool.py` 內加上一行顯示以上訊息：

```python
# cool.py

def cool_func():
    print('cool_func(): Super Cool!')

print('__name__:', __name__)
print('Call it locally')
cool_func()
```

再分別執行 `cool.py` 與 `other.py`，結果會是：

```bash
>> python cool.py

__name__: __main__
Call it locally
cool_func(): Super Cool!

>> python other.py

__name__: cool
Call it locally
cool_func(): Super Cool!
Call it remotely
cool_func(): Super Cool!
```

你就可以看到 `__name__` 的值在檔案被直接執行時與被引用時是不同的。所以回到上面的問題：要怎麼讓檔案在被引用時，不該執行的程式碼不被執行？當然就是靠 `__name__ == '__main__'`做判斷！

```python
# cool.py

def cool_func():
    print('cool_func(): Super Cool!')

if __name__ == '__main__':
    print('Call it locally')
    cool_func()
```

執行結果是:

```bash
>> python cool.py

Call it locally
cool_func(): Super Cool!

>> python other.py

Call it remotely
cool_func(): Super Cool!
```

問題就完美解決了！最後一點要說明的是，之所以常看見這樣的寫法，是因為該程式可能有「單獨執行」（例如執行一些本身的單元測試）與「被引用」兩種情況，因此用上述判斷就可以將這兩種情況區分出來。希望這篇文章有解答到你的疑問。

參考資料：
<https://docs.python.org/3/library/__main__.html>
<https://stackoverflow.com/questions/419163/what-does-if-name-main-do>