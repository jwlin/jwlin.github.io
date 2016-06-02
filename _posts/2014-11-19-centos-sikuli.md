---
layout: post
title: 在 CentOS 6.5 下安裝 Sikuli
date: 2014-11-19
categories: Programming
tags:
  - Software Testing

---

為什麼要裝 Sikuli 呢？故事是這樣的，這是正常的網站畫面： 

![sikuli-1](https://raw.githubusercontent.com/jwlin/jwlin.github.io/master/images/2014-11-19-sikuli-1.jpg)

這是壞掉的：

![sikuli-2](https://raw.githubusercontent.com/jwlin/jwlin.github.io/master/images/2014-11-19-sikuli-2.jpg)

兩個網頁的原始碼一模一樣，只是壞掉的網頁有 javascript error，故無法用 XPath 之類的元素辨認方式設定 test oracle。那要怎麼設定 test oracle？方法有很多，首先根據 [Test Pyramid](http://martinfowler.com/bliki/TestPyramid.html) 實務，最正確的就是「javascript error 應該在 javascript unit test 層級就解決，不應該留給 GUI / Acceptance Test 處理」，打完收工。如果要在 System Test 層級做一些事情，以這個案例的作法有二：一是在 test case 執行過程中驗證是否有 javascript error 的產生，有則 test fail，這部分可參考[這篇](http://mguillem.wordpress.com/2011/10/11/webdriver-capture-js-errors-while-running-tests/)，我是用其中的第一個方式：deploy 到 dev server 之前先插一段 javascript 捕捉 js error，不是很理想 (如果有人知道文章中所寫的安裝 firefox toolbar 的方式請指教)；二是更直覺的做法：直接判斷畫面元件是不是 optically visible (因為 logical visible 上沒有差異)，所以 sikuli 就出馬了。

至於為什麼要自找麻煩在 CentOS 下安裝 Sikuli，而不是用官方測試過的 Windows、Ubuntu 或 Mac 呢？還不是因為這邊的 jenkins server 架在 CentOS 上面的關係 QQ，而且還沒有 GUI，要用 [headless](http://blog.castman.net/programming/2014/11/12/robotframework-jenkins-headless.html) 的方式跑 sikuli test cases。本文進度只到「在有預設 GUI 的 CentOS 6.5 上面把 sikuli 裝起來」，把 sikuli test case 改到 Xvfb 上面跑的時候就失敗了，在 DISPLAY :0.0 (實體顯示器) 上捕捉的圖案，在 Xvfb 上無法辨認出來，我想是螢幕圖案的辨識非常相依於顯示設備吧！如果有人知道如何在 Xvfb 上跑請指教。

在 CentOS 上安裝 Sikuli 幾乎都要手動，主要參考[這篇](http://www.sikulix.com/quickstart.html)跟[這篇](https://code.google.com/p/python-tesseract/wiki/HowToCompileForCentos)：

必要套件

```
yum -y groupinstall "Development Tools"
yum -y install wget cmake
yum -y install libjpeg-devel libpng-devel libtiff-devel zlib-devel
yum -y install gcc gcc-c++ make numpy
```

leptonica 1.71

```
wget http://www.leptonica.com/source/leptonica-1.71.tar.gz tar zxvf leptonica-1.71.tar.gz
cd leptonica-1.71 ./configure --prefix=/usr
make
make install
```

opencv 2.4.10

```
wget -O opencv-2.4.10.zip http://downloads.sourceforge.net/project/opencvlibrary/opencv-unix/2.4.10/opencv-2.4.10.zip?r=&ts=1415654591&use_mirror=superb-dca3
unzip opencv-2.4.10.zip cd opencv-2.4.10
mkdir release
cd release
cmake -D CMAKE_BUILD_TYPE=RELEASE -D CMAKE_INSTALL_PREFIX=/usr ..
make
make install
```

tesseract

```
svn checkout http://tesseract-ocr.googlecode.com/svn/trunk/ tesseract-ocr
cd tesseract-ocr/
./autogen.sh
./configure --prefix=/usr
make
make install
```

裝完以上之後記得 `ldconfig` 一下

終於輪到 sikuli 了

```
wget https://launchpad.net/sikuli/sikulix/1.0.1/+download/sikuli-setup.jar
java -jar sikuli-setup.jar
```

裝好之後 `java -jar sikuli-ide.jar` 或 `./runIDE` 即可啟動

![sikuli-3](https://raw.githubusercontent.com/jwlin/jwlin.github.io/master/images/2014-11-19-sikuli-3.png)

P.S. Windows 上安裝 sikuli 只需要第五步...  
