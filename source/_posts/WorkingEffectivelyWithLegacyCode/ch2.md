---
title: Working Effectively with Legacy Code - Ch2 - 帶著回饋工作
date: 2020-11-19 16:35:39
tags:
- SQL
- 讀書會報告
- Legacy Code
- Test
- Refactoring
categories: 
- 讀書會報告
cover: https://picsum.photos/id/169/800/600
---

# Working Effectively with Legacy Code - Ch2 - 帶著回饋工作

對系統進行變動，主要有兩種模式

1. 編輯並祈禱(edit and pray)
2. 覆蓋並修改(cover and modify)

### 編輯並祈禱(edit and pray)

> Software and cathedrals are much the same – first we build them, then we pray. 
> (Sam Redwine [Proceedings of the 4th International Software Process Workshop, Moretonhampstead, Devon, U.K., 11–13 May 1988, IEEE Computer Society])
>> 軟體和教堂非常相似——建成之後我們就開始祈禱。（Sam Redwine）

仔細的計畫要做的更動，並確保自己理解要修改的程式碼，這一系列的動作看似細心謹慎且需要高度專業，但卻對於不變的部分卻缺乏任何工程手段去掌握，無法做出任何保證。通常在完成變更後，我們還需要大量的時間去驗證我們真的把修改做對了。可惜這樣的方式幾乎是業界的常態了。

### 覆蓋並修改(cover and modify)

傳統的方式中，我們會在修改完成後去進行所謂的「回歸測試」(regression testing)，但這種測試通常都是應用程式介面上去執行的，但得到回饋的時間非常長，且粒度大，很難掌握確切出問題的關鍵，常常我們需要透過~~通靈~~推測來定位出問題的地方。

我們期望用測試去覆蓋軟體，並且在修改的當下能夠快速的驗證，馬上知道修改是否影響到其他我期望的不該改變的部分。所以我們得要依靠「單元測試」(unit test)

## 甚麼是單元測試?

什麼是單元(unit)呢？語意上來說這泛指事物中的基本單位，而對於軟體系統來說，可以稱之為基本單位的東西，大概不外乎就是「函數」(function)或者是「類別」(class)了。所我們期望可以透過程式去將這些基礎的單元做驗證。

這聽起來並不困難，但實際上真的是這樣嗎？測試的隔離性是單元測試中極為重要的一環，而我們回頭想想，我們實際上寫的類別有多少是完全沒有使用到其他類別或是其他環境的呢？應該是少之又少。

那麼我們不能連帶著這其他類別以及環境一起做測試嗎？其實像這樣依賴於別的類別或是環境的測試也相當重要，但這樣的測試存在一些問題

- **錯誤定位**：當我們的測試距離測試的對象越遠，中間干擾的因素就會增加，而當測試告知錯誤的時候，我們很難去定位錯誤發生的地方
- **執行時間**：這樣與其他部分耦合的測試常常會消耗比較多的時間，當量大的時候我們得到回饋的時間就會變得很漫長
- **覆蓋**：當我們新增新的程式碼的時候，我們往往必須耗費更大的心力去思考怎與其他部分耦合的測試要怎樣才能覆蓋我們新增的程式碼

好的單元測試應該具備下列特質：

1. 執行快
2. 能幫助我們定位問題所在

而通常具有以下特徵的就不算是單元測試

1. 與資料庫有互動
2. 進行了網路通訊
3. 接觸到檔案系統
4. 需要對環境做特定的準備才能執行

當然並非不屬於單元測試就是不好的測試，寫其他種類的測試往往也是有價值的，只是區分什麼測試是可以快速執行、立即回饋的還是相當重要的議題，所以我們必須將單元測試與其他測試的範疇跟劃分開來。

延伸閱讀——[91 30天快速上手TDD](https://www.dotblogs.com.tw/hatelove/2013/01/11/learning-tdd-in-30-days-catalog-and-reference)

## 高層測試

如上，不屬於單元測試的測試也相當重要，他們往往可以檢驗程式之間的互動是否正確，如「整合測試」、「API 測試」、「E2E 測試」...之類的。

延伸閱讀——[保哥 一次搞懂單元測試、整合測試、端對端測試之間的差異
](https://blog.miniasp.com/post/2019/02/18/Unit-testing-Integration-testing-e2e-testing)

## 測試覆蓋

讓我們來看一個範例，如下圖，我們想要修改 InvoiceUpdateResponder 的 getResponseText() 以及 Invoice 的 getValue()，對於這些變動的部分，我們想透過為他們撰寫單元測試來將他們覆蓋。

![](https://i.imgur.com/VT1vdQa.jpg)


接下來讓我們來觀察一下要為他們撰寫單元測試的時候可能會遇到的狀況，我們會透過測試框架去建立這些 class 的實例，並對裡面的 function 去測試

- Invoice 的 getValue()
    > 建立實例的過程應該會相當順暢，因為他有一個空白的建構子
- InvoiceUpdateResponder 的 getResponseText()
    > 首先我們會在建立實力的時候遇到一些問題
    > 1. 建構子需要一個 DBConnection ，而在單元測試中我們無法提供真正的資料庫去做連線，也不希望測試的範圍擴及到別的class
    > 2. 建構子需要一個 InvoiceUpdateServlet ，我們無法預估建立一個這樣的class需要多龐大的工作量，而且萬一 InvoiceUpdateServlet 的實例化又需要用到別的 class 呢?

以上這些問題都是出自於依賴關係而導致的問題，當我們的類別直接依賴於難以在測試中使用的東西的時候，就會變得難以去撰寫測試及修改程式，


**遺留程式碼的困境**
```
我們在修改程式碼的時候，應該要有測試作為保護。
而為了將這些測試安置妥當，我們往往又得先去修改程式碼。
```

那麼遇到這些問題的時候該怎麼辦呢?在這個例子中，書上介紹了**樸素化參數**(Primitivize Parameter，401頁)、**介面提取**(Extract Interface，377頁)幾種方法，在本書後面會有篇幅詳細說明如何去解決這樣的問題，這裡就先不提及，而修改完的結構大概會向下圖。

![](https://i.imgur.com/yGYvffN.jpg)

## 遺留程式碼修改演算法

遵照以下的步驟可以有效地對遺留的code進行修改

1. 確定變動（修改）點——參見本書第16、17章
2. 找出測試點——參見本書第11、12章
3. 解依賴——參見本書第7、9、10、22、23、25章
4. 編寫測試——參見本書第13章
5. 修改（變動）、重構——參見本書第8、20、21、22章