---
layout: post
title: 測試的可重複性及 flaky tests
categories: 研究
tags:
  - Software Testing
---

如果你曾在網路上看過[程式設計師最常回答的20句話](https://www.geeksaresexy.net/2014/04/14/top-20-replies-by-programmers-when-their-programs-dont-work-pict/)，你可能有印象其中好幾句都跟程式執行結果的不穩定性有關，例如"程式昨天還好好的"、"之前不會這樣"等等，而第一名：

> "It works on my machine."

除了讓人會心一笑之外，也點出了在軟體測試在實務上的問題：需要測試的環境組態太多，且輕微的組態差異就可能造成不同的程式行為或結果(註)。Google Testing Blog 最近的文章: [Flaky Tests at Google and How We Mitigate Them](http://googletesting.blogspot.tw/2016/05/flaky-tests-at-google-and-how-we.html) 就提到了 Google 內部的情況以及他們的處理方式。這篇文章將 flaky tests 定義為 "在一樣的待測程式、待測環境及測試程式碼下，測試結果有時候 pass, 有時候 fail 的那些 test cases"，中文姑且稱之為不穩定的測試個案。不穩定的測試個案會造成什麼影響？可能會大幅降低 programmer 的生產力：

1. 對於 falied test 要花時間去看錯在哪裡, 花時間 debug
2. 一時之間找不到 bug, 再花時間重跑 test 看看, 結果竟然過了?!
3. 若此情況發生多次, developer 對於 test 的效力就會失去信心, 以後看到 failed test 可能就直接略過 (忽略了真正的 bug), 或是多跑幾次看會不會運氣好 pass (又是時間的浪費)

因此, flaky tests 對於軟體品質跟工程師的生產力傷害甚鉅。你可能會好奇為什麼同樣的環境下測試為什麼有時候會過有時候不過？文章中提到了幾個 flaky test 的成因，例如：程式本身的行為就是不確定的 (Nondeterministic)(例如程式內會根據隨機產生的值有相對應的行為)、所用的 third party code 不穩、環境問題等等。[ICSE'15 的這篇論文](http://dl.acm.org/citation.cfm?id=2818764)也提到了影響 System GUI Testing 可重複性的幾個因素：

1. 執行環境，例如不同 OS，不同 browser，不同的 Java 版本，甚至只是 Oracle Java 與 OpenJDK 的不同也會有差
2. 程式啟始組態與輸入檔
3. 自動測試工具的設定 (以 GUI 測試來說最常見的就是每個動作的間隔時間)
4. 其他無法控制的因素，例如以程式啟動的時間日期做 random seed 等

論文中也提到：就算把前三個變因都控制到一模一樣，還是無法避免程式在 code coverage 或 GUI 畫面上會出現不同的測試結果。那 flaky tests 在 Google 內部出現的情況如何？文章中提到兩個統計數據：

1. 全部的測試中有 1.5% 是 flaky tests
2. 在程式提交後的自動測試 (post-submit testing), CI (Continuous Integration) 系統找出的由 passed 變成 failed 的 tests 中, 有 84% 含有 flaky tests

比例不算低，而 Google 用了一些方法去對應 flaky tests：

1. test 連續 fail 三次才標記為 fail (降低是 flaky test 的可能性, 但就多了執行時間)
2. 對於 test 可以標記其 flakiness (我猜是人工標註), 自動把高 flakiness 的 test 放到隔離區, 日後處理, 以免影響整體測試.
3. 目前正在試著從 code 或 execution traces 裡找出與 flakiness 有高度相關的 features

而上面提到的論文也建議了一些讓 System GUI Testing 可重複的一些建議步驟：

1. 確保執行環境是有紀錄的，且保持一致
2. 同一個 test case 跑多次再確定結果
3. 注意 application specific 的要求 (例如用到執行時間做 rand seed, file/network permissions, memory 要求等)

其實可以看到目前還沒有什麼好的解決方式。

註：專門探討如何系統化測試程式的各種參數或環境組合的研究叫做 Combinatorial Testing, [這篇論文](http://dl.acm.org/citation.cfm?id=1883618)有廣泛的回顧



