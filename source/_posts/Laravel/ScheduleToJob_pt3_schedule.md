---
title: 晚點再說啦三部曲之三 - Laravel Task Scheduling - 基礎篇
date: 2020-12-04 22:05:12
tags:
- PHP
- Laravel
- Schedule
- Crontab
categories: 
- Laravel
cover: https://picsum.photos/id/357/800/600
---

# 晚點再說啦三部曲之三 - Laravel Task Scheduling - 基礎篇

本文章使用的開發環境為

PHP : 7.3.11
Laravel : 8.16.1
Redis : 6.0
MySQL : 8.0

---

## 前言

如上一篇所述，本篇要來探討「如何在某個時間點觸發」。

我們一樣來想像一個情境，假設今天你的客戶要做一個電商網站，他跟你說他需要在每週一收到一封上週的銷售報表，時間的結算週期為週一的凌晨00:00到週日的晚上12:00，這樣的話我們該如何去實作這樣的需求呢？

我們可能會先想到，我們的網頁是在 Linux 上運作，那我們使用 Linux 上的 crontob 去設定週期，讓他在每週一的凌晨 00:05 去執行生成上週的報表並寄信的功能就好啦！

的確，這可以解決問題，但當今天我們的電商網站逐漸茁壯，我們要記得報表越來越多元，每一種爆報表的時間週期也都不盡相同，而且因為流量的上升，我們常常需要開好幾台 Server 去做流量的分流，這意味著我必須在每一台 Server 上都定義相同的 crontab 才行，而如果這時候，我要新增一種新的寄送報表任務呢？又或是要更改某一個報表的寄送週期呢？免不了的我必須進到每一台 Server 裡面去修改 crontab 才可以，而且修改有沒有成功，也要等到下一個觸發時間點才能夠知曉，這對於整個維運上無疑的是一場災難。

所以這裡產生了一個想法，「要是我可以讓 Code 本身掌握所有任務的時間點就好了，這樣更新 Code 就能套用新的規則了」。

非常幸運的， Laravel 也幫我們準備好了這樣子的功能，讓我們可以在 Code 內去管控任務的執行時間與週期，以下我們來看看該怎麼去實作。

## 定義

首先我們看到 ```app/Console/Kernel.php``` 

```php=
// app/Console/Kernel.php

namespace App\Console;

use App\Console\Commands\SendEDMCommand;
use Carbon\Carbon;
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
        
    ];

    /**
     * Define the application's command schedule.
     *
     * @param \Illuminate\Console\Scheduling\Schedule $schedule
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
        $this->load(__DIR__ . '/Commands');

        require base_path('routes/console.php');
    }
}
```

我們可以在這支檔案內找到 ```schedule()``` 這個 function ，我們可以在這個 function 內去定義我們的執行任務還有他的週期時間

## 執行

### 在 Server 上

該如何執行呢？
以往我們可能會在 crontab 中寫入很多指令，但現在我們只需要將它精簡到只剩下一條

```
* * * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1
```

簡單的來說，這樣設定的用意就是讓我們的 Server 每分鐘去執行 ```php artisan schedule:run``` 這個指令，而在執行這個指令的當下， Laravel 會去幫你檢查有哪些任務該做了哪些還不用，如果遇到該做的任務他就會幫你去執行。


### 本地開發

在本地開發的時候，我們可能會因為開發使用的系統的不同，無法使用 crontab ，而這時候該怎麼辦呢？
同樣的，你的考量 Laravel 也知道，所以 Laravel 也提供了一個```可以模擬每分鐘執行 php artisan schedule:run ```的指令，只要改成下這個指令就可以了

```
php artisan schedule:work
```

以下示範的例子都會使用這個指令去當作模擬而執行

## 加入排程

接下來我們要正式的將我們的任務加入 ```schedule()``` 了， Laravel 也提供了很多不同的執行任務手段給我們使用

### call

我們可以將自己想做的事情直接定義在 ```call()``` 裡面，可以透過 ```closure``` 或是類別的 ```__invoke()``` function 達到呼叫的效果

