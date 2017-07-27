---
layout: post
title: hahow 課程爬蟲與簡單定價分析
categories: 教學
tags:
  - python
  - data science
  - tutorial
---

[本文範例程式](https://gist.github.com/jwlin/d9574ef1f4191a7d823fb9467d599d90), [hahow_courses.json](https://gist.github.com/jwlin/5f00612ca0ecc4f2bc93d0b2cba441fd)

本文講解兩件事：

1. 如何爬取 [hahow](https://hahow.in) 所有已開課課程資料
2. 計算募資價/上線價/課程長度/學生數的統計資料 (平均值)，及前三者與學生數的相關性 (是否定價越低/長度越長則學生數越多?)

Pycone 松果城市 ([網站](http://pycone.com), [粉絲頁](https://www.facebook.com/pycone2016/)) 是致力於對初學者友善的 Python 教學團隊，除了已經開的[初心者課程](https://hahow.in/cr/python-for-beginners)與[爬蟲課程](https://hahow.in/cr/python-web-crawler)外，陸續還有新課程在籌備中。而一門課的定價該怎麼定，大家往往有不同的意見，有人認為技術有價，不能破壞行情，且越少見的課應該越貴；有人認為較低定價可以吸引更多學生。與其靠經驗或直覺，不如**讓數據說話**！我們何不直接分析 hahow 上程式類課程的平均價格及各項係數與學生數的相關性？

## 爬取 hahow 所有已開課課程資料

身為一個懶人，寫爬蟲前第一件事當然是看 hahow 有沒有提供打包下載的課程資料或可存取的公開 API，簡單搜尋之後沒有收穫，只好從 hahow 的[課程列表](https://hahow.in/courses)下手。在課程列表的網頁你會發現，這個網頁並不會一次回傳所有課程，而是隨著瀏覽器捲軸下拉，逐漸顯示更多課程。通常這種網頁多是透過 AJAX 與網站主機做非同步的溝通與資料傳輸，打開開發者工具瀏覽一下之後，很快找到了可能的 API

![2017-07-25-1](https://jwlin.github.io/images/170725-hahow-crawler-1.png)
---
![2017-07-25-2](https://jwlin.github.io/images/170725-hahow-crawler-2.png)

把該網址的 response 貼到 online JSON parser 驗證，果然就是課程資料

![2017-07-25-3](https://jwlin.github.io/images/170725-hahow-crawler-3.png)

接著繼續觀察畫面捲動時是透過什麼 API 取得更多課程資訊，最後確定了：

1. 一開始先透過 GET https://api.hahow.in/api/courses?limit=12&status=PUBLISHED 取得最初的 12 筆資料 (經測試一次最多可以取 30 筆)
2. 接著一樣透過 GET 回傳目前最後一筆課程的 id 與募資時間，取得接下來 12 筆課程資料，直到沒有資料為止 

不得不說 hahow 工程師的 API 寫得滿好的，很簡單易用 (雖然他們並沒有要給大家用 XD)，因此對應的爬蟲程式邏輯也很簡單

``` python
def crawl():
    # 初始 API: https://api.hahow.in/api/courses?limit=12&status=PUBLISHED
    # 接續 API: https://api.hahow.in/api/courses?latestId=54d5a117065a7e0e00725ac0&latestValue=2015-03-27T15:38:27.187Z&limit=30&status=PUBLISHED
    headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) '
                             'AppleWebKit/537.36 (KHTML, like Gecko) '
                             'Chrome/59.0.3071.115 Safari/537.36'}
    url = 'https://api.hahow.in/api/courses'
    courses = list()
    resp_courses = requests.get(url + '?limit=30&status=PUBLISHED', headers=headers).json()
    while resp_courses:  # 有回傳資料則繼續下一輪擷取
        time.sleep(3)  # 放慢爬蟲速度
        courses += resp_courses
        param = '?latestId={0}&latestValue={1}&limit=30&status=PUBLISHED'.format(
            courses[-1]['_id'], courses[-1]['incubateTime'])
        resp_courses = requests.get(url + param, headers=headers).json()
	# 將課程資料存下來後續分析使用
    with open('hahow_courses.json', 'w', encoding='utf-8') as f:
        json.dump(courses, f, indent=2, sort_keys=True, ensure_ascii=False)
    return courses
```

將所有課程擷取下來後，我們看到每一筆課程資料內容如下:

```
{
    "_id": "58744feda8aae907000d06c0",
    "categories": [
      "55de81ac9d1fa51000f94770",
      "55de81929d1fa51000f94769"
    ],
    "coverImage": {
      "_id": "588421e46ecf3a0700b7a31d",
      "url": "https://hahow.in/images/588421e46ecf3a0700b7a31d"
    },
    "incubateTime": "2017-02-09T05:45:14.673Z",
    "metaDescription": "你想自動擷取網站上的資料嗎？你學了 Python 卻不知道該從什麼程式開始練習嗎？這堂課就是為你準備的！本課程會循序漸進地說明如何撰寫 Python 網頁爬蟲，從環境設定開始，涵蓋網頁解構、資料擷取與儲存，及多項實戰演練，讓你在學習過程中及對於學習成果都有滿滿的成就感。",
    "numSoldTickets": 514,
    "owner": {
      "_id": "58744a86a8aae907000d0684",
      "name": "Jun-Wei Lin",
      "profileImageUrl": "https://hahow.in/images/58744c3ca8aae907000d0697",
      "username": "junwei"
    },
    "preOrderedPrice": 990,
    "price": 1890,
    "proposalDueTime": "2017-03-11T00:00:00.000Z",
    "reviewing": false,
    "status": "PUBLISHED",
    "successCriteria": {
      "numSoldTickets": 50
    },
    "title": "Python 網頁爬蟲入門實戰",
    "totalVideoLengthInSeconds": 15290,
    "type": "COURSE",
    "uniquename": "python-web-crawler"
}
```

各欄位的名稱都很直覺，唯一就是課程的分類 (categories) 代碼意義不明，此時只要觀察一下各類課程的連結就可以知道代碼

![2017-07-25-4](https://jwlin.github.io/images/170725-hahow-crawler-4.png)

## 計算各項係數之統計資料與相關性

收集資料是為了分析資料並進一步回答問題。我們的問題是：程式類課程的平均價格、課程長度及學生數為多少？各項係數是否與學生數有相關性？相較於[之前的文章](http://blog.castman.net/%E6%95%99%E5%AD%B8/2016/12/31/python-data-science-tutorial-5.html)中直接寫程式計算數據，這邊我們改用 numpy 來計算，只要將感興趣的資料分別存成 list，就能夠用 numpy 直接計算統計資料

``` python
with open('hahow_courses.json', 'r', encoding='utf-8') as f:
    courses = json.load(f)
	
# 取出程式類課程的募資價/上線價/學生數，並顯示統計資料
pre_order_prices = list()
prices = list()
tickets = list()
lengths = list()
for c in courses:
    if '55de81ac9d1fa51000f94770' in c['categories']:
        pre_order_prices.append(c['preOrderedPrice'])
        prices.append(c['price'])
        tickets.append(c['numSoldTickets'])
        lengths.append(c['totalVideoLengthInSeconds'])
		
print('程式類課程共有 %d 堂' % len(prices))  # 23
print('平均募資價:', np.mean(pre_order_prices))  # 719.09
print('平均上線價:', np.mean(prices))  # 1322.57
print('平均學生數:', np.mean(tickets))  # 483.22
print('平均課程分鐘:', np.mean(lengths)/60)  # 515.12

corrcoef = np.corrcoef([tickets, pre_order_prices, prices, lengths])
print('募資價與學生數之相關係數: ', corrcoef[0, 1])  # 0.18
print('上線價與學生數之相關係數: ', corrcoef[0, 2])  # 0.36
print('課程長度與學生數之相關係數: ', corrcoef[0, 3])  # 0.65
```

我們可以看到目前 23 堂程式類課程的平均募資價 (720) 與上線價 (1320)，而令人意外的是募資價與學生數並沒有太大的相關性 (不是募資價越低學生就越多)，反倒是課程長度與學生數呈現滿強的正相關。而我們在之前的文章已經提過，相關性並不代表因果關係，同時很明顯地，其他非數據的因素如講師名氣、文案內容、影片生動度等對於吸引學生也非常重要，因此這些數據只是幫助決策的參考資訊。
