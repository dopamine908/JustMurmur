---
title: Design Patterns - Creational - Abstract Factory Pattern 抽象工廠模式
date: 2020-08-16 16:44:20
tags:
- Design Patterns
- Creational Patterns
categories: 
- 讀書會報告
cover: https://picsum.photos/id/252/800/600
---

# 設計模式 - Abstract Factory Pattern 抽象工廠模式

## Intro

假設我們今天一開始的需求是必須實作出A銀行的金流操作（有刷卡服務跟行動支付），
而在往後的需求擴充中，我又必須要新增B銀行的金流操作（一樣具有刷卡服務跟行動支付）

## 實作

### 簡單工廠

一開始我們只會有一個A銀行，所以一開始我們用簡單工廠去解決這個問題

```php=

interface ICardPayment
{
    public function payByCard(): string;
}

class ABankCardPayment implements ICardPayment
{
    public function payByCard(): string
    {
        return 'ABank 刷卡付款 -> 回饋:' . $this->getCashReturn();
    }

    public function getCashReturn(): string
    {
        return '3%現金回饋';
    }
}

interface IMobilePayment
{
    public function payByMobile(): string;
}

class ABankMobilePayment implements IMobilePayment
{
    public function payByMobile(): string
    {
        return 'ABank 行動支付 -> 回饋:' . $this->getBonusPoint();
    }

    public function getBonusPoint(): string
    {
        return '100元1點的點數回饋';
    }
}

class ABankServiceFactory
{
    public function createBankService(string $service_type)
    {
        switch ($service_type) {
            case 'card':
                return new ABankCardPayment();
                break;
            case 'mobile':
                return new ABankMobilePayment();
                break;
        }
    }
}

```

使用方式大概如下

```php=

//刷卡付款
$ABankService = new ABankServiceFactory();
$PayService = $ABankService->createBankService('card');
$PayService->payByCard();

//行動支付
$ABankService = new ABankServiceFactory();
$PayService = $ABankService->createBankService('mobile');
$PayService->payByMobile();

```

#### 類別圖

