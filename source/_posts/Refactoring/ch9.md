---
title: 重構 - Chapter 9 - 組織資料
date: 2020-05-23 19:43:32
tags:
- Refactoring
- 讀書會報告
categories: 
- 讀書會報告
cover: https://picsum.photos/id/221/800/600
---
# 重構 - ch9 - 組織資料

## 拆開變數(Split Variable)

我們經常用變數來保存程式碼產生的結果，以方便引用。如果同一個變數被設定了多次，可能代表變數在方法內的責任超過一個。

```php

/**before*/

$temp = $length * $width;
$temp = $temp * $height;

/**after*/

$area = $length * $width;
$volume = $area * $height;

```

## 更改欄位名稱(Rename Field)

名稱非常重要，尤其當一個資料結構在很多地方都被使用的時候，好的名稱可以幫助人更快理解程式。

```php

/**before*/

class Orgnization
{
    public $name;
}

/**after*/

class Orgnization
{
    public $title;
}


```

## 將衍生的變數換成查詢函式(Replace Derived Variable with Query)

資料通常都會跟你的程式碼有相當程度上的耦合關係，所以當我們修改了某個資料，有可能會造成有使用到這筆資料的程式碼也發生我們無法預期的改變。
移除「可用簡單的計算來取代的變數」，介意減少區域變數，達到提高穩定性的作用。

## 將參考改成值(Change Reference to Value)

當我內嵌一個物件或資料結構時，可以將內部物件當成參考(Reference)或值(Value)，如果我將欄位視為值的話，可以把它變成值物件。
值物件通常比較容易理解，通常他們是不可變的。
值物件在分散(distributed)與並行(concurrent)系統中相當好用。

```php

/**before*/

class Person
{
    public function __construct()
    {
        $this->telephoneNumber = new TelephoneNumber();
    }
}

class TelephoneNumber
{
    
}

/**after*/

class TelephoneNumber
{
    public function __construct(int $areaCode, int $number)
    {
        $this->setAreaCode($areaCode);
        $this->setNumber($number);
    }
}


```

## 將值改成參考(Change Value to Reference)

將參考改成值的逆操作，當我想要讓多個物件對於共用物件的任何變動都可以被看到，就會將該物件做成參考。