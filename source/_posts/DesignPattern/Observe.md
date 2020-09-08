---
title: Design Patterns - Behavioral - Observer Pattern 觀察者模式
date: 2020-08-5 17:34:45
tags:
- Design Patterns
- Behavioral Patterns
categories: 
- 讀書會報告
cover: https://picsum.photos/id/177/800/600
---

# 設計模式 - Observer Pattern 觀察者模式

## Intro

假設youtube頻道需要很多種通知機制(如：訂閱、鈴鐺、寄信)，而在頻道新增影片時我們必須通知我們的使用者。

## 實作

首先，我們先新增YoutubeChannel類別，因為我們希望在新增影片的時候可以通知我們的觀眾，所以我在類別中給訂了一個private變數，希望能夠將我需要通知的對象給儲存進去。

```php=

class YoutubeChannel
{
    private $Notifications = [];

    public function attach($Notification)
    {
        $this->Notifications[] = $Notification;
    }

    public function detach($Notification)
    {
        foreach ($this->Notifications as $key => $notification) {
            if ($notification->getNotificationId() == $Notification->getNotificationId()) {
                array_splice($this->Notifications, $key, 1);
            }
        }
    }

    public function getNotifications(): array
    {
        return $this->Notifications;
    }

    public function notify(): array
    {
        $notification_console = [];
        foreach ($this->Notifications as $notification) {
            $notification_console[] = $notification->update();
        }
        return $notification_console;
    }
}


```

再來是我們的通知類別，各種不一樣的通知都需要實作不同的通知方法，所以這邊都預留了一個update()給YoutubeChannel類別使用

```php=

class Mail
{
    private $NotificationId;

    public function __construct(int $AccountId)
    {
        $this->setNotificationId($AccountId);
    }

    public function getNotificationId(): int
    {
        return $this->NotificationId;
    }

    public function setNotificationId($NotificationId)
    {
        $this->NotificationId = $NotificationId;
    }

    public function update(): string
    {
        return 'NotificationID=' . $this->getNotificationId() . '+信件通知';
    }
}

class Bells
{
    private $NotificationId;

    public function __construct(int $AccountId)
    {
        $this->setNotificationId($AccountId);
    }

    public function getNotificationId(): int
    {
        return $this->NotificationId;
    }

    public function setNotificationId($NotificationId)
    {
        $this->NotificationId = $NotificationId;
    }

    public function update(): string
    {
        return 'NotificationID=' . $this->getNotificationId() . '+鈴鐺通知';
    }
}

```

當我們實際上在使用的時候，會像下面這種方式

```php=

$SomeOnesChannel = new YoutubeChannel();
$Bells = new Bells(1);
$Mail = new Mail(2);
$SomeOnesChannel->attach($Bells);
$SomeOnesChannel->attach($Mail);
$Notifications = $SomeOnesChannel->notify();

```

### 類別圖

而當我們去關心這個通知機制的類別圖我們會發現，YoutubeChannel類別與通知類別處於一種基於實體類別的強依賴狀態。

