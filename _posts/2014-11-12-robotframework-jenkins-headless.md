---
layout: post
title: Robotframework, Selenium, Jenkins, Headless, Xvfb in CentOS
date: 2014-11-12
categories: Programming
tags:
  - Software Testing

---

工作上有使用 Robotframework 搭配 Selenium2Library 驅動 Firefox/Chrome 對網站做測試，並整合進 jenkins 的自動部署流程中。最近需要讓這些 test cases 能夠在沒有 X server (GUI) 的 CentOS 6.5 上面跑，同時也設定到該主機上的 jenkins 專案內。上網查了資料後很快找到解法，主要參考[這篇](http://laurent.bristiel.com/robot-framework-selenium-and-xvfb/)。

要讓瀏覽器能夠在沒有 GUI 的機器上跑，關鍵字就是 [Xvfb](http://en.wikipedia.org/wiki/Xvfb) -- X virtual frame buffer，其原理是在記憶體內創造出虛擬的顯示裝置，讓 client 送給 X server 的顯示訊息經由設定 DISPLAY 這個環境變數送給它，而不是真正的顯示器。以下紀錄在 CentOS 6.5 64 bit 下安裝的方式：

安裝 Xvfb 及 Firefox：

```
yum install xorg-x11-server-Xvfb.x86_64 firefox
```

裝好之後測試一下有沒有畫面：(指令參考[這篇](http://en.wikipedia.org/wiki/Xvfb)、[這篇](http://semicomplete.com/blog/geekery/xvfb-firefox.html)跟[這篇](http://www.x.org/releases/X11R7.6/doc/man/man1/Xvfb.1.xhtml)，或 man Xvfb)

```
Xvfb :1 -screen 0 1600x1200x16 &
export DISPLAY=:1 (數字自選)
DISPLAY=:1 firefox http://www.google.com
DISPLAY=:1 import -window root test.png
```

打開 test.png 來看看，有網站畫面就成功啦。在行文當下我安裝好 Firefox 之後，啟動時會有錯誤訊息，可能是 Firefox 的 [issue](https://support.mozilla.org/zh-TW/questions/1025819?esab=a&s=language&r=101&as=s)，我後來去下載 Firefox 的 [binary](https://www.mozilla.org/en-US/firefox/organizations/all/)，解壓縮覆蓋到原 firefox 目錄 (/usr/lib64/firefox) 就好了。

裝好 Xvfb 及 Firefox，指定 DISPLAY =:1 之後，就可以跑跑看 Robotframework 的 test cases，應該沒問題。至於如何整合進 jenkins？當然可以土法煉鋼先在 server 上開一個 Xvfb，並在 jenkins 的 robot command 前增加 export DISPLAY=:1，但 jenkins 其實有 [Xvfb Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Xvfb+Plugin)，裝好之後就可以自動在專案 build 之前開一個 Xvfb，build 之後自動關閉。

![jenkins-1](https://raw.githubusercontent.com/jwlin/jwlin.github.io/master/images/2014-11-12-jenkins-1.png)
![jenkins-2](https://raw.githubusercontent.com/jwlin/jwlin.github.io/master/images/2014-11-12-jenkins-2.png)