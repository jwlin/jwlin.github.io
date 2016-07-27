---
layout: post
title: 再探 Robot Framework 與 Test Case 撰寫的 Best Practice
categories: Programming
tags:
  - Software Testing
---

因為工作的關係又碰到了 [Robot Framework](http://robotframework.org/)，之前用它寫 test case 時比較隨性，這一次因為要將其導入並介紹給團隊，花了些時間看了官方文件跟一些 best practices，也有些實作經驗可分享。

## 什麼是 Robot Framework, 它能做什麼

Robot Framework 簡單來說就是一個**讓你撰寫 keyword driven script 的 framework**. 根據你想做的事情引入不同的 library 之後，就能使用各種 keyword 去兜出你想完成的工作。所以理論上它能夠做到 Python 可以做到的任何事情，反正如果沒有現成的 library 或 keyword 就自己寫即可。例如我們寫 Python 時經常用 [requests](http://docs.python-requests.org/en/master/) 與 server 溝通:

```
import requests

url = 'http://yourhost.com'
payload = {'uname': 'user', 'pwd': 'password'}
resp = requests.post(url, data=payload)
print resp.status_code  # 200
```

如果改用 RobotFramework 來寫的話，就引入 [RequestsLibrary](https://github.com/bulkan/robotframework-requests) 即可:

```
*** Settings ***
Library  RequestsLibrary

*** Test Cases ***
Post request
  Create Session  my_session  http://yourhost.com
  &{params}=  Create Dictionary  uname=user  pwd=password
  ${resp}=  Post Request  alias=my_session   uri=/  data=&{params}
  Log  ${resp.status_code}
```

我們可以看到 `Create Session` 與 `Post Request` 等 keywords 就是在做 requests 的 `requests.post()`，而事實上 RequestsLibrary 也的確是 requests 的 wrapper, 把 requests 能做到的事情打包成 keywords 讓你在 Robot Framework 裡面使用。再看一個 webdriver 的例子，用 Python 驅動 browser 去點擊或輸入網頁是這樣寫：

```
from selenium import webdriver

driver = webdriver.Chrome()
driver.get('http://yourhost.com')
element = driver.find_element_by_id('THE_ID')
element.send_keys('VALUE')
element = driver.find_element_by_xpath('THE_XPATH')
element.click()
```

在 Robot Framework 中則是引入 [Selenium2Library](http://robotframework.org/Selenium2Library/doc/Selenium2Library.html):

```
*** Settings ***
Library  Selenium2Library 

*** Test Cases ***
Open browser and run
  Open Browser  http://yourhost.com  chrome
  Wait Until Element Is Visible  id=THE_ID
  Input Text  id=THE_ID  VALUE
  Wait Until Element Is Visible   xpath=THE_XPATH
  Click Element  xpath=THE_XPATH
```

同樣地，`Open Browser`, `Wait Until Element Is Visible`, `Click Element` 等都是 Selenium2Library 提供的 Keywords。當然，你也可以用 Keywords 組成自己的 Keyword (`Wait And Click`, `Wait And Input`):

```
*** Settings ***
Library  Selenium2Library 

*** Test Cases ***
Open browser and run
  Open Browser  http://yourhost.com  chrome
  Wait And Input  id=THE_ID  VALUE
  Wait And Click  xpath=THE_XPATH

*** Keywords ***
Wait And Input
  [Arguments]  ${locator}  ${text}
  Wait Until Element Is Visible  ${locator}
  Input Text  ${locator}  ${text}
  
Wait And Click
  [Arguments]  ${locator}
  Wait Until Element Is Visible   ${locator}
  Click Element  ${locator}
```

如果沒有現成的 Keywords 可用，自己寫一個也很方便，例如這個做 Base64 編碼的 class:

```
import base64

class Base64Library:
    def base64_encode(self, text):
        return base64.b64encode(text)
```

只要路徑正確，就能在 test script 內引用自己定義的 keyword (`Base64 Encode`):

```
*** Settings ***
Library  Base64Library.py

*** Test Cases ***
Self-defined Keywords
  ${encoded}=  Base64 Encode  abc123  
  Log  ${encoded}  # YWJjMTIz
```

當然常見的變數宣告、參數、If-else、迴圈等，也都可以使用。

## 為什麼要用 Robot Framework

雖然 Robot Framework 可以做到幾乎任何 Python 能做的事情，但看到這邊你一定有個疑問：為何我還要為了這個 framework 去查各種 library 與 keyword 的用法，直接寫 Python code 不是快得多嗎？沒錯，如果只是要翻譯 Python code，根本不必用到這個 framework，這個 framework 是用在**自動化測試**，尤其是**自動化驗收測試 ([acceptance testing](https://en.wikipedia.org/wiki/Acceptance_testing))** 上的。使用 Robot Framework 做自動化測試的好處有:

* **Keyword Abstration.** 從前面的例子可以看到，雖然我們能夠直接寫 Python code, 但是 Robot Framework 有可能已經幫我們把好幾行程式碼濃縮成一個 keyword，例如使用 webdriver 時我們常常在等某個 DOM element 出現，在 python 裡面可能要寫個 try block 之類的去處理，但用 Robot Framework + Selenium2Library 一個關鍵字就搞定了 (`Wait Until Page Contains / Wait Until Element Is Visible`等)；另外 keyword driven script 對於程式經驗較少的人來說也會比 Python 或 Java 程式碼容易上手；最後，**自訂關鍵字的架構更可以把操作步驟一路抽象到自然語言層級，達到 "Test case 即註解" / "Test case 即文件" 的地步**。這部分後面會再說明。

* **Test Report and Jenkins Integration.** 它的報表與 Jenkins 整合功能大概是很多人選擇它的原因。Test script 執行之後會自動產生 HTML log，若 test case fail 會顯示停在哪一步，若是可以抓圖的話也會幫你把畫面抓下來，如下圖。當然它也支援一些測試相關功能，如 tag, test setup/teardown 等，Jenkins 也有 plugin 能顯示報表內容。

![2016-07-28-rbfk](https://raw.githubusercontent.com/jwlin/jwlin.github.io/master/images/2016-07-28-rbfk.png)

* **Human Readable Acceptance Test.** **把 test case 的步驟用自然語言描述，讓不寫程式的人 (工讀生/PM/老闆) 都能看懂**，是 Robot Framework 官方文件的建議寫法，但很多人往往只著重在把程式邏輯翻譯成 test script，這樣固然可以享受到 Robot Framework 的測試報表及 CI 整合等好處，但沒有利用到它全部的優點，比較可惜。以下分享一些 best practices.

## Robotframework 的 Do's and Dont's: 怎麼寫易維護的 Test Case 

剛開始寫 Robot Framework test script 的人，往往著重在把測試程式的邏輯翻譯成 keyword 的形式，這個起手式沒錯，例如上面的 webdriver 例子，直接等待並操作 DOM elements:

```
*** Settings ***
Library  Selenium2Library 

*** Test Cases ***
Open browser and run
  Open Browser  http://yourhost.com  chrome
  Wait Until Element Is Visible  id=THE_ID
  Input Text  id=THE_ID  VALUE
  Wait Until Element Is Visible   xpath=THE_XPATH
  Click Element  xpath=THE_XPATH  
  ...
```

用了一段時間之後，你可能會將測資設為變數，增加使用彈性，也可能把常用的實作細節包裝成自訂 keyword, 例如:

```
*** Settings ***
Library  Selenium2Library 

*** Variables ***
${HOST}  http://yourhost.com
${BROWSER}  chrome
${TEST_VALUE}

*** Test Cases ***
Open browser and run
  Open Browser  ${HOST}  ${BROWSER}  
  Wait And Input  id=THE_ID  {TEST_VALUE}
  Wait And Click  xpath=THE_XPATH
  ...

*** Keywords ***
Wait And Input
  ...

Wait And Click
  ...
```

做到這一步其實已經有一定的彈性，但還是有一些缺點，最明顯的就是**只看得出 test case 怎麼做 (How)，但看不出它在做什麼 (What)**。你要測試的 test requirement (system responsibility)，與實作細節 (按鈕的 Xpath 或 id) 顯然無關。另外，如果其他 test script 也要用相同的自定義 keywords 怎麼辦？或日後 test case 一多，該如何降低維護成本？要比較好地處理這些問題，我們要回到驗收測試的基本精神：驗收系統功能，**在不知道系統的介面及實作細節的前提下，利害關係人一樣知道系統該做什麼**。因此，**把系統功能步驟以自然語言的描述方式留在 test case 層級，實作細節留在 keyword / library，並把常用的實作細節抽出來方便重複引用**，就是官方推薦的使用方式。例如若要驗收從瀏覽器登入的功能，以下是一個可能的寫法：

```
*** Settings ***
Library  Selenium2Library

Resource  resource.txt  # 有 Wait And Click, Wait And Input 等
                        # 其他 test script 也會用到的 keywords
			
# 每個 test case 開始前與結束後需要的步驟可寫在 Setup 與 Teardown
# 或是整個 test suite 所需的步驟可以寫在 Suite Setup/Teardown

Test Setup  Proceed With Login Page
Test Teardown  Close All Browsers

*** Variables ***
${HOST}  http://yourhost.com
${BROWSER}  chrome
${USERNAME}  user
${PASSWORD}  pass
${EMPTY_USERNAME}  ${EMPTY}
${EMPTY_PASSWORD}  ${EMPTY}

*** Test Cases ***
Valid Login
  Login With Valid Credentials
  Dashboard Should Be Presented

Invalid Login
  Should Reject Invalid Credentials  ${USERNAME}  ${EMPTY_PASSWORD}
  Should Reject Invalid Credentials  ${EMPTY_USERNAME}  ${PASSWORD}

*** Keywords ***
Proceed With Login Page
  Open Browser  ${HOST}  ${BROWSER}
  Wait Until Page Contains  Welcome

Login With Valid Credentials
  Login With Credentials  ${USERNAME}  ${PASSWORD}
  
Login With Credentials
  [Arguments]  ${uname}  ${pwd}
  Wait And Input  id=username  ${uname}
  Wait And Input  id=password  ${pwd}
  Wait And Click  id=submit

Dashboard Should Be Presented
  Wait Until Page Contains  dashboard

Should Reject Invalid Credentials
  [Arguments]  ${uname}  ${pwd}
  Login With Credentials  ${uname}  ${pwd}
  Wait Until Page Contains  Invalid username or password
```

這樣寫有什麼好處呢？第一個是**把 test requirement 與實作細節分開**，如果只看 test case，任何人都能輕易了解第一個是在測合法登入，登入後要看到 dashboard；第二個是在測非法登入，並且分成空的密碼與空的使用者名稱兩種，只關心功能的人完全不需要知道怎麼實作。另外，盡量抽出重複的實作，如果實作細節改變了 (例如登入鈕的 id / xath 改了)，就能把修改成本降到最低。很多公司完全沒有導入自動化測試，因為**維護 test case 是需要成本的，而成本主要來自兩個地方：需求變動與實作細節變動**。把這兩個地方拆分有助於降低維護成本。你可以想像若直接把程式邏輯翻譯在 test case 層級，當需求變動時，我們必須花時間去看這一段邏輯是在做什麼事情；若實作細節改變了，就得大改一堆 test case。所有花在了解 test case 的時間，都在消耗工程師的生產力。

[這份文件](https://github.com/robotframework/HowToWriteGoodTestCases/blob/master/HowToWriteGoodTestCases.rst)與[這份投影片](http://www.slideshare.net/pekkaklarck/robot-framework-dos-and-donts)建議了 Robot Framework 的 Do's and Dont's，我摘錄如下：

* Test case 的命名: Tell what, not how
* 使用變數取代 hard coding
* 適當的抽象化
* 用等的 (Wait Until...) 而不是睡的 (Sleep)
* 讓 test case 自我描述，非必要不使用 document/comment
* 盡量不要讓 test cases 有相依性
* 盡量不要在 test case 層級有 variable assignment；複雜的邏輯如迴圈等盡量放在 library

[這份文件](http://cwd.dhemery.com/2009/11/wmaat/)在講如何寫易維護的 test case，原則如上所述，也可以看看。

## Robot Framework 與 Jenkins 的整合

與 Jenkins 的整合似乎就沒有什麼好講的了，就該裝的 plugin ([robotframework](https://wiki.jenkins-ci.org/display/JENKINS/Robot+Framework+Plugin), [virtualenv](https://wiki.jenkins-ci.org/display/JENKINS/ShiningPanda+Plugin)) 裝一裝，該設定的 dependency 設一設。我目前做的事情是把一些手動測項改寫成 script，導入 Robot Framework 由 Jenkins 驅動。每當待測程式有新版的 build, 就從 git server 上把 script 拉下來, 建一個 virtualenv, 把最新版 build deploy 到機器上做 regression testing. 瀏覽器的部分在寫 script 時是用 Chrome webdriver, Jenkins 上是用 PhantomJS 跑 headless.

雖然說是自動化測試，但目前在業界還是手工藝，只能把一些簡單或常規的測項寫好之後讓它重複跑，省去 regression testing 的時間，讓 test engineer 能夠把心力放在複雜的測試上。至於真正的全自動化測試，目前還是在學術研究的階段，必須搭配自然語言處理與機器學習的進展，也是我的研究興趣。



