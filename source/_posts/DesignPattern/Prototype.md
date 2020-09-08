---
title: Design Patterns - Creational - Prototype Pattern 原型模式
date: 2020-07-11 19:24:55
tags:
- Design Patterns
- Creational Patterns
categories: 
- 讀書會報告
cover: https://picsum.photos/id/1073/800/600
---

# 設計模式 - Prototype Pattern 原型模式

## Intro

假設我們今天要瘋狂產出同一台MacBook裝置，而裝置中皆含有相同規格的配件


## 實作

首先，如果是一般的建立物件方法及使用方法，我們會先寫一個物件，然後再需要用到它的時候將他new出來

```php=

class MacBook
{
    private $cpu;
    private $ssd;
    private $ram;
    private $monitor;
    private $keyboard;

    public function __construct()
    {
        $this->cpu = 'i7';
        $this->ssd = '256G';
        $this->ram = '16G';
        $this->monitor = '13 inch';
        $this->keyboard = 'default keyboard';
    }
}

```

假設製造MacBook的建構子中做的事情相當複雜，我要生成很多MacBook就意味著我必須要負擔相同且重複又複雜的建構過程，而這些過程卻是完全一樣的。

```php=
$MacBook1 = new MacBook();
$MacBook2 = new MacBook();
$MacBook3 = new MacBook();
$MacBook4 = new MacBook();
$MacBook5 = new MacBook();
```

這無疑是造成資源上的浪費，所以底下我們實作普遍上來說的Prototype

```php=

class MacBook
{
    private $cpu;
    private $ssd;
    private $ram;
    private $monitor;
    private $keyboard;

    public function __construct()
    {
        $this->cpu = 'i7';
        $this->ssd = '256G';
        $this->ram = '16G';
        $this->monitor = '13 inch';
        $this->keyboard = 'default keyboard';
    }
    
    public function clone()
    {
        return $this;
    }
}

```

我們多新增了一個clone方法，我們的MacBook可以透過clone回傳一個已經建構好的MacBook而不需要再經過那相同且複雜的建構子過程。

```php=
$MacBook = new MacBook();
$MacBook1 = $MacBook->clone();
$MacBook2 = $MacBook->clone();
$MacBook3 = $MacBook->clone();
$MacBook4 = $MacBook->clone();
$MacBook5 = $MacBook->clone();
```

> 注意上方程式碼只是將Prototype的精神實作出來
> 事實上這樣做的話每個被clone的類別的reference會變成完全相同。

其實現在一般的語言都會實作這個模式讓大家可以直接使用了，例如PHP本身就有clone()語法可以使用，也提供了magic method __clone()讓人可以自定義當呼叫clone的時候可以額外多做的事情

```php=

class MacBook
{
    private $cpu;
    private $ssd;
    private $ram;
    private $monitor;
    private $keyboard;

    public function __construct()
    {
        $this->cpu = 'i7';
        $this->ssd = '256G';
        $this->ram = '16G';
        $this->monitor = '13 inch';
        $this->keyboard = 'default keyboard';
    }

    public function __clone()
    {
        
    }
}

```

```php=
$MacBook=new MacBook();
$MacBook1 = clone $MacBook;
$MacBook2 = clone $MacBook;
$MacBook3 = clone $MacBook;
$MacBook4 = clone $MacBook;
$MacBook5 = clone $MacBook;
```

### 淺複製(Shallow Copy)

我們常常遇到一種狀況，就是物件內的變數其實還是一個物件，而這種時候如果使用的是預設的clone則會發生一種狀況
在clone完之後會發現物件內的物件reference會全部指向同一個位置
此現象我們稱為淺複製

如底下的例子，如果我將RAM獨立出來變成一個類別，這樣子去clone的話我們會發現所有的MacBook中的RAM的reference都會指向同一個位置

```php=

class MacBook2RAM
{
    private $cpu;
    private $ssd;
    /**
     * @var RAM
     */
    private $ram;
    private $monitor;
    private $keyboard;

    public function __construct()
    {
        $this->cpu = 'i7';
        $this->ssd = '256G';
        $this->ram = new RAM();
        $this->monitor = '13 inch';
        $this->keyboard = 'default keyboard';
    }
    
    public function __clone()
    {
    
    }
}

class RAM
{
    public $ram1 = '8G';
    public $ram2 = '8G';
}

```

```php=
$MacBook=new MacBook();
$MacBook1 = clone $MacBook;
$MacBook2 = clone $MacBook;
$MacBook3 = clone $MacBook;
$MacBook4 = clone $MacBook;
$MacBook5 = clone $MacBook;
```

### 深複製(Deep Copy)

如上面的情況，如果我們沒有要更改各自的RAM內容的話，就不會有什麼問題發生
但如果我們今天想要更改某幾台的RAM配置的話，在相同reference的前提下，會發生我改一個全部就一起被更改的問題，導致我不能單獨為特定幾台去更改RAM的配置

所以底下實作了搭配__clone去進行深複製的寫法

```php=
class MacBook2RAM
{
    private $cpu;
    private $ssd;
    /**
     * @var RAM
     */
    private $ram;
    private $monitor;
    private $keyboard;

    public function __construct()
    {
        $this->cpu = 'i7';
        $this->ssd = '256G';
        $this->ram = new RAM();
        $this->monitor = '13 inch';
        $this->keyboard = 'default keyboard';
    }

    public function __clone()
    {
        $this->ram = clone $this->ram;
    }
}
```

## 時機

- 當目標物件的建構需付出龐大代價時
- 當我們需要對象物件的副本時

## 目的

制定可以用原型個體生成的物件類型，之後只需要複製此物件即可生成新物件

## 類別圖

![](https://i.imgur.com/PC7euN5.png)


### Plant UML

```=
@startuml

skinparam classAttributeIconSize 0

interface Prototype{
{method} +clone()
}

class ConcretePrototypeA{
{method} +clone()
}
class ConcretePrototypeB{
{method} +clone()
}

Prototype <|.. ConcretePrototypeA
Prototype <|.. ConcretePrototypeB

@enduml
```


## 優點

- 不需要跟其他的類別有耦合（例如工廠）
- 避免反覆付出建構子中的初始化邏輯代價，方便的生成複雜的類別

## 缺點

- 如果物件結構中含有循環指涉([circular reference](https://stackoverflow.com/questions/36913443/how-to-handle-class-object-with-circular-references))的情形，情況會變得相當複雜且崩潰
