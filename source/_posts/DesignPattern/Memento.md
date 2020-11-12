---
title: Design Patterns - Behavioral - Memento Pattern 備忘錄模式
date: 2020-10-19 20:47:23
tags:
- Design Patterns
- Behavioral Patterns
categories: 
- 讀書會報告
cover: https://picsum.photos/id/532/800/600
---

# 設計模式 - Memento Pattern 備忘錄模式

## Intro

假設今天我們想製作一個可以"上一步"的編輯器

## 實作

為了顧及功能的靈活性，我們做了一些分工
我們建立一個IDE的class(IDEOriginator)，讓他掌管原本IDE該有的所有功能
再來我們需要一個儲存狀態的class(Memento)，讓我們可以把IDE的狀態變成某種資料格式儲存起來
最後既然我們需要將IDE的狀態儲存起來，我們再新增一個可以管理儲存內容的class(CareTaker)
接著來看看怎麼將他一步一步建構出來吧

首先，我們看一下IDEOriginator
除了他的原本的功能外，我們為了儲存狀態及設定狀態多給了他2個function

1. createMemento : 將現在IDE的狀態轉化為一個Memento物件，讓這個物件負責掌管IDEOriginator的參數設定
2. setMemento : 可以將Memento輸入，並且將IDEOriginator的參數全部設定為Memento所記錄的樣子

```php=

class IDEOriginator
{
    private $action;

    //Restore Memento
    public function setMemento(Memento $memento)
    {
        $this->action = $memento->getState();
    }

    //Return save memento
    public function createMemento(): Memento
    {
        return new Memento($this->action);
    }

    public function setAction(string $action)
    {
        $this->action = $action;
    }

    public function getAction(): string
    {
        return $this->action;
    }
}

```

而負責儲存狀態的Memento則像下面這樣，僅負責儲存狀態

```php=

class Memento
{
    private $state;

    public function __construct(string $state)
    {
        $this->setMemento($state);
    }

    public function setMemento(string $state)
    {
        $this->state = $state;
    }

    public function getState()
    {
        return $this->state;
    }
}

```

而這些Memento的管理則交由CareTaker負責

```php=

class CareTaker
{
    private $saves = [];

    public function getSave(int $index): Memento
    {
        return $this->saves[$index - 1];
    }

    public function saveMemento(Memento $memento)
    {
        $this->saves[] = $memento;
    }
}

```

使用方式則如下

```php=
$Originator = new IDEOriginator();
$CareTaker = new CareTaker();

$Originator->setAction('press a');
$Step1Memento=$Originator->createMemento();
$CareTaker->saveMemento($Step1Memento);

$Originator->setAction('press b');
$Step2Memento = $Originator->createMemento();
$CareTaker->saveMemento($Step2Memento);

$Originator->setAction('press c');
$Step3Memento = $Originator->createMemento();
$CareTaker->saveMemento($Step3Memento);

$Originator->setMemento($CareTaker->getSave(1));
$Originator->getAction(); // return 'press a'

$Originator->setMemento($CareTaker->getSave(2));
$Originator->getAction(); // return 'press b'

$Originator->setMemento($CareTaker->getSave(3));
$Originator->getAction(); // return 'press c'
```

## 時機

- 當我們的程式需提供先前的狀態給使用者使用時
- 當我們想將狀態做封裝不被使用者直接取用時

## 目的

在不違反封裝性的前提下，捕捉物件的內部狀態並儲存在外面，以便日後回復至此一狀態

## 類別圖

![](https://i.imgur.com/DSGJ5QA.png)

### Plant UML

```
@startuml

skinparam classAttributeIconSize 0

class Originator{
{field} - $state

{method} + setMemento(Memento $memento)
{method} + createMemento() : Memento
}

class Memento{
{field} - $state

{method} + __construct($state)
{method} + setMemento($state)
{method} + getMemento() : $state
}

class CareTaker

Originator .right.> Memento
Memento o-right-> CareTaker

@enduml
```

## 優點

- 可以恢復物件先前的狀態
- 可將"主要功能"、"狀態"、"狀態管理"三個職責分離

## 缺點

- 紀錄狀態本身就是一種記憶體消耗
- CareTaker如果不知道Originator的某些特殊的生命週期或是運作前提，可能會在移除Memento的過程發生問題
