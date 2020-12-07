---
title: 晚點再說啦三部曲之二 - Laravel Artisan Console - 基礎篇
date: 2020-12-03 16:13:39
tags:
- PHP
- Laravel
- Artisan
- Command
categories: 
- Laravel
cover: https://picsum.photos/id/284/800/600
---

# 晚點再說啦三部曲之二 - Laravel Artisan Console - 基礎篇

本文章使用的開發環境為

PHP : 7.3.11
Laravel : 8.16.1
Redis : 6.0
MySQL : 8.0

---

## 前言

在開發的過程中，「如果我想要在特定的時間點做某些事情呢？」這種需求也相當常見，這個所謂的「某些事情」工作量可能非常龐大，很自然的我們可以想到，「要是我可以在某個時間點觸發某個動作，然後讓他把對應的工作放到某個地方慢慢消化就好了」，而「把對應的工作放到某個地方慢慢消化」這就是我們上一篇的主題，接下來我們要專注於，「如何在某個時間點觸發某個動作」

而這篇主要著重於如何處理「觸發某個動作」，而下一篇將要探討「如何在某個時間點觸發」

講到要「觸發某個動作」， Web 的開發者會很自然的想到要去寫一個可以瀏覽的網址去當作觸發的媒介，當我們瀏覽這個網址一次，就觸發對應的動作，這看起來相當合理也很直覺，但視情況而言，這種方式可能會成為一個不是很好的解決方式。

舉個例子來說吧，電商網站常常會有寄送廣告信件的功能，那們我如果將「寄送廣告給所有使用者」這個動作寫成一個網址，再讓知道網址的人瀏覽去觸發，這樣真的好嗎？

首先，假設該電商公司是一間血汗公司，離職員工懷恨在心，於是寫了一支程式每秒去瀏覽這個神奇的寄信用網址，可想而知的會發生以下的事情

1. 大量的信件被寄出， Mail Server 的計費直達天際
2. 客戶收到很多重複的信件，客戶觀感會很差
3. 這些重複的信件會被某些信箱視為垃圾郵件，導致發送信件的來源信任度下降

那麼，應該怎麼辦呢？我們換個想法，讓「寄送廣告給所有使用者」的觸發媒介改為在 Server 上輸入特定指令如何？這可以確保只有公司內部人士可以使用，也可以對主機的使用者做權限控制，只讓有這個權責的工程師才有辦法操作，不會被超量寄出信件，更不會讓客戶收到很多重複的信件。

所以本篇的主角登場了，我們該如何在 Laravel 上寫一個指令呢？ Laravel 本身提供了一系列相當好用的 Artisan 指令，同時他也支援我們可以去新增自己想要的指令，開放了一些介面可以讓我們達到高度客製化自己的指令

## 建立第一個 Artisan Command

首先來建立第一個 Artisan Command 吧！我們只要透過下面的指令便可以輕鬆的新增一個 Laravel 幫你包裝好的指令物件（```SendEDMCommand```）到 ```app\Console\Commands```資料夾

```
php artisan make:command SendEDMCommand
```

然後我們就得到了一個 Laravel 幫我們定義好的空白 Artisan Command

```php=
namespace App\Console\Commands;

use Illuminate\Console\Command;

class SendEDMCommand extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'command:name';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Command description';

    /**
     * Create a new command instance.
     *
     * @return void
     */
    public function __construct()
    {
        parent::__construct();
    }

    /**
     * Execute the console command.
     *
     * @return int
     */
    public function handle()
    {
        return 0;
    }
}
```

接下來我們來介紹這個類別

**signature**

簡單的來說這個參數就是讓你為你的 Artisan Command 命名，及給他一些設定，舉個例子來說我們可以將這個參數設置為下面這樣

```
$signature = 'edm:send';
```

之後我們便可以透過```php artisan edm:send```去寄發所有的 EDM 信件

**description**

就是該 Artisan Command 的描述，我們看一將他設定為下面這樣

```
$description = '寄送 EDM 給所有的使用者';
```

**handle()**

這邊就是主角的部分了，我想要執行的工作內容都會寫在 ```handle()``` 這個 function 裡面，舉例來說像是這樣

```php=
public function handle()
{
    echo "寄送 EDM 給所有的使用者\n";
}
```

接下來我們只要將這個 Artisan Command 註冊，接著就可以開始使用了

## 註冊 Artisan Command

要如何註冊呢？我們找到 ```app\Console\Kernel.php``` 

```php=
// app\Console\Kernel.php

namespace App\Console;

use Illuminate\Console\Scheduling\Schedule;
use Illuminate\Foundation\Console\Kernel as ConsoleKernel;

class Kernel extends ConsoleKernel
{
    /**
     * The Artisan commands provided by your application.
     *
     * @var array
     */
    protected $commands = [
        //
    ];

    /**
     * Define the application's command schedule.
     *
     * @param  \Illuminate\Console\Scheduling\Schedule  $schedule
     * @return void
     */
    protected function schedule(Schedule $schedule)
    {
        // $schedule->command('inspire')->hourly();
    }

    /**
     * Register the commands for the application.
     *
     * @return void
     */
    protected function commands()
    {
        $this->load(__DIR__.'/Commands');

        require base_path('routes/console.php');
    }
}
```

接著我們將 ```$commands``` 的 array 加入我們的 Artisan Command 類別，就算是註冊完成了，如下

```php=
protected $commands = [
    SendEDMCommand::class
];
```

接著當我們將所有的指令列出來的時候就會看到我們剛剛寫的 Artisan Command 了