```php=
protected function schedule(Schedule $schedule)
{
    // by closure
    $schedule->call(function () {
        DB::table('recent_users')->delete();
    })->daily();

    // by __invoke
    $schedule->call(new DeleteRecentUsers)->daily();
}
```

> ```daily() ``` 的意思是每天執行一次，對於這種頻率的設定底下會有說明

### command

我們可以呼叫任何已存在的 Artisan Command 

```php=
protected function schedule(Schedule $schedule)
{
    $schedule->command('emails:send Taylor --force')->daily();
    $schedule->command(EmailsCommand::class, ['Taylor', '--force'])->daily();
}
```

### queue

對於之前新增過的 Job ，也可以直接得去 dispatch

```php=
protected function schedule(Schedule $schedule)
{
    $schedule->job(new Heartbeat)->everyFiveMinutes();

    // Dispatch job 到 "heartbeats" 這個 queue
    $schedule->job(new Heartbeat, 'heartbeats')->everyFiveMinutes();
}
```

### exec

想要跳脫專案的範圍直接對主機下指令也是辦得到的

```php=
protected function schedule(Schedule $schedule)
{
    $$schedule->exec('node /home/forge/script.js')->daily();
}
```

##  Frequency Options

可以執行任務了，那麼多久執行一次呢？可以在特定時間點才執行嗎？

以下列出一些比較常用的時間頻率設定，詳細的可以參考[這邊](https://laravel.com/docs/8.x/scheduling#schedule-frequency-options)

### 每過某一段時間執行一次

```php=
// 每分鐘一次
$schedule->command('edm:send')->everyMinute();	

// 每小時一次
$schedule->command('edm:send')->hourly();	

// 每天凌晨00:00
$schedule->command('edm:send')->daily();

// 每週一凌晨00:00
$schedule->command('edm:send')->weekly();	

// 每個月的第一天的凌晨00:00
$schedule->command('edm:send')->monthly();	
```
### 固定某個時間點執行

```php=
//以下依序為 週一～週日
$schedule->command('edm:send')->mondays();
$schedule->command('edm:send')->tuesdays();	
$schedule->command('edm:send')->wednesdays();	
$schedule->command('edm:send')->thursdays();	
$schedule->command('edm:send')->fridays();	
$schedule->command('edm:send')->saturdays();	
$schedule->command('edm:send')->sundays();
```
### 全客製化週期

```php=
// 跟 crontab 一樣的用法，可以自訂想要的週期
$schedule->command('edm:send')->cron('* * * * *');
```

## 實踐

我們用上面的技巧來做一個每分鐘模擬寄信的功能吧

首先，我們在 ```app/Console/Kernel.php``` 的 ```schedule()``` 中新增我們想要的任務內容及設定時間

```php=
// app/Console/Kernel.php

namespace App\Console;

use App\Console\Commands\SendEDMCommand;
use Carbon\Carbon;
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
        
    ];

    /**
     * Define the application's command schedule.
     *
     * @param \Illuminate\Console\Scheduling\Schedule $schedule
     * @return void
     */
    protected function schedule(Schedule $schedule)
    {
        $schedule->call(
            function () {
                $time = Carbon::now()->toDateTimeString();
                echo $time . " : send EDM to every user~ \n";

            }
        )->everyMinute();
    }

    /**
     * Register the commands for the application.
     *
     * @return void
     */
    protected function commands()
    {
        $this->load(__DIR__ . '/Commands');

        require base_path('routes/console.php');
    }
}

```

我們使用 ```call()``` 直接將我們想做的事情寫在 closure 內，這邊大致上用 ```echo``` 代表我們已經寄信了，而後面的時間頻率我們用 ```->everyMinute()``` 設定每分鐘執行，最後在 Local 環境上我們用 ```php artisan schedule:work``` 去模擬 crontab 的每分鐘執行，結果會如下

![](https://i.imgur.com/nH9PT7p.png)

我們可以從上方的圖看到每一分鐘的00秒，代表寄信的 ```echo``` 被執行了

整體的流程大致上會像是這樣

![](https://i.imgur.com/ZZfyji4.png)

## Reference

- [Laravel Task Scheduling](https://laravel.com/docs/8.x/scheduling)
