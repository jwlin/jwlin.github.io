---
layout: post
title: 將 Windows 下 os.path.join 的結果轉為 Unix style
date: 2015-10-04 14:40:45
categories: Programming
tags:
  - Python
---

(Convert os.path.join() to Unix style under Windows)​

`os.path.join()`是組合檔案路徑好用的函式，會自動依照當前的OS組合出適合的檔案路徑。例如在Windows下

```
>>> print os.path.join('path', 'to', 'directory') 
path\to\directory
```

如果要在Window下強迫把路徑轉為Unix style輸出，其實也不難，只要改用`posixpath.join()`就可以了，例如

```
>>> print posixpath.join('path', 'to', 'directory') 
path/to/directory
```

但我遇到的問題是：在Windows下承接`os.path.join()`的結果，並join其他路徑後，轉換為Unix style輸出到JSON。找了一陣子沒有看到現成的解法，因此我的作法如下：

1. 把 `os.path.join()`的內容解構為list (用`os.sep`取得split用字元)
2. 把解構後的list作為參數傳入posixpath.join() (用* operator (sequence unpacking))，其結果可以繼續posixpath.join()下去

範例如下

```
>>> win_path = os.path.join('path', 'to', 'data')
>>> print win_path
path\to\data
>>> posixpath.join(posixpath.join(*win_path.split(os.sep)), 'my_file.txt')
'path/to/data/my_file.txt'
```