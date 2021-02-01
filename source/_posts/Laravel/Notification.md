---
title: Laravel Notifications - 以 Slack 為例子
date: 2021-01-07 10:35:41
tags:
- PHP
- Laravel
- Notification
categories:
- Laravel
cover: https://picsum.photos/id/58/800/600
---

# Laravel Notifications - 以 Slack 為例子

故事是這樣的，最近筆者在研究如何擷取 RSS 內的資訊，並將最新資訊自動通知到 Slack 上，想寫一個訂閱 RSS 的機器人這樣的功能，於是乎在找資料的時候輾轉找到了 Laravel Notification 的 feature ，由於之前完全沒有使用過，所以就研究了一番，最後也真的有實作在自動通知的機器人的 code 裡面，以下來記錄一下學習的心得。

## 先來看看官方文件吧！

在此先奉上[官方文件](https://laravel.com/docs/8.x/notifications)

先跟著官方文件看一看這功能到底在幹嘛吧，首先官方告訴你 Notification 大致上就是可以讓你用來串接與通知有關的一些功能，內建也支援了一些 driver，例如： email, SMS, Slack... 之類的，接著開始告訴你該如何一步一步建構出這樣的功能

首先我們需要一個 Laravel 幫我們寫好的 Notification Class，於是我們一樣老方法呼叫 Artisan 幫我們把 Notification Class 建構出來

```
php artisan make:notification HelloNotification
```

接著我們可以在 ``` app/Notifications ``` 裡面發現我們剛剛建構出來的 ``` HelloNotification.php ``` 模板

**HelloNotification.php**
```php=
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Notification;

class HelloNotification extends Notification
{
    use Queueable;

    /**
     * Create a new notification instance.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }

    /**
     * Get the notification's delivery channels.
     *
     * @param  mixed  $notifiable
     * @return array
     */
    public function via($notifiable)
    {
        return ['mail'];
    }

    /**
     * Get the mail representation of the notification.
     *
     * @param  mixed  $notifiable
     * @return \Illuminate\Notifications\Messages\MailMessage
     */
    public function toMail($notifiable)
    {
        return (new MailMessage)
                    ->line('The introduction to the notification.')
                    ->action('Notification Action', url('/'))
                    ->line('Thank you for using our application!');
    }

    /**
     * Get the array representation of the notification.
     *
     * @param  mixed  $notifiable
     * @return array
     */
    public function toArray($notifiable)
    {
        return [
            //
        ];
    }
}

```

稍微來導覽一下這個 Notification Class 內到底在幹嘛吧

首先 ``` __construct() ``` 的部分就是讓你將要用到的東西丟進來或是注入會使用到的 Service 的

接著看到 ``` via(notifiable) ```

```php=
public function via($notifiable)
{
    return ['mail'];
}
```

這個部分所回傳的 array element ，會對應到你要發送的通知媒介，像是這樣他會對應到 mail

怎麼對應呢？當 Laravel 會去歷遍 ``` via($notifiable) ``` 所回傳的 array 裡面所有的值，假設當他歷遍到 ``` 'mail' ``` 的時候，Laravel 會去找底下的 ``` toMail($notifiable) ``` 去執行，也就是說假設我今天在 ``` via($notifiable) ```  回傳了 ```  ['mail', 'slack'] ``` ，則我底下必須要有兩個 function ，一個是 ``` toMail($notifiable) ``` ，一個是 ``` toSlack($notifiable) ``` ，就像下面這樣

```php=
 public function via($notifiable)
    {
        return ['mail', 'slack'];
    }

 
    public function toMail($notifiable)
    {
        
    }
    
    public function toSlack($notifiable)
    {
     
    }
```

至於要怎麼呼叫他讓他可以傳送通知呢？

這個部分官網用了 User Model 去做範例，我們在 ``` app/Models/User.php ``` 裡面可以看到他比我們另外新增的 Model 多使用了一個 Notifiable Trait，而這個 Trait 就是讓整個流程運行起來的關鍵

**app/Models/User.php**

```php=
<?php

namespace App\Models;

use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use HasFactory, Notifiable;

    // 下略
}

```

這樣的狀況下，我們在使用 User Model 的時候就可以用下面的方式呼叫通知並送出通知到對應的對象（email、slack）

```php=
use App\Notifications\HelloNotification;

$user = User::find(1);
$user->notify(new HelloNotification());
```

## 等等，怎麼莫名其妙就發送通知了？

首先閒聊一下，我自己在看 Laravel 文件的時候總有種文件寫的太簡潔讓我不太能夠了解用法的感覺，或許是我悟性不夠或者是能力不足吧，我有好一段時間一直在思考，這個功能是不是只能跟 Model 合作呢？畢竟我在網路上搜尋到的一些教學跟範例大家都是這麼幹的，但這很不合理啊，不只是我對資料表變動了才有契機去發送通知的吧？就像我想到要做的功能一樣，我只是去看 RSS 有沒有新的貼文，有的話就傳送，或許過程根本不見得會經過資料庫的判斷，我只能保證有一個邏輯判斷為基準去確定要不要發送通知，而這個判斷基準不一定是依賴在 Model 上

所以對我來說，最好的方法就是搞清楚這個機制背後是在玩什麼把戲，所以接下來我們先不實作功能，我們先來看看我們設定在 Model 中的 Notifiable Trait 吧，既然他令 Model 變得如此特別，那表示他一定有它的魔力在

讓我們往回追到 Notifiable Trait 的源頭

**vendor/laravel/framework/src/Illuminate/Notifications/Notifiable.php**

```php=
<?php

namespace Illuminate\Notifications;

trait Notifiable
{
    use HasDatabaseNotifications, RoutesNotifications;
}
```

我們可以看到 Notifiable Trait 的本體其實封裝了兩個另外的 Trait

1. HasDatabaseNotifications
2. RoutesNotifications

HasDatabaseNotifications 是當我們的 Notification Driver 設定為 Database 的時候才會用到的部分，所以這邊我們先跳過，讓我們把焦點放在 RoutesNotifications 上

首先我們來看看這支 Trait 裡面有什麼內容吧

**vendor/laravel/framework/src/Illuminate/Notifications/RoutesNotifications.php**

```php=
<?php

namespace Illuminate\Notifications;

use Illuminate\Contracts\Notifications\Dispatcher;
use Illuminate\Support\Str;

trait RoutesNotifications
{
    /**
     * Send the given notification.
     *
     * @param  mixed  $instance
     * @return void
     */
    public function notify($instance)
    {
        app(Dispatcher::class)->send($this, $instance);
    }

    /**
     * Send the given notification immediately.
     *
     * @param  mixed  $instance
     * @param  array|null  $channels
     * @return void
     */
    public function notifyNow($instance, array $channels = null)
    {
        app(Dispatcher::class)->sendNow($this, $instance, $channels);
    }

    /**
     * Get the notification routing information for the given driver.
     *
     * @param  string  $driver
     * @param  \Illuminate\Notifications\Notification|null  $notification
     * @return mixed
     */
    public function routeNotificationFor($driver, $notification = null)
    {
        if (method_exists($this, $method = 'routeNotificationFor'.Str::studly($driver))) {
            return $this->{$method}($notification);
        }

        switch ($driver) {
            case 'database':
                return $this->notifications();
            case 'mail':
                return $this->email;
        }
    }
}
```

還記得我們是怎麼使用帶有 Notifiable Trait 的 Model 嗎？

```
$user->notify(new HelloNotification());
```

User Model 使用了 Notifiable Trait 裡面的 notify() 對吧！
所以我們接下來將焦點放在這個 function 上

```php=
/**
 * Send the given notification.
 *
 * @param  mixed  $instance
 * @return void
 */
public function notify($instance)
{
    app(Dispatcher::class)->send($this, $instance);
}
```

可以觀察到當我們呼叫這個 function 的時候我們是把 ``` new HelloNotification() ``` 給帶入，所以在執行這個 ```notify()``` 的時候其實是執行了下面的內容

```
app(Dispatcher::class)->send($this, (new HelloNotification()));
```

``` $this ``` 指的就是我們原本的 Class ，像是這邊就是指 ```User``` ，我們可以大致上感覺到當我們呼叫 ```notify()``` 的時候，他會將 User 本身以及 HelloNotification 交給一個 Dispatcher class 讓它將通知給發送出去。可以注意到的是這邊的 ```$this``` 參數並沒有限制一定要是 Model 型別才能丟進來。

> 繼續往 source code 裡面去追其實還有一大堆過程在裡面
> 包括生成 Dispatcher 、 擷取要配送的目標 、 trigger 配送方法(toMail 、 toSalck)、對應配送的 Driver 生成、實際配送...之類的
> 深刻討論下去篇幅會拉得相當長，這邊就暫時先不討論

這個時候我開始思考，是不是隨便一個 Class 套用了 Notifiable Trait 之後都可以如期運作呢？或許他不一定要使用 Model 才能用

答案是可以的，我可以生成任意的一個 Class 然後讓它帶有 Notifiable ，在 Notification Class 運作都正常的狀況下，他便可以如期發送通知

```php=
class AnyClass
{
    use Notifiable;
}

$AnyClass = new AnyClass();
$AnyClass->notify(new HelloNotification());
```

於是我可以確定，我能夠在任何的動作前提下去執行發送 Notification 的動作，所以接下來我們來稍微簡單做一下我原本想做的功能吧

## 來做個 Slack 通知吧！

首先我們把最重要的 Notification Class 先做出來，先用 Artisan 指令把 Notification Class 的模板做出來

```
php artisan make:notification SlackDemoNotification
```

再來我們將要傳送 Slack 訊息的部分給修改一下

**SlackDemoNotification.php**
```php=
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Notifications\Messages\SlackMessage;
use Illuminate\Notifications\Notification;

class SlackDemoNotification extends Notification
{
    use Queueable;

    private $message;

    public function __construct(string $message)
    {
        $this->message = $message;
    }

    public function via($notifiable)
    {
        return ['slack'];
    }

    public function toSlack($notifiable)
    {
        return (new SlackMessage)
            ->content($this->message);
    }

}

```

最後我們建構一個主要負責去判斷該不該送出訊息的 Class

**RssFeederTrigger.php**
```php=

namespace App;

use Illuminate\Notifications\Notifiable;

class RssFeederTrigger
{
    use Notifiable;

    public function isRssHasNewMessage(): bool
    {
        /**
         * 這邊去偵測了 RSS 有沒有新的訊息
         * 假設有
         * 所以我直接回傳 true
         */
        return true;
    }

    public function getMessageContent(): string
    {
        return '這是一則新的 slack 訊息';
    }

    public function routeNotificationForSlack($notification)
    {
        return 'https://hooks.slack.com/services/...';
    }
}
```

這邊的 ``` routeNotificationForSlack($notification) ``` 是針對傳送 Slack 訊息要多加的一個特殊 function ，用意在設定 slack api 的 webhook

最後，我們在程式的某個地去呼叫

```php=

$RssFeederTrigger = new RssFeederTrigger();
if ($RssFeederTrigger->isRssHasNewMessage()) { // always true
    $RssFeederTrigger->notify(
        new SlackDemoNotification($RssFeederTrigger->getMessageContent())
    );
}
```

最後我們就可以在 slack 的 channel 裡面看到

![](https://i.imgur.com/406OsOS.png)

## Reference

- https://laravel.com/docs/8.x/notifications