![](https://i.imgur.com/1Uxpo8X.jpg)

 ## 執行
 
 註冊完的 Artisan Command 有兩種方式可以執行，一種是直接在黑窗裡面下指令，除此之外， Laravel 也有提供讓我們可以在程式內去呼叫指令
 
 ### 打指令執行
 
 就像我們一般在下 Artisan 指令一樣，我們只要輸入 ```php artisan edm:send``` 就可以執行了，如圖
 
 ![](https://i.imgur.com/ujAW5qD.png)

### 在程式中執行

另外一種方式是從程式碼中去呼叫並執行，我們只要在程式中加入下面這行就可以了

```
Artisan::call('edm:send');
```

使用起來像下方這樣

```php=
use Illuminate\Support\Facades\Artisan;

Route::get('/',function(){
    Artisan::call('edm:send');
});
```

## 做更多的事

除了上面描述的基本指令格式以外，其實市面上許多指令可以搭配一些可以選擇的項目、或者一些表示成功或失敗的文字跟顏色之類的， Laravel 也幫這樣的客製化準備好了

### 回應文字的 style

如果我想要將回饋給使用者的文字做一些變化，可以使用下面語法

```php=
public function handle()
{
    // 綠色
    $this->info('寄送 EDM 給所有的使用者');
    // 紅底
    $this->error('寄送 EDM 給所有的使用者');
    // 新增三行空白的空間
    $this->newLine(3);
    // 新的一行的文字
    $this->line('寄送 EDM 給所有的使用者');
}
```

以上執行起來會像下圖

![](https://i.imgur.com/9u3r1ge.png)

### option 選項

我們可以給我們的 Artisan Command 一些可以互動的選項，讓他使用起來可以更方便

#### Options Without Values 沒有值得選項

例如我們想讓信件只被寄出一次

```php=
 protected $signature = 'edm:send {--once}';
```

當指令有加入這個選項的時候 once 會為 ```true``` ，反之則為 ```false```

#### Options With Values 可指定值的選項

如果我們想要自己指定寄出的信件的次數，且讓次數預設為一次

```php=
 protected $signature = 'edm:send {--times=1}';
```

這樣我們就可以在指令後面加上 ```--times=5``` 讓信重複寄出五次

#### Option Shortcuts 簡寫

我也可以給選項一些簡寫

```php=
 protected $signature = 'edm:send {--o|once}';
```

這樣一來我們便可以用 ```-o``` 取代 ```--once```

#### Input Descriptions 描述

當然我們給的設定應該要寫一些說明給使用者看

```php=
 protected $signature = 'edm:send {user? : 沒有填寫會寄給全部的使用者，有寫的話可以寄給指定的人} 
                         {--o|once : 只寄出一封，可以使 --times 失效} 
                         {--times=1 : 預設為一次，可指定次數}';
```

這樣子我們在看指令的 help 的時候就可以看到我們寫的說明

![](https://i.imgur.com/RZgjaz6.png)

### 取得選項的輸入值

當我們下了像這樣的指令

```
php artisan edm:send user_a --times=4
```
我們的程式要怎麼拿到這些指令代表的值呢？
我們將參數分為 option 跟 argument 兩類，這兩類個別有自己取得參數值的方法

#### option 

我們可以使用下面的語法取得特定的 option

```php=
$this->option('once')
$this->option('times')
```

又或者可以直接取得全部的 option

```php=
$this->options()
```

#### argument 

我們可以使用下面的語法取得特定的 argument

```php=
$this->argument('user')
```

又或者可以直接取得全部的 argument

```php=
$this->arguments()
```

## 最終版本

經過上面的介紹，我們把自己新增的 Artisan Command 改寫成了下面的版本

```php=
namespace App\Console\Commands;

use Illuminate\Console\Command;

class SendEDMCommand extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'edm:send {user? : 沒有填寫會寄給全部的使用者，有寫的話可以寄給指定的人} {--o|once : 只寄出一封，可以使 --times 失效} {--times=1 : 預設為一次，可指定次數}';
    
    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = '寄送 EDM 給使用者';

    /**
     * Create a new command instance.
     *
     * @return void
     */
    public function __construct()
    {
        parent::__construct();
    }

    /**
     * Execute the console command.
     *
     * @return int
     */
    public function handle()
    {
        $user = $this->getSendUser();
        $send_times = $this->getSendTimes();
        for ($i = 0; $i < $send_times; $i++) {
            $this->info('寄送 EDM 給 ' . $user);
        }
    }

    private function getSendTimes()
    {
        if ($this->option('once') || $this->option('times') == 1) {
            $send_times = 1;
        } else {
            $send_times = $this->option('times');
        }
        return $send_times;
    }

    private function getSendUser()
    {
        if ( ! is_null($this->argument('user'))) {
            $user = $this->argument('user');
        } else {
            $user = '所有的使用者';
        }
        return $user;
    }
}
```

然後將 Artisan Command 給註冊

```php=
// app\Console\Kernel.php

protected $commands = [
    SendEDMCommand::class
];
```

之後我們便可以在黑窗內看到註冊成功的 Artisan Command 

![](https://i.imgur.com/1Uxpo8X.jpg)

而且可以 ```-h``` 看到他使用的方法

![](https://i.imgur.com/RZgjaz6.png)

最後我們來執行一下吧

![](https://i.imgur.com/AxBrWy6.png)

## Reference

- [Laravel Artisan Console](https://laravel.com/docs/8.x/artisan)
