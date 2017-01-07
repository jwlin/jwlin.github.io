---
layout: post
title: 給初學者的 Python 網頁爬蟲與資料分析 (4) 擷取資料及下載圖片
categories: 教學
tags:
  - python
  - data science
  - tutorial
---

本系列[完整範例爬蟲程式](https://gist.github.com/jwlin/b1cd5963be76ad5d7a13eed4ad7d8675)

這篇文章會說明如何將各文章內的圖片下載到本機端，並計算、儲存圖片數。經過之前的步驟，我們已經有了文章列表，其格式是：

``` python
articles = [
    {'push_count': 8, 'title': '[正妹] 韓國瑜的女兒', 'href': '/bbs/Beauty/M.1482411674.A.855.html'}
    {'push_count': 5, 'title': '[正妹] 佐々木優佳里 Happiness', 'href': '/bbs/Beauty/M.1482414319.A.C09.html'}
    {'push_count': 13, 'title': '[正妹] 甜美笑容 第一二五彈', 'href': '/bbs/Beauty/M.1482416491.A.656.html'}
    ...
]
```
因此，我們的步驟是：

1. 連線到網站，取得該文章網頁 (`get_web_page()`)
2. 找到文章內的圖片網址們 (`parse()`)
3. 在本機新增以文章標題為名的資料夾，將圖片存到本機 (`save()`)
4. 紀錄圖片數量；繼續巡訪下一篇文章直到沒有文章為止

程式碼如下，非常簡單：

``` python
PTT_URL = 'https://www.ptt.cc'

for article in articles:
    page = get_web_page(PTT_URL + article['href'])
    if page:
        img_urls = parse(page)
        save(img_urls, article['title'])
        article['num_image'] = len(img_urls)        
```

以下分別說明各步驟。

## 取得文章網頁

這部分之前已經說明過，只是文章的網址換一下而已。要注意的是 PTT 網頁內文章的 href 屬性是相對路徑，因此連線時要加上完整網址名稱 (PTT_URL)

## 找到文章內的圖片網址們

這部分一樣是用 BeautifulSoup 的 `find_all()` 來完成。我們先假設圖片網址一定是 "http://i.imgur.com" 開頭，用 Chrome 開發者工具檢視網頁區塊後，我們知道我們要找的是 \<div id="main-content"\> 區塊內所有的 \<a\> 標籤，且 href 屬性是 "http://i.imgur.com" 開頭的連結：

``` python
def parse(dom):
    soup = BeautifulSoup(dom, 'html.parser')
    links = soup.find(id='main-content').find_all('a')
    img_urls = []
    for link in links:
        if link['href'].startswith('http://i.imgur.com'):
            img_urls.append(link['href'])
    return img_urls
```

看到這邊你一定有疑問：imgur 網站圖片的網址不一定是 http 開頭，也可能是 https 開頭；網址也可能是 m.imgur.com 或 imgur.com，例如以下網址都是同一張圖片的合法網址：

``` python
test_urls = [
    'http://i.imgur.com/A2wmlqW.jpg',
    'http://i.imgur.com/A2wmlqW',  # 沒有 .jpg
    'https://i.imgur.com/A2wmlqW.jpg',
    'http://imgur.com/A2wmlqW.jpg',
    'https://imgur.com/A2wmlqW.jpg',
    'https://imgur.com/A2wmlqW',
    'http://m.imgur.com/A2wmlqW.jpg',
    'https://m.imgur.com/A2wmlqW.jpg'
]
```

但我們的程式只能辨認出前兩種網址。當然你可以增加字串值與條件判斷去辨認更多種格式的網址，但較簡潔的方法是透過正規表示式 (Regular Expression) 指定字串的格式。例如能辨識出以上全部格式的正規表示式為:

```
'^https?://(i.)?(m.)?imgur.com'
```

"^" 表示字串開頭，字元緊接著 "?" 表示該字元可出現 0 或 1 次，所以 "^https?" 表示的是 "http" (s 出現 0 次) 或 "https" (s 出現 1 次) 開頭的字串，同理 (i.)? 表示 "i." 可以出現 0 或 1 次。我們用 `re.match()` 判斷字串是否符合所定義的正規表示式：

``` python
import re
for url in test_urls:
    print(re.match('^https?://(i.)?(m.)?imgur.com', url))  # 符合則回傳 SRE_Match Object, 不符合則回傳 None
# <_sre.SRE_Match object; span=(0, 18), match='http://i.imgur.com'>
# <_sre.SRE_Match object; span=(0, 18), match='http://i.imgur.com'>
# <_sre.SRE_Match object; span=(0, 19), match='https://i.imgur.com'>
# <_sre.SRE_Match object; span=(0, 16), match='http://imgur.com'>
# <_sre.SRE_Match object; span=(0, 17), match='https://imgur.com'>
# <_sre.SRE_Match object; span=(0, 17), match='https://imgur.com'>
# <_sre.SRE_Match object; span=(0, 18), match='http://m.imgur.com'>
# <_sre.SRE_Match object; span=(0, 19), match='https://m.imgur.com'>
```

因此，我們將 `parse()` 改寫為：

``` python
def parse(dom):
    soup = BeautifulSoup(dom, 'html.parser')
    links = soup.find(id='main-content').find_all('a')
    img_urls = []
    for link in links:
        if re.match(r'^https?://(i.)?(m.)?imgur.com', link['href']):
            img_urls.append(link['href'])
    return img_urls
```

## 將圖片存到本機端

有了圖片網址，我們會創造一個以文章標題為名的資料夾，並將圖片下載到該資料夾內。在此要注意的有 3 點：

1. 我們擷取了 imgur.com 網址的各種形式，但下載圖片時用的網址必須是 i.imgur.com 開頭，因此要把 m.imgur.com 換成 i.imgur.com，或把 imgur.com 補成 i.imgur.com
2. 網址結尾不一定有 .jpg，為了順利下載，記得補上 .jpg。這些字串處理過程，就是資料淨化與清理的工作。
3. 因為文章標題可能會有作業系統不支援的字元，所以 `os.makedirs()` 可能會失敗，失敗時就無法創造資料夾，並印出 exception。要處理這個問題，一個解法是用正規表示式過濾系統不支援的字元，在此先略過。

``` python
def save(img_urls, title):
    if img_urls:
        try:
            dname = title.strip()  # 用 strip() 去除字串前後的空白
            os.makedirs(dname)
            for img_url in img_urls:
                if img_url.split('//')[1].startswith('m.'):
                    img_url = img_url.replace('//m.', '//i.')
                if not img_url.split('//')[1].startswith('i.'):
                    img_url = img_url.split('//')[0] + '//i.' + img_url.split('//')[1]
                if not img_url.endswith('.jpg'):
                    img_url += '.jpg'
                fname = img_url.split('/')[-1]
                urllib.request.urlretrieve(img_url, os.path.join(dname, fname))
        except Exception as e:
            print(e)
```

到這邊為止，你的程式已經可以下載 PTT Beauty 板今日文章的圖片，並且有了今天每一篇文章的標題、推文數、圖片數、文章連結等資訊，你可以把資訊存成 json 檔案如下:

``` python
import json

with open('data.json', 'w', encoding='utf-8') as f:
    json.dump(articles, f, indent=2, sort_keys=True, ensure_ascii=False)
```

這個簡單的範例還有許多可改進的地方，例如：處理文章標題的特殊字元，只有推文數多的文章才下載圖片、略過推文的圖片、支援更多圖床網址、多執行緒下載圖片等，但已經展示了一些基礎爬蟲技巧與概念。下一篇文章會說明如何做簡單的資料分析 (統計資料與畫圖)。