![](https://i.imgur.com/CECSgIQ.png)


#### Plant UML

```=
@startuml

skinparam classAttributeIconSize 0

interface IMobilePayment{
{method} + payByMobile(): string
}

interface ICardPayment{
{method} + payByCard(): string
}

class ABankMobilePayment{
{method} + payByMobile(): string
{method} + getBonusPoint(): string
}

class ABankCardPayment{
{method} + payByCard(): string
{method} + getCashReturn(): string
}

class ABankServiceFactory{
{method} + createBankService(string $service_type)
}

ABankServiceFactory .down.> ABankCardPayment : create
ABankServiceFactory .down.> ABankMobilePayment : create
ABankMobilePayment .down.|> IMobilePayment
ABankCardPayment .down.|> ICardPayment
@enduml

```

### 工廠方法

再來因為我們知道當用簡單工廠實作需求時，如果遇到要擴充的時候會違反開放封閉原則，所以我們在這邊當然也可以選擇用工廠方法去時做這個需求

```php=

interface ICardPayment
{
    public function payByCard(): string;
}

class ABankCardPayment implements ICardPayment
{
    public function payByCard(): string
    {
        return 'ABank 刷卡付款 -> 回饋:' . $this->getCashReturn();
    }

    public function getCashReturn(): string
    {
        return '3%現金回饋';
    }
}

interface IMobilePayment
{
    public function payByMobile(): string;
}

class ABankMobilePayment implements IMobilePayment
{
    public function payByMobile(): string
    {
        return 'ABank 行動支付 -> 回饋:' . $this->getBonusPoint();
    }

    public function getBonusPoint(): string
    {
        return '100元1點的點數回饋';
    }
}

interface ICardPaymentFactory
{
    public function createCardPayment(): ICardPayment;
}

class ABankCardPaymentFactory implements ICardPaymentFactory
{
    public function createCardPayment(): ICardPayment
    {
        return new ABankCardPayment();
    }
}

interface IMobilePaymentFactory
{
    public function createMobilePayment(): IMobilePayment;
}

class ABankMobilePaymentFactory implements IMobilePaymentFactory
{
    public function createMobilePayment(): IMobilePayment
    {
        return new ABankMobilePayment();
    }
}

```

使用方式大概如下

```php=

//刷卡付款
$ABankCardPaymentFactory = new ABankCardPaymentFactory();
$PayService = $ABankCardPaymentFactory->createCardPayment();
$PayService->payByCard();

//行動支付
$ABankMobilePaymentFactory = new ABankMobilePaymentFactory();
$PayService = $ABankMobilePaymentFactory->createMobilePayment();
$PayService->payByMobile();

```

#### 類別圖

![](https://i.imgur.com/8LAOE7K.png)

#### Plant UML

```=
@startuml

skinparam classAttributeIconSize 0

interface IMobilePayment{
{method} + payByMobile(): string
}

interface ICardPayment{
{method} + payByCard(): string
}

class ABankMobilePayment{
{method} + payByMobile(): string
{method} + getBonusPoint(): string
}

class ABankCardPayment{
{method} + payByCard(): string
{method} + getCashReturn(): string
}

interface ICardPaymentFactory{
{method} + createCardPayment(): ICardPayment
}

interface IMobilePaymentFactory{
{method} + createMobilePayment(): IMobilePayment
}

class ABankCardPaymentFactory{
{method} + createCardPayment(): ICardPayment
}

class ABankMobilePaymentFactory{
{method} + createMobilePayment(): IMobilePayment
}

ICardPaymentFactory .left.> ICardPayment : create
IMobilePaymentFactory .left.> IMobilePayment : create
ABankMobilePayment .up.|> IMobilePayment
ABankCardPayment .up.|> ICardPayment
ABankCardPaymentFactory .up.|> ICardPaymentFactory
ABankMobilePaymentFactory .up.|> IMobilePaymentFactory
@enduml
```

以上我們用兩種工廠解決了我們要使用A銀行的金流操作的需求，而在此時你的需求方跟你說我們現在要新增B銀行的金流操作，底下我們來看看如果用簡單工廠及工廠方法擴充B銀行的金流操作會是什麼樣的狀況吧

### 簡單工廠新增B銀行

以簡單工廠為例，我希望我的工廠可以多生產B銀行的刷卡服務跟B銀行的行動支付，所以我新增了兩個case去製造出這兩個具體的服務，當然這種修改方式理所當然地違反了開放封閉原則

```php=

class BBankCardPayment implements ICardPayment
{
    public function payByCard(): string
    {
        return 'BBank 刷卡付款 -> 回饋:' . $this->getCashReturn();
    }

    public function getCashReturn(): string
    {
        return '5%現金回饋';
    }
}

class BBankMobilePayment implements IMobilePayment
{
    public function payByMobile(): string
    {
        return 'BBank 行動支付 -> 回饋:' . $this->getBonusPoint();
    }

    public function getBonusPoint(): string
    {
        return '10元1點的點數回饋';
    }
}

class BankServiceFactory
{
    public function createBankService(string $service_type)
    {
        switch ($service_type) {
            case 'a_card':
                return new ABankCardPayment();
                break;
            case 'a_mobile':
                return new ABankMobilePayment();
                break;
            case 'b_card':
                return new BBankCardPayment();
                break;
            case 'b_mobile':
                return new BBankMobilePayment();
                break;
        }
    }
}

```

使用方式大概如下

```php=

//A銀行刷卡付款
$BankServiceFactory = new BankServiceFactory();
$PayService = $BankServiceFactory->createBankService('a_card');
$PayService->payByCard();

//A銀行行動支付
$BankServiceFactory = new BankServiceFactory();
$PayService = $BankServiceFactory->createBankService('a_mobile');
$PayService->payByMobile()

//B銀行刷卡付款
$BankServiceFactory = new BankServiceFactory();
$PayService = $BankServiceFactory->createBankService('b_card');
$PayService->payByCard();

//B銀行行動支付
$BankServiceFactory = new BankServiceFactory();
$PayService = $BankServiceFactory->createBankService('b_mobile');
$PayService->payByMobile()

```

#### 類別圖

![](https://i.imgur.com/GzGtXHI.png)

#### Plant UML

```=
@startuml

skinparam classAttributeIconSize 0

interface IMobilePayment{
{method} + payByMobile(): string
}

interface ICardPayment{
{method} + payByCard(): string
}

class ABankMobilePayment{
{method} + payByMobile(): string
{method} + getBonusPoint(): string
}

class ABankCardPayment{
{method} + ABankCardPayment(): string
{method} + getCashReturn(): string
}

class BBankMobilePayment{
{method} + payByMobile(): string
{method} + getBonusPoint(): string
}

class BBankCardPayment{
{method} + payByCard(): string
{method} + getCashReturn(): string
}

class BankServiceFactory{
{method} + createBankService(string $service_type)
}

BankServiceFactory .down.> ABankCardPayment : create
BankServiceFactory .down.> ABankMobilePayment : create
BankServiceFactory .down.> BBankCardPayment : create
BankServiceFactory .down.> BBankMobilePayment : create
ABankMobilePayment .down.|> IMobilePayment
ABankCardPayment .down.|> ICardPayment
BBankMobilePayment .down.|> IMobilePayment
BBankCardPayment .down.|> ICardPayment
@enduml
```
### 工廠方法新增B銀行

再來我們嘗試著用工廠方法去擴充我們B銀行的金流操作，可以看到當我使用工廠方法去做擴充的時候，一個銀行有n種金流服務時，我就必須要新增該金流服務以及他對應的工廠，總共要新增n*2個類別，也就是說我們除了上方的類別外，還需要新增以下4個類別

```php=

class BBankCardPayment implements ICardPayment
{
    public function payByCard(): string
    {
        return 'BBank 刷卡付款 -> 回饋:' . $this->getCashReturn();
    }

    public function getCashReturn(): string
    {
        return '5%現金回饋';
    }
}

class BBankMobilePayment implements IMobilePayment
{
    public function payByMobile(): string
    {
        return 'BBank 行動支付 -> 回饋:' . $this->getBonusPoint();
    }

    public function getBonusPoint(): string
    {
        return '10元1點的點數回饋';
    }
}

class BBankCardPaymentFactory implements ICardPaymentFactory
{
    public function createCardPayment(): ICardPayment
    {
        return new BBankCardPayment();
    }
}

class BBankMobilePaymentFactory implements IMobilePaymentFactory
{
    public function createMobilePayment(): IMobilePayment
    {
        return new BBankMobilePayment();
    }
}

```

使用方式大概如下

```php=

//A銀行刷卡付款
$ABankCardPaymentFactory = new ABankCardPaymentFactory();
$PayService = $ABankCardPaymentFactory->createCardPayment();
$PayService->payByCard();

//A銀行行動支付
$ABankMobilePaymentFactory = new ABankMobilePaymentFactory();
$PayService = $ABankMobilePaymentFactory->createMobilePayment();
$PayService->payByMobile();

//B銀行刷卡付款
$BBankCardPaymentFactory = new BBankCardPaymentFactory();
$PayService = $BBankCardPaymentFactory->createCardPayment();
$PayService->payByCard();

//B銀行行動支付
$BBankMobilePaymentFactory = new BBankMobilePaymentFactory();
$PayService = $BBankMobilePaymentFactory->createMobilePayment();
$PayService->payByMobile();

```

#### 類別圖

![](https://i.imgur.com/T6rJiA3.png)


#### Plant UML

```=
@startuml

skinparam classAttributeIconSize 0

interface IMobilePayment{
{method} + payByMobile(): string
}

interface ICardPayment{
{method} + payByCard(): string
}

class ABankMobilePayment{
{method} + payByMobile(): string
{method} + getBonusPoint(): string
}

class ABankCardPayment{
{method} + payByCard(): string
{method} + getCashReturn(): string
}

class BBankMobilePayment{
{method} + payByMobile(): string
{method} + getBonusPoint(): string
}

class BBankCardPayment{
{method} + payByCard(): string
{method} + getCashReturn(): string
}

interface ICardPaymentFactory{
{method} + createCardPayment(): ICardPayment
}

interface IMobilePaymentFactory{
{method} + createMobilePayment(): IMobilePayment
}

class ABankCardPaymentFactory{
{method} + createCardPayment(): ICardPayment
}

class ABankMobilePaymentFactory{
{method} + createMobilePayment(): IMobilePayment
}

class BBankCardPaymentFactory{
{method} + createCardPayment(): ICardPayment
}

class BBankMobilePaymentFactory{
{method} + createMobilePayment(): IMobilePayment
}

ICardPaymentFactory .left.> ICardPayment : create
IMobilePaymentFactory .left.> IMobilePayment : create
ABankMobilePayment .up.|> IMobilePayment
ABankCardPayment .up.|> ICardPayment
BBankMobilePayment .up.|> IMobilePayment
BBankCardPayment .up.|> ICardPayment
ABankCardPaymentFactory .up.|> ICardPaymentFactory
ABankMobilePaymentFactory .up.|> IMobilePaymentFactory
BBankCardPaymentFactory .up.|> ICardPaymentFactory
BBankMobilePaymentFactory .up.|> IMobilePaymentFactory
@enduml
```
### 抽象工廠

底下我們嘗試著使用抽象工廠去重新將程式碼整理一次

```php=

interface ICardPayment
{
    public function payByCard(): string;
}

class ABankCardPayment implements ICardPayment
{
    public function payByCard(): string
    {
        return 'ABank 刷卡付款 -> 回饋:' . $this->getCashReturn();
    }

    public function getCashReturn(): string
    {
        return '3%現金回饋';
    }
}

class BBankCardPayment implements ICardPayment
{
    public function payByCard(): string
    {
        return 'BBank 刷卡付款 -> 回饋:' . $this->getCashReturn();
    }

    public function getCashReturn(): string
    {
        return '5%現金回饋';
    }
}

interface IMobilePayment
{
    public function payByMobile(): string;
}

class ABankMobilePayment implements IMobilePayment
{
    public function payByMobile(): string
    {
        return 'ABank 行動支付 -> 回饋:' . $this->getBonusPoint();
    }

    public function getBonusPoint(): string
    {
        return '100元1點的點數回饋';
    }
}

class BBankMobilePayment implements IMobilePayment
{
    public function payByMobile(): string
    {
        return 'BBank 行動支付 -> 回饋:' . $this->getBonusPoint();
    }

    public function getBonusPoint(): string
    {
        return '10元1點的點數回饋';
    }
}

interface IBankServiceFactory
{
    public function createCardPaymentService(): ICardPayment;

    public function createMobilePaymentService(): IMobilePayment;
}

class ABankServiceFactory implements IBankServiceFactory
{
    public function createCardPaymentService(): ICardPayment
    {
        return new ABankCardPayment();
    }

    public function createMobilePaymentService(): IMobilePayment
    {
        return new ABankMobilePayment();
    }
}

class BBankServiceFactory implements IBankServiceFactory
{
    public function createCardPaymentService(): ICardPayment
    {
        return new BBankCardPayment();
    }

    public function createMobilePaymentService(): IMobilePayment
    {
        return new BBankMobilePayment();
    }
}

```

使用方式大概如下

```php=

//A銀行刷卡付款
$ABankServiceFactory = new ABankServiceFactory();
$PayService = $ABankServiceFactory->createCardPaymentService();
$PayService->payByCard()

//A銀行行動支付
$ABankServiceFactory = new ABankServiceFactory();
$PayService = $ABankServiceFactory->createMobilePaymentService();
$PayService->payByMobile();

//B銀行刷卡付款
$BBankServiceFactory = new BBankServiceFactory();
$PayService = $BBankServiceFactory->createCardPaymentService();
$PayService->payByCard();

//B銀行行動支付
$BBankServiceFactory = new BBankServiceFactory();
$PayService = $BBankServiceFactory->createMobilePaymentService();
$PayService->payByMobile();

```

#### 類別圖

![](https://i.imgur.com/U3CkT7k.png)


#### Plant UML

```=
@startuml

skinparam classAttributeIconSize 0

interface IMobilePayment{
{method} + payByMobile(): string
}

interface ICardPayment{
{method} + payByCard(): string
}

class ABankMobilePayment{
{method} + payByMobile(): string
{method} + getBonusPoint(): string
}

class ABankCardPayment{
{method} + ABankCardPayment(): string
{method} + getCashReturn(): string
}

class BBankMobilePayment{
{method} + payByMobile(): string
{method} + getBonusPoint(): string
}

class BBankCardPayment{
{method} + payByCard(): string
{method} + getCashReturn(): string
}

interface IBankServiceFactory{
{method} + createCardPaymentService(): ICardPayment
{method} + createMobilePaymentService(): IMobilePayment
}

class ABankServiceFactory{
{method} + createCardPaymentService(): ICardPayment
{method} + createMobilePaymentService(): IMobilePayment
}

class BBankServiceFactory{
{method} + createCardPaymentService(): ICardPayment
{method} + createMobilePaymentService(): IMobilePayment
}

ABankMobilePayment .down.|> IMobilePayment
ABankCardPayment .down.|> ICardPayment
BBankMobilePayment .down.|> IMobilePayment
BBankCardPayment .down.|> ICardPayment
ABankServiceFactory .up.|> IBankServiceFactory
BBankServiceFactory .up.|> IBankServiceFactory
IBankServiceFactory .up.> ICardPayment : create
IBankServiceFactory .up.> IMobilePayment : create
@enduml
```

## 時機

- 當生產的產品有多個系列時（如上例子，同一種金流服務有多個銀行支援）
- 期望產品系列可以簡易的抽換時
- 當你期望產品系列的搭配可以較為順利時
- 一整個系列的產品必須一起使用，又要確保搭配上不會錯誤時

## 目的

以同一個介面來建立一系列相關或是相依的產品，不需指定個物件真正所屬的具體類別

## 類別圖

![](https://i.imgur.com/LnFEMC9.png)

### Plant UML

```
@startuml

skinparam classAttributeIconSize 0

interface IProductA
interface IProductB
class ConcreteProductB1
class ConcreteProductA1
class ConcreteProductB2
class ConcreteProductA2

interface IAbstractFactory{
{method} + createProductA(): IProductA
{method} + createProductB(): IProductB
}

class ConcreteFactory1{
{method} + createProductA(): IProductA
{method} + createProductB(): IProductB
}

class ConcreteFactory2{
{method} + createProductA(): IProductA
{method} + createProductB(): IProductB
}

ConcreteProductB1 .down.|> IProductB
ConcreteProductA1 .down.|> IProductA
ConcreteProductB2 .down.|> IProductB
ConcreteProductA2 .down.|> IProductA
ConcreteFactory1 .up.|> IAbstractFactory
ConcreteFactory2 .up.|> IAbstractFactory
IAbstractFactory .up.> IProductA : create
IAbstractFactory .up.> IProductB : create
@enduml
```

## 優點

- 不同的工廠產出的產品因為介面相同，故彼此相容，大大降低了抽換產品系列的難易度
- 可以避免產品與使用端的程式碼的耦合
- 開放封閉原則：新增新的產品系列只需新增相同介面的類別就可以了，不會改變原本的程式碼
- 增進了產品物件一致性

## 缺點

- 程式碼複雜化
- 開放封閉原則：如果我今天要新增的不是新的產品系列，而是產品的新的行為（假想我們今天要新增轉帳付款），如此一來經由介面的擴充，及新增帶有新介面的產品外，我們勢必還得要對現有的工廠介面作出修改，連帶著就必須修改到實作該介面的所有實體工廠
