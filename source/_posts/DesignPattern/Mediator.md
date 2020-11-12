---
title: Design Patterns - Behavioral - Mediator Pattern 仲介者模式
date: 2020-09-23 16:33:15
tags:
- Design Patterns
- Behavioral Patterns
categories: 
- 讀書會報告
cover: https://picsum.photos/id/249/800/600
---

# 設計模式 - Mediator Pattern 仲介者模式

## Intro

當很多物件互相需要協作的時候，物件間的耦合關係常常會變得比較混亂
假設我們有三個國家的人在共同會議上需要傳訊息，但因為語言的隔閡，他們只能私訊另外兩個國家的人，並且貼上他們翻譯過後的文字，他們的互動關係會像下面這樣

![](https://i.imgur.com/VBat4ev.png)

但是如果今天有一個相當厲害的翻譯人員協助的話，我們就只要將自己的話告訴翻譯人員，他就會幫我們翻譯並且轉述給另外兩個國家的人了，加入了翻譯人員的互動關係會像下面這樣

![](https://i.imgur.com/PidI6vP.png)

讓某個腳色在中間為各種物件做協調，這就是仲介者模式的初始的想法

## 實作

首先我們新增中介者的介面Mediator
我們希望他可以扮演傳訊息給其他使用者的腳色，所以制定了一個sendToOther方法

```php=

interface Mediator
{
    public function sendToOther(string $country_user, string $message): array;
}

```

再來我們定義一個抽象類別Colleague，這個腳色他必須知道要把訊息傳給誰，讓誰來擔任中介者，所以制定了一個Mediator參數及setter function
另外給繼承Colleague的類別公定的介面，每個聊天室的成員都必須要可以送出跟接收訊息，所以定義了兩個抽象函式send & receive

```php=

abstract class Colleague
{
    /**
     * @var Mediator $Mediator
     */
    protected $Mediator;

    abstract public function send(string $message);

    abstract public function receive(string $message): string;

    public function setMediator(Mediator $mediator)
    {
        $this->Mediator = $mediator;
    }
}

```

再來我們新增聊天室的成員，有美國人、台灣人、日本人，將這些類別繼承我們的Colleague

```php=

class American extends Colleague
{
    public function send(string $message)
    {
        $this->Mediator->sendToOther('American',$message);
    }

    public function receive(string $message): string
    {
        return 'American : '.$message;
    }
}

class Taiwanese extends Colleague
{
    public function send(string $message)
    {
        $this->Mediator->sendToOther('Taiwanese',$message);
    }

    public function receive(string $message): string
    {
        return 'Taiwanese : '.$message;
    }
}

class Japaneses extends Colleague
{
    public function send(string $message)
    {
        $this->Mediator->sendToOther('Japaneses', $message);
    }

    public function receive(string $message): string
    {
        return 'Japanese : ' . $message;
    }
}

```

最後我們新增我們的聊天室本體，讓他去實作Mediator的介面，成為中介者
在ChatRoom創建的時候，我們必須知道我們是為了哪些人去做居中協調的動作，所以我比須將這些Colleague的子類別給輸入進來
並且，中介者還須要讓每個人都知道有任何問題的時候請找仲介者來做訊息的轉交，所以在這邊我們要對每個Colleague的子類別去設定他們應該要關注的中介者
然後我們就可以去實作居中協調的功能(sendToOther)本身了，讓他對應各種狀況去做出對應的動作

```php=

class ChatRoom implements Mediator
{
    private $American;
    private $Taiwanese;
    private $Japanese;

    public function __construct(American $American, Taiwanese $Taiwanese, Japaneses $Japanese)
    {
        $this->American = $American;
        $this->Taiwanese = $Taiwanese;
        $this->Japanese = $Japanese;

        $this->American->setMediator($this);
        $this->Taiwanese->setMediator($this);
        $this->Japanese->setMediator($this);
    }

    public function sendToOther(string $country_user, string $message): array
    {
        $send_messages = [];
        switch ($country_user) {
            case 'American':
                $send_messages[] = $this->Taiwanese->receive($message);
                $send_messages[] = $this->Japanese->receive($message);
                break;
            case 'Taiwanese':
                $send_messages[] = $this->American->receive($message);
                $send_messages[] = $this->Japanese->receive($message);
                break;
            case 'Japaneses':
                $send_messages[] = $this->American->receive($message);
                $send_messages[] = $this->Taiwanese->receive($message);
                break;
            case 'All':
            default:
            $send_messages[] = $this->American->receive($message);
            $send_messages[] = $this->Taiwanese->receive($message);
            $send_messages[] = $this->Japanese->receive($message);
                break;
        }
        return $send_messages;
    }
}

```

使用方式如下

```php=
//首先把聊天室及參與者都具體化
$Mediator = new ChatRoom(new American(), new Taiwanese(), new Japaneses());

//假設傳訊息的訊息內容本身就叫做message
$message = 'message';

/*美國人傳給其他人**/
$Mediator->sendToOther('American', $assert_message);
// [
//     'Taiwanese : message',
//     'Japanese : message',
// ];

/*台灣人傳給其他人**/
$Mediator->sendToOther('Taiwanese', $assert_message);
// [
//     'American : message',
//     'Japanese : message',
// ];

/*日本人傳給其他人**/
$Mediator->sendToOther('Japaneses', $assert_message);
// [
//     'American : message',
//     'Taiwanese : message',
// ];

/*其實系統管理員也可以傳給大家**/
$Mediator->sendToOther('All', $assert_message);
// [
//     'American : message',
//     'Taiwanese : message',
//     'Japanese : message',
// ];
```

## 時機

- 類別之間互相的聯繫過多時
- 由於引用關係複雜導致類別難以重用或者難以更改時
- 
## 目的

用一個仲介物件來封裝一系列的物件互動。仲介物件使各物件不需要顯式的互相參考，從而使各物件的耦合鬆散化，而且可以獨立改變物件之間的互動。

## 類別圖

![](https://i.imgur.com/kpgeCSP.png)

### Plant UML

```
@startuml

skinparam classAttributeIconSize 0

together {
 abstract class Colleague{
 {field} - $Mediator : Mediator
 {method} + Action()
 }

 class ConcreteColleague1{
 {method} + Action()
 }
 class ConcreteColleague2{
 {method} + Action()
 }
}

together {
 interface Mediator{
 {method} + Action()
 }

 class ConcreteMediator{
 {method} + Action()
 }
}

ConcreteMediator .up.|> Mediator
ConcreteColleague1 -up-|> Colleague
ConcreteColleague2 -up-|> Colleague
ConcreteMediator -left-> ConcreteColleague1
ConcreteMediator -left-> ConcreteColleague2
Colleague .right.> Mediator

@enduml

```

## 優點

- 可以將Colleague之間的關係解耦，可一併達到對於Colleague物件更好的重用率
- 對於新增需要協同多個Colleague物件的功能，我們不必去改變Colleague的對外行為，可透過新增Mediator達成
- 物件關係由多對多轉變為一對多，這樣的關係更容易理解
- 物件協作方式的抽象化

## 缺點

- Mediator職責太重的話可能會成為上帝的物件([God Object](https://zh.wikipedia.org/wiki/%E4%B8%8A%E5%B8%9D%E5%AF%B9%E8%B1%A1))
