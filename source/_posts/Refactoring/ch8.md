---
title: 重構 - Chapter 8 - 移動功能
date: 2020-05-23 19:40:26
tags:
- Refactoring
- 讀書會報告
categories: 
- 讀書會報告
cover: https://picsum.photos/id/1026/800/600
---
# 重構 - Ch8 - 移動功能

## 移動函式(Move Function)

當你的函式參考其他環境的元素數量，比目前所處的物件來得多時，把他與那些元素放在一起，通常可以改善封裝，讓軟體的其他部分比較不依賴模組的細節

```php
/**before*/
class Account 
{
    public function getBankCharge();
    
    //可能根據不同的狀況會有不同的計算方法
    public function getOverDraftCharge();
}

/**after*/
class Account 
{
    public function getBankCharge();
    public function getOverDraftCharge()
    {
        return $this->AccountType->getOverDraftCharge()
    }
}

//將不同類型的實作細節移動到
class AccountType
{
    public function getOverDraftCharge();
}
```

## 移動欄位(Move Field)

- 當傳遞A變數的時候，很常需要伴隨B變數的時候
- 如果對一筆紀錄的修改，常常導致另一筆紀錄的更動時
- 如果我必須更新多個不同結構的同一個欄位時，他應該在同一個地方被，讓他只需要更新一次

```php
/**before*/
class Customer
{
    public $name;
    public $discountRate;
    public $contract
}

class CustomerContract
{
    public $startDate
}

/**after*/
class Customer
{
    public $name;
    public $contract
}

class CustomerContract
{
    public $startDate
    public $discountRate;
}

```

## 將陳述式移入函式(Move Statements into Function)

當我在很多地方重複執行同一段程式時，將他們整理在一起，這樣一來我如果需要做修改就在一個地方就可以修改完成。

```php
/**before*/
public function A()
{
    ...
    //do something
    ...
}

public function B()
{
    ...
    //do something
    ...
}
/**after*/
public function A()
{
    ...
    $this->doSomething();
    ...
}

public function B()
{
    ...
    $this->doSomething();
    ...
}

public function doSomething();

```

## 將陳述式移入呼叫方(Move Statements to Callers)
    
    將陳述式移入函式(Move Statements into Function) 的逆操作

如果有一段code被封裝成function並在多處被使用，但有些地方需要更改呼叫他的方式，就有機會使用這個重構方法。

## 將內聯的程式碼換成函式呼叫式(Replace Inline Code with Function Call)

當看到現有的程式碼做的事情與既有的函式相同時，通常會把現有的程式碼使用既有的函式取代。

```php
/**before*/

$fruits = ['Apple', 'Banana'];
$last_fruit = $fruit[count($fruits)-1];
unset($fruit[count($fruits)-1]);

/**after*/

$fruits = ['Apple', 'Banana'];
$last_fruit = array_pop($fruits);

```

## 移動陳述式(Slide Statements)

將彼此相關的東西放在一起，可以讓程式碼更容易理解。


```php

/**before*/
public function shopping()
{
    $price = getPrice();
    $order = getOrder();
    $charge = '';
    $chargePerUnit = $this->PricePlane->getUnit();
}

/**after*/
public function shopping()
{
    $price = getPrice();
    $order = getOrder();
    $chargePerUnit = $this->PricePlane->getUnit();
    $charge = '';
}

```

## 拆開迴圈(Split Loop)

如果迴圈裡面做了兩件事情，你要修改他時你必須要先了解那兩件事情，把迴圈拆開的話，一方面你只需要去理解你想改的那個迴圈做的事情就好，另外一方面可以使迴圈的內容更容易被重用。

```php

/**before*/

foreach ($array => $key as $value) {
    //do A logic
    //do B logic
}

/**after*/

foreach ($array => $key as $value) {
    //do A logic
}

foreach ($array => $key as $value) {
    //do B logic
}

```

## 將迴圈改成流程(Replace Loop with Pipeline)

用流程來表示邏輯運作比較容易讓人理解，可讓人從上而下閱讀，查看過程經歷了什麼流程。

```php

/**before*/

$programmer_user_names = [];

foreach ($users => $key as $user) {
    if($user['job'] == 'programmer') {
        $programmer_user_names[] = $user['name'];
    }
}

/**after*/

$users = collect(
            [
                ['job' => 'programmer', 'name' => 'name_a'],
                ['job' => 'programmer', 'name' => 'name_b'],
                ['job' => 'aaa', 'name' => 'name_c'],
                ['job' => 'bbb', 'name' => 'name_d'],
            ]
        );
        
$programmer_user_names = $users->filter(
    function ($user) {
        return $user['job'] == 'programmer';
    }
)->pluck('name');

```

## 廢除死碼(Remove Dead Code)

沒有用途的程式碼在你試著了解軟體如何工作時，會成為絆腳石。他沒有明確訊號告知我們可以忽略他，所以多數時候我們被迫去理解這些程式碼的用途。