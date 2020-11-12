---
title: Design Patterns - Behavioral - Chain of Responsibility Pattern 職責鏈模式
date: 2020-10-05 17:12:55
tags:
- Design Patterns
- Behavioral Patterns
categories: 
- 讀書會報告
cover: https://picsum.photos/id/403/800/600
---

# 設計模式 - Chain of Responsibility Pattern 職責鏈模式

## Intro

假設我們今天要設計一個流程，當處理人處理不來的時候則轉交給下一個處理的人

例如今天產品出了問題
小問題的話CEO就可以唬兩句就解決了
但如果問題大一點的話，可能就要靠經理出來圓個場
但如果今天問題更大的話，
那只好摘贓底下的工程師，讓工程師加班來解決問題

以下我們來實作一下這個處理流程的程式

## 實作

我們先定義一下出問題的類別，好讓各階層處理的人員可以判斷這個問題有多大條

```php=

class Trouble
{
    private $Level;

    public function __construct(int $Level)
    {
        $this->Level = $Level;
    }
}

```

我們希望每個腳色都有一個固定的介面可以去處理問題，所以這邊我們先新增一個抽象類別Handler，讓他制定一個handleTrouble方法，讓外部知道我們只要去使用這個方法就可以解決問題。
另外因為處理不來的事情我們就要丟包給下一個負責的人，所以我們新增一個setNextHandler方法讓外部可以設定處理不來的時候需要交付給誰

```php=

abstract class Handler
{
    protected $NextHandler;

    abstract public function handleTrouble(Trouble $trouble): string;

    public function setNextHandler(Handler $handler)
    {
        $this->NextHandler = $handler;
    }
}

```

接下來我們來新增三個腳色，CEO、Manager、Engineer去繼承Handler
讓他們都用handleTroubl去接收問題並處理，然後處理不來的就丟包給下一個人

```php=

class CEO extends Handler
{
    public function handleTrouble(Trouble $trouble): string
    {
        if ($trouble->Level <= 10) {
            return 'CEO can handle';
        } else {
            return $this->NextHandler->handleTrouble($trouble);
        }
    }
}

class Manager extends Handler
{
    public function handleTrouble(Trouble $trouble): string
    {
        if ($trouble->Level <= 100) {
            return 'Manager can handle';
        } else {
            return $this->NextHandler->handleTrouble($trouble);
        }
    }
}

class Engineer extends Handler
{
    public function handleTrouble(Trouble $trouble): string
    {
        return 'Engineer can handle';
    }
}

```

使用的情況會如下，我們會將CEO、Manager、Engineer職責順序透過setNextHandler去做設定，而今天不管出了甚麼問題，我們就是指向職責最前端的負責人，並且呼叫他的handleTrouble方法，這個問題便可以被解決

```php=

$SmallTrouble = new Trouble(1);
$MediumTrouble = new Trouble(11);
$BigTrouble = new Trouble(111);

$Engineer = new Engineer();
$Manager = new Manager();
$CEO = new CEO();
$Manager->setNextHandler($Engineer);
$CEO->setNextHandler($Manager);

//return 'CEO can handle'
$CEO->handleTrouble($SmallTrouble);

//return 'Manager can handle'
$CEO->handleTrouble($MediumTrouble);

//return 'Engineer can handle'
$CEO->handleTrouble($BigTrouble);

```

## 時機

- 類別對於判斷狀況的職責過多的時候
- 執行程式時無法提前知道會遇到什麼樣的狀況時
- 當必須按照特定順序執行多個處理程序時
- 當我們期望可以在執行時動態更改執行順序時

## 目的

讓多個物件都有機會處理某一個訊息，以降低發送者與接收者之間的耦合關係。他將接收者的物件串連起來，讓訊息流經其中，直到被處理為止。

## 類別圖

![](https://i.imgur.com/KHBXlfD.png)

### Plant UML

```
@startuml

skinparam classAttributeIconSize 0

class Trouble{
{field} + $Level : int
}

abstract class Handler{
{field} # $NextHandler : Handler
{abstract} + handleTrouble(Trouble $trouble)
{method} + setNextHandler(Handler $handler)
}

class ConcreteHandlerA{
{field} # $NextHandler : Handler
{method} + handleTrouble(Trouble $trouble)
}

class ConcreteHandlerB{
{field} # $NextHandler : Handler
{method} + handleTrouble(Trouble $trouble)
}

package Client <<Rectangle>> {
}

ConcreteHandlerA -up-|> Handler
ConcreteHandlerB -up-|> Handler
ConcreteHandlerA o-up-> Handler : NextHandler
ConcreteHandlerB o-up-> Handler : NextHandler
Client -right-> Handler

@enduml
```

## 優點

- 可以控制請求處理的順序
- 發送者及接收者不必直接相關，而串列裡的個別物件也不必知道整條串鏈的構造
- 單一職責原則。可以將調用及執行操作的類別職責分離
- 開放封閉原則。可以在不破壞使用者端程式碼的狀況下新增新的處理順序

## 缺點

- 某些請求可能會無法被執行
- 效率較低，請求有可能被歷遍之後才得以被處理