![](https://i.imgur.com/zsTpCOe.png)

#### Plant UML

```=
@startuml

skinparam classAttributeIconSize 0

class Mail{
{field} - NotificationId : int
{method} + __construct(int $AccountId) : void
{method} + getNotificationId() : int
{method} + setNotificationId() : void
{method} + update() : void
}

class Bells{
{field} - NotificationId : int
{method} + __construct(int $AccountId) : void
{method} + getNotificationId() : int
{method} + setNotificationId() : void
{method} + update() : void
}

class YoutubeChannel{
{field} - Notifications : array
{method} + attach($Notification) : void
{method} + detach($Notification) : void
{method} + notify() : array
{method} + getNotifications() : array

}

YoutubeChannel -down-> Mail : notify(update)
YoutubeChannel -down-> Bells : notify(update)

@enduml
```

為了將耦合鬆散化，首先我們將YoutubeChannel類別中的attach()、detach()、notify()提取成interface，如此一來要是我有別的服務(例如：twitch)擁有類似的機制，我便可以透過實作這個interface來確保我的服務也擁有相似的行為

> 另外我將attach()、detach()的輸入參數限制為底下會提到的IObserver介面

```php=

interface ISubject
{
    public function attach(IObserver $Notifications);

    public function detach(IObserver $Notification);

    public function notify(): array;
}

class YoutubeChannel implements ISubject
{
    private $Notifications = [];

    public function attach(IObserver $Notification)
    {
        $this->Notifications[] = $Notification;
    }

    public function detach(IObserver $Notification)
    {
        foreach ($this->Notifications as $key => $notification) {
            if ($notification->getNotificationId() == $Notification->getNotificationId()) {
                array_splice($this->Notifications, $key, 1);
            }
        }
    }

    public function getNotifications(): array
    {
        return $this->Notifications;
    }

    public function notify(): array
    {
        $notification_console = [];
        foreach ($this->Notifications as $notification) {
            $notification_console[] = $notification->update();
        }
        return $notification_console;
    }
}


```

再來我們希望我們的被通知者都有相同的接口可以讓社群媒體通知，所以我們將update()也提取成介面，如此一來要是我有新增了別的通知方式(例如：Line)，我只要實作這個interface就可以確保我的通知類別可以跟YoutubeChannel類別或是其他的社群媒體類別能夠安心的配合

```php=

interface IObserver
{
    public function update(): string;
}

class Mail implements IObserver
{
    private $NotificationId;

    public function __construct(int $AccountId)
    {
        $this->setNotificationId($AccountId);
    }

    public function getNotificationId(): int
    {
        return $this->NotificationId;
    }

    public function setNotificationId($NotificationId)
    {
        $this->NotificationId = $NotificationId;
    }

    public function update(): string
    {
        return 'NotificationID=' . $this->getNotificationId() . '+信件通知';
    }
}

class Bells implements IObserver
{
    private $NotificationId;

    public function __construct(int $AccountId)
    {
        $this->setNotificationId($AccountId);
    }

    public function getNotificationId(): int
    {
        return $this->NotificationId;
    }

    public function setNotificationId($NotificationId)
    {
        $this->NotificationId = $NotificationId;
    }

    public function update(): string
    {
        return 'NotificationID=' . $this->getNotificationId() . '+鈴鐺通知';
    }
}

```

### 類別圖

透過這種方式重新整理之後，我們可以看到我們的類別關係有了改變，實體的相依變成以interface為基底去做依賴，我們通知者與被通知者之間的依賴給抽象化了

![](https://i.imgur.com/PqutOwb.png)

#### Plant UML

```=
@startuml

skinparam classAttributeIconSize 0

interface IObserver{
{method} + update() : void
}

class Mail{
{field} - NotificationId : int
{method} + __construct(int $AccountId) : void
{method} + getNotificationId() : int
{method} + setNotificationId() : void
{method} + update() : void
}

class Bells{
{field} - NotificationId : int
{method} + __construct(int $AccountId) : void
{method} + getNotificationId() : int
{method} + setNotificationId() : void
{method} + update() : void
}

interface ISubject{
{method} + attach(IObserver $Notification) : void
{method} + detach(IObserver $Notification) : void
{method} + notify() : array
}

class YoutubeChannel{
{field} - Notifications : array
{method} + attach(IObserver $Notification) : void
{method} + detach(IObserver $Notification) : void
{method} + notify() : array
{method} + getNotifications() : array
}

Bells .up.|> IObserver
Mail .up.|> IObserver
YoutubeChannel .up.|> ISubject

ISubject -right-> IObserver : notify(update)

@enduml
```

透過提取ISubject與IObserver，不只將依賴從實體化為抽象，將強耦合變為弱耦合，我們還確保了通知者與被通知者的行為架構，此種設計方式我們稱之為「觀察者模式」

## 時機

- 想要在一個物件的狀態改變時通知很多相關物件的時候
- 某一個物件有變化時，對應物件也得跟著改變，但事先又不知道對應的物件要做的改變有多少時
- 當物件必須能夠通知其他的物件，但又不能假設後者是誰

## 目的

定義一種一對多的物件依賴關係，讓該物件的狀態一有變動，就自動通知其他相依物件做出該做的更新行為。

## 類別圖

![](https://i.imgur.com/hwLF7wQ.png)


### Plant UML

```=
@startuml

skinparam classAttributeIconSize 0

interface IObserver{
{method} + update()
}
note right of IObserver : ISubscribe


interface ISubject{
{method} + attach(IObserver Observer)
{method} + detach(IObserver Observer)
{method} + notify()
}
note left of ISubject : IPublish

class Observer{
{method} + update()
}
note bottom of Observer : Subscriber


class Subject{
{method} + attach()
{method} + detach()
{method} + notify()
}
note bottom of Subject : Publisher

ISubject -right-> IObserver : Notify(update)
Subject .up.|> ISubject
Observer .up.|> IObserver
@enduml
```

## 優點

- 這樣的類別關係遵守開放封閉原則，加入新的訂閱者並不會修改到發布者(反之亦然)
- 可以在程式運行的時候才動態的去構築類別間的依賴關係，且隨時可以解除
- Observer與Subject之間的耦合為抽象的、較微弱

## 缺點

- 接收過多額外通知
- 由於Observer互相獨立，如果我們要改變Subject的話，有可能因為改變而影響到數個Observer以及其相依的動作，而導致錯誤難以追查

## 實際案例

### PHP

事實上，在PHP的Library中有提供一組可以實現Observer Pattern的Interface

[SplSubject](https://www.php.net/manual/en/class.splsubject.php) 
[SplObserver](https://www.php.net/manual/en/class.splobserver.php) 
[在Stackoverflow上有人發問了這兩個介面的使用時機](https://stackoverflow.com/questions/13774149/how-is-splsubject-splobserver-useful)

### Angular with RxJS 

Angular作為前端框架，在更新頁面資料上使用了觀察者模式

我可以建立一個Observable的實例(也就是上面提到的Subject)，在Observable實例中，定義了一個subscribe()函式(也就是上面提到的attach())，而subscribe()只能傳入符合Observer介面的參數(也就是上面提到的IObserver)

底下的範例為Angular在接API的時候使用RxJS去實作觀察者模式，可以讓response回來之後自動的去渲染

* Reference
    * [使用可觀察物件（Observable）來傳遞值](https://angular.tw/guide/observables)
    * [RxJS 函式庫](https://angular.tw/guide/rx-library)

```typescript=

import { from } from 'rxjs';

// Create an "Observable" out of a promise
const data = from(fetch('/api/endpoint'));

// Subscribe to begin listening for async result
data.subscribe({
  //可在next()內寫更新資料來達成自動渲染畫面
  next(response) { console.log(response); }, 
  error(err) { console.error('Error: ' + err); },
  complete() { console.log('Completed'); }
});

```

### 類別圖

其架構的類別圖大概長得像式這樣(僅示意)

![](https://i.imgur.com/LpINzzT.png)


#### Plant UML

```
@startuml

skinparam classAttributeIconSize 0

class Observable{
{method} + subscribe(Observer observer)
{method} + SomeFunctionToCall_next()
}

class observer{
{method} + next()
{method} + error()
{method} + complete()
}

interface Observer{
{method} + next()
{method} + error()
{method} + complete()
}

Observable -right-> Observer : call next() to update
observer .up.|> Observer : implements

@enduml

```__
