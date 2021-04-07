---
title: 單元測試的藝術 - Ch1 單元測試基礎 - Sec. 1.5 ~ 1.8
date: 2021-02-24 13:42:52
tags:
- Unit Test
- Test
- 讀書會報告
categories:
- 讀書會報告
cover: https://picsum.photos/id/153/800/600
---

# 單元測試的藝術 - Ch1 單元測試基礎 - Sec. 1.5 ~ 1.8

## 1.5 一個簡單的單元測試範例

其實就算沒有測試框架的輔助，我們也可以自己寫一個自動化的單元測試

舉例來說，有一個類別 ```SimpleParser``` ，裡面有一個 ```ParseAndSum``` 方法

### ParseAndSum()

#### Input

輸入零個或多個逗號分開的數字所組成的字串

exaple: 1,2,3

#### Output

- 輸入不包含數字
    - 則回傳 0
- 包含一個數字
    - 則回傳該數字
- 包含多個數字
    - 則回傳這些數字的相加

為了示範方便，該 fucntion 現在只能處理0個或是1個數字的情況

---

首先，我們有了一個已經實作了上述需求的 ```SimpleParser``` 類別

```SimpleParser```
```php=
<?php

namespace Src;

use Exception;

class SimpleParser
{
    public function ParseAndSum(string $number): int
    {
        if (strlen($number) == 0) {
            return 0;
        }
        if (strpos($number, ',') !== false) {
            return (int)$number;
        } else {
            throw new Exception('I can only handle 0 and 1 numbers for now!');
        }
    }
}
```

接著我們新增一個要用來測試 ```SimpleParser``` 的 ```SimpleParserTest``` 類別，然後撰寫某一個 case 的測試內容

我們想要測試在空字串的情況下， ```ParseAndSum()``` 會不會如期返回 0


```SimpleParserTest```
```php=
<?php


namespace Test;


use Src\SimpleParser;

class SimpleParserTest
{
    public function TestReturnsZeroWhenEmptyString()
    {
        try {
            $SimpleParser = new SimpleParser();
            $result = $SimpleParser->ParseAndSum('');
            if ($result != 0) {
                echo " Error on SimpleParserTest::TestReturnsZeroWhenEmptyString\n
                       ParseAndSum should have returned 0 on an empty string";
            } else {
                echo "SimpleParserTest::TestReturnsZeroWhenEmptyString verify success";
            }
        } catch (\Exception $exception) {
            var_dump($exception->getMessage());
        }
    }
}
```

最後我們將測試的過程註冊到 ```TestMain.php``` 中，這樣我只要去執行這隻檔案就可以執行所有在這之中被註冊過的測試

```TestMain.php```
```php=
<?php

require __DIR__ . '/test/SimpleParserTest.php';
require __DIR__ . '/src/SimpleParser.php';

use Test\SimpleParserTest;

try {
    $SimpleParserTest = new SimpleParserTest();
    $SimpleParserTest->TestReturnsZeroWhenEmptyString();
} catch (Exception $exception) {
    var_dump($exception->getMessage());
}
```

最後用指令去執行這隻可以運行測試的腳本 php 檔

```
php TestMain.php
```

## 1.6 測試驅動開發

大致上理解了該怎麼去寫測試之後，下一個問題就是我們該在何時轉寫測試呢？

有不少人會在軟體完成之後去撰寫測試，但越來越多人選擇在撰寫產品程式碼之前就先寫測試，這種方式稱為 **測試驅動開發(Test-Driven Development, TDD)**

書上的圖 1-3, 1-4 呈現了傳統撰寫方式與 TDD 的不同之處

<!-- <p style="color:red">插入圖片</p>-->

但 TDD 並不能保證你的專案一定會成功，測試的命名、測試的可維護性和可讀性，以及是否測試了正確的內容，或是測試本身有無缺陷或者設計不良都可能左右專案的成敗。本書的重點會把主軸放在將測試寫的到位上。

以下簡述一下 TDD 的開發過程

1. 寫一個失敗的測試，已證明產品功能上的缺失
2. 撰寫產品程式碼，已通過測試
3. 重構程式碼

> 【重構】
>> 重構意味著在不改變功能的前提下，修改其程式碼，改善其可讀性、可維護性


## 1.7 成功進行 TDD 的三種核心技能

想要成功的使用 TDD ，你需要三種核心技能

- 僅僅做到先撰寫測試，並不能保證測試是可維護的、可讀且可靠的。
- 僅僅做到撰寫出可維護、可讀、可靠的測試，並不能保證你能獲得測試先行的各種好處。
- 僅僅做到測試先行，且測試可讀、可維護、可靠，並不能保證你能產出一個設計完善的系統。

學習 TDD 的一個實際方式就是分別學習以上三個技能。

## 1.8 小結

本章節中，作者定義了一個優秀的單元測試應該具有的特質，如下

- 一段自動化程式，他會呼叫另一個方法，然後驗證這個方法或是該類別的邏輯行為或某些預期結果
- 用一個自動化測試框架進行撰寫
- 容易撰寫
- 執行快速
- 能由開發團隊裡的任何人重複執行且得到一樣的結果

並且告訴我們，區分整合測試與單元測試之間的差異非常重要，每當想要寫一個測試的時候，我們就必須去判斷該寫的是單元測試或是整合測試才適當
