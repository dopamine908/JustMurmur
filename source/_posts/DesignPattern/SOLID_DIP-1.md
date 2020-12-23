---
title: SOLID - Dependency Inversion Principle - 依賴反轉轉原則
date: 2020-08-28 20:14:52
tags:
- SOLID
categories: 
- 日常開發筆記
cover: https://picsum.photos/id/325/800/600
---

# SOLID - DIP (1) - DIP 與 IoC

## Dependency Inversion Principle - 依賴反轉轉原則

首先，我們先來看一段關於依賴反轉原則的敘述

### 定義

[wiki](https://en.wikipedia.org/wiki/Dependency_inversion_principle)
> A.High-level modules should not depend on low-level modules. Both should depend on abstractions (e.g. interfaces).
> B.Abstractions should not depend on details. Details (concrete implementations) should depend on abstractions.
> > A. 高階模組不應該依賴低階模組，兩者都應該依賴於抽象(例如interface)
> > B. 抽象不應該依賴於具體，而具體實現應該依賴於抽象介面

如果你像我一樣看完之後一頭霧水的話，這邊應該大致上會產生幾個疑問

1. 什麼是高階模組?什麼又是低階模組?
2. 什麼是具體?什麼是抽象?
3. 什麼是依賴?
4. 什麼是抽象依賴於具體?
5. 什麼是具體依賴於抽象?

## 依賴反轉? 先了解下什麼是"依賴"吧!

### 例子

首先，我們先來看個例子吧!
我們需要一個收銀台，我們先新增一個CheckoutCounter類別，再來我們的收銀檯是收現金的，所以我再新增一個Cash類別

```php=

class Cash
{
    public function payByCash(): string
    {
        return 'pay by cash';
    }
}

class CheckoutCounter
{
    private $Cash;

    public function __construct()
    {
        $this->Cash = new Cash();
    }

    public function pay(): string
    {
        return $this->Cash->payByCash();
    }
}

```

### 使用方式

使用的時候如下

```php=
$CheckoutCounter = new CheckoutCounter();
$CheckoutCounter->pay();
```

### 類別圖

![](https://i.imgur.com/JCIKm8R.png)

#### plant uml

```=
@startuml

skinparam classAttributeIconSize 0

class Cash {
{method} + payByCash():string
}

class CheckoutCounter {
{field} - Cash $Cash
{method} + pay():string
}

CheckoutCounter .right.> Cash

@enduml
```

以上面的例子來說，CheckoutCounter使用了Cash，CheckoutCounter掌握了Cash的使用權力，所以這種狀況下我們會將CheckoutCounter稱為"高階模組"，而被使用的Cash稱為"低階模組"。

由上面的例子我們也可以發現，CheckoutCounter的運作必須要使用到Cash，一旦沒有了Cash就沒辦法運作，像是這樣誰沒有了誰就活不下去的關係我們就稱之為"依賴"。

再來我們是用new Cash()這種方式去產生依賴，也就是說我們將Cash創造出來從而建構出了依賴關係，這邊的new Cash()因為實際被創造出來，我們稱new Cash()為"具體"，而此種依賴方式我們稱之為"依賴具體"。

## 依賴"反轉"? 反轉什麼?

到目前為止看起來並沒有甚麼問題，但如果我想新增收銀的方法呢?如果我想新增刷卡的付款方式，之後又想新增行動支付的付款方式呢?

這個時候我們的CheckoutCounter類別很可能會變成下面這樣的寫法

### 例子

```php=

class Cash
{
    public function payByCash(): string
    {
        return 'pay by cash';
    }
}

class Card
{
    public function payByCard(): string
    {
        return 'pay by card';
    }
}

class Mobile
{
    public function payByMobile(): string
    {
        return 'pay by mobile';
    }
}

class CheckoutCounter
{
    private $Cash;
    private $Card;
    private $Mobile;

    public function __construct()
    {
        $this->Cash = new Cash();
        $this->Card = new Card();
        $this->Mobile = new Mobile();
    }

    public function payByCash(): string
    {
        return $this->Cash->payByCash();
    }

    public function payByCard(): string
    {
        return $this->Card->payByCard();
    }

    public function payByMobile(): string
    {
        return $this->Mobile->payByMobile();
    }
}
```

### 使用方式

使用起來會像下方這樣

```php=
$CheckoutCounter = new CheckoutCounter();
$CheckoutCounter->payByCash(); //付現
$CheckoutCounter->payByCard(); //刷卡
$CheckoutCounter->payByMobile(); //行動支付
```


### 類別圖

![](https://i.imgur.com/pUV6zjg.png)

#### plant uml

```=
@startuml

skinparam classAttributeIconSize 0

class Cash {
{method} + payByCash():string
}

class Card {
{method} + payByCard():string
}

class Mobile {
{method} + payByMobile():string
}

class CheckoutCounter {
{field} - Cash $Cash
{field} - Card $Card
{field} - Mobile $Mobile
{method} + payByCash():string
{method} + payByCard():string
{method} + payByMobile():string
}

CheckoutCounter .down.> Cash : Concrete Dependent
CheckoutCounter .down.> Card : Concrete Dependent
CheckoutCounter .down.> Mobile : Concrete Dependent

@enduml
```

因應擴充的需求，我們多新增了兩個付款相關的類別，並且修改了CheckoutCounter。
這樣的方式違反了開放封閉原則，更何況我們可以設想當今天的付款步驟相當複雜，而我們要一邊擴充還要一邊去管理步驟的邏輯，可預見大概會相當不好去做擴充。

在解決這樣的問題之前，我先來介紹一個評估依賴的方式，這個方式是我之前在某個前輩的文章中看到的，我相當喜歡，可惜前輩的文章現在已經找不到了。

首先，我們想要去評估依賴關係強還是弱，那麼如果我可以將依賴關係給量化，將整個系統的依賴關係變為一個數字，那麼我便可以很輕易地去比較出他的依賴關係強弱了。

現在我們來訂定一個規則，當我今天的依賴為具體的時候，這個依賴關係就可以獲得2分，而當我今天我的依賴為抽象的時候(interface or abstract)，則這段依賴關係取得1分的依賴分數，最後我將總分對於類別數取平均，當作整個系統的依賴測量分數。

那麼以上面的類別圖來說，我們可以列出向下面這樣的表格

|                 | CheckoutCounter | Cash | Card | Mobile |
| ----------------|-----------------|------|------|--------|
| CheckoutCounter | -               | 2    | 2    | 2      |
| Cash            | 2               | -    | 0    | 0      |
| Card            | 2               | 0    | -    | 0      |
| Mobile          | 2               | 0    | 0    | -      |

則現在這樣的設計，我們可以取得我們整個系統內的依賴分數為(12/4)=3

底下我們換一個設計方式，我們對於所有的付款類別提出一個interface，而在CheckoutCounter中我們改變運作方式，原本在CheckoutCounter中我們會需要事先將三種付款方式都準備齊全才能收銀，現在我想要變成依據客人指定的付款方式我們才去準備收銀的媒介。

### 例子

```php=

interface IPayment
{
    public function pay(): string;
}

class Cash implements IPayment
{
    public function pay(): string
    {
        return 'pay by cash';
    }
}

class Card implements IPayment
{
    public function pay(): string
    {
        return 'pay by card';
    }
}

class Mobile implements IPayment
{
    public function pay(): string
    {
        return 'pay by mobile';
    }
}

class CheckoutCounter
{
    private $Payment;

    public function __construct(IPayment $Payment)
    {
        $this->Payment = $Payment;
    }

    public function pay(): string
    {
        return $this->Payment->pay();
    }
}

```

### 使用方式

顧客自己去決定要使用哪種方式付款，使用方式變為下面這樣

```php=
//刷卡
$CheckoutCounter = new CheckoutCounter(new Cash());
$CheckoutCounter->pay();

//付現
$CheckoutCounter = new CheckoutCounter(new Card());
$CheckoutCounter->pay();

//行動支付
$CheckoutCounter = new CheckoutCounter(new Mobile());
$CheckoutCounter->pay();
```
### 類別圖

![](https://i.imgur.com/GBOANzW.png)

#### plant uml

```
@startuml

skinparam classAttributeIconSize 0

interface IPayment{
{method} + pay():string
}
class Cash {
{method} + pay():string
}

class Card {
{method} + pay():string
}

class Mobile {
{method} + pay():string
}

class CheckoutCounter {
{field} - IPayment $Payment
{method} + pay():string
}

CheckoutCounter .down.> IPayment : Abstract Dependent
Cash .up.|> IPayment : Implements Interface
Card .up.|> IPayment : Implements Interface
Mobile .up.|> IPayment : Implements Interface

@enduml
```

我們新增了一個IPayment介面，讓這個介面去規定Cash、Card、Mobile三種付款方式應該具有的行為，而現在不管甚麼樣的類別，只要該類別有去實作IPayment介面，則他就可以被CheckoutCounter使用，這個地方我們的依賴關係就由依賴實體的類別轉化成去依賴IPayment介面，這個就是所謂的"依賴抽象"。


再來我們拿上面的估算方法去測量一下新的寫法的依賴測量分數

|                 | CheckoutCounter | Cash | Card | Mobile | IPayment |
| ----------------|-----------------|------|------|--------|----------|
| CheckoutCounter | -               | 0    | 0    | 0      | 1        |
| Cash            | 0               | -    | 0    | 0      | 1        |
| Card            | 0               | 0    | -    | 0      | 1        |
| Mobile          | 0               | 0    | 0    | -      | 1        |
| IPayment        | 1               | 1    | 1    | 1      | -        |

在新的寫法中我們獲得的依賴測量分數為(8/5)=1.6，降低了耦合的程度

從比較重構前後的類別圖來看，其實我們可以觀察到我們的依賴關係出現了一點變化，從單方面的高階模組依賴低階模組變成無論高階或低階的模組都依賴於介面(抽象)，對於高階模組來說她依賴的是低階模組的介面(抽象)，而對於低階模組來說也是只要依賴於自己的介面(抽象)就可以。並且如果今天要再新增另外的付款方式，我只要新增一個類別去實作IPayment的介面，即可很輕易的被CheckoutCounter帶入去做使用，在系統內部解決了開放封閉原則的問題。

在簡化過的類別圖上他的轉變會像是這樣

### 類別圖

![](https://i.imgur.com/LGI8bnE.png)

#### plant uml

```
@startuml

skinparam classAttributeIconSize 0

package before{
class Service1
class Component1
Service1 .down.> Component1 : Concrete Dependent
}

package after{
class Service2
class Component2
interface IComponent2
Service2 .down.> IComponent2 : Abstract Dependent
Component2 .up.|> IComponent2 : Implements Interface
}

@enduml
```

由上面的變化可以看出，我們本來從單方向的依賴，轉為去依賴某種介面，而這種依賴關係的轉向，正式DIP中的Inversion(反轉)所想表達的意思。


## Inversion of Control - 控制反轉

接下來介紹常常會跟DIP一起出現的一個原則，
IoC - Inversion of Control(控制反轉)

[Wiki](https://en.wikipedia.org/wiki/Inversion_of_control)
> In software engineering, inversion of control (IoC) is a programming principle. IoC inverts the flow of control as compared to traditional control flow. In IoC, custom-written portions of a computer program receive the flow of control from a generic framework.
> > 軟體工程中，控制反轉是一種程式撰寫原則。相對於傳統的控制流程來說，IoC反轉了整個控制的流向。在IoC中，程式自定義的編寫部分會被動的接收來自框架的控制流。

## 控制"反轉"? 反轉什麼?

由上面的例子來說，可以看到原本的付款服務都是由收銀台去決定去控制，對比新版本的程式來說，新版本的程式將決定跟控制的權力交到使用者手中，這種控制權的移交，將控制由系統內部往外拉，從而由外部去做控制，這樣的控制權的轉移也就是控制反轉的"反轉"所代表的意思，上面的例子其實已經也同時實現了IoC。

這種方式很好的實現了["好萊塢守則"](https://en.wiktionary.org/wiki/Hollywood_principle)

> Don't call us, we'll call you.
> > 別找我們，我們找你。

## 幾種實踐IOC的方式

DIP與IoC常常相輔相成的成對出現，當我將依賴轉化為抽象的時候，常常會伴隨著我必須要將管理依賴的權責移交出去。
轉化為抽象的時候實現DIP，而將管理權責移交出去的時候則實現Ioc。

然而，為了達成鬆散耦合的目的，實現這些原則，古聖先賢們其實研發了不少種模式可以套用及實踐。

1. 四人幫歸類的Creational Pattern
2. Dependency Injection(依賴注入)
3. Service Locator
4. Dependency Lookup
5. ...待補充

