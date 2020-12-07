---
title: 晚點再說啦三部曲之一 - Laravel Queue Job with Redis - 基礎篇
date: 2020-12-02 16:00:24
tags:
- PHP
- Laravel
- Job
- Queue
- Redis
categories: 
- Laravel
cover: https://picsum.photos/id/319/800/600
---

# 晚點再說啦三部曲之一 - Laravel Queue Job with Redis - 基礎篇

本文章使用的開發環境為

PHP : 7.3.11
Laravel : 8.16.1
Redis : 6.0
MySQL : 8.0

---

## 前言

對於網站來說，一個 Request 就代表一個從頭開始生成到最後銷毀的生命週期，時常我們會遇到一種狀況 :「如果我有一件事情想要在 request 結束之後的才做呢?」
舉個例子來說，試想我們有一個需求，需要在客戶下單後去寫入一個特殊的 Log ，以供之後做分析使用，而這個 Log 寫入普遍上來說又比較耗時(假設通常每一筆的寫入需耗費約一秒的時間成本)，那這樣不是會拖慢我們網站給使用者回饋的速度嗎?那麼在這種情境之下我們開如何去處理，從而節省那寶貴的一秒鐘呢?

想法其實很簡單，就像我們的日常工作一樣，有些東西不急，我們就先加入代辦事項，晚點再來做。而對於上面的需求來說，寫入特殊的 Log 並不是一定要立即更新那麼急迫的，所以我們要做的是

1. 找一個地方，把不急的工作給放進去記錄下來
2. 有空的時候我們就可以去這個地方把工作撿回來做完

這樣的情境與想法可以當作學習 Laravel Queue Job 的起點，有些狀況會變得不那麼抽象。

### 流程

當我們把情境拉回上面的例子，首先我們要做的是，把「寫入 Log 的工作」先放置到「某處」儲存起來。
在此處， Laravel 將「寫入 Log 的工作」的工作本體定義為一個稱作做 Job 的物件
而所謂的「某處」， Laravel 則提供了很多選擇可以使用，在這邊我們先用 [Redis](https://redis.io/) 作為例子。
而這時我們還得要指派一個角色，讓他可以有空的時候就去執行代辦的工作，在這個例子中， Sever 因為可以多工，我們讓他同時需要處理進來的 request ，同時也要去關心 Redis 內還沒有有其他工作要處理

如下圖，簡單示意了一下從放入工作到 Redis ，再到拿工作去處理的流程

![](https://i.imgur.com/BgXxhtD.png)

底下我們一個一個階段的去詳細解說

## Job Storage Driver 儲存 Job 的地方

既然我們想把工作儲存在某個地方，如上面舉例的想存在 Redis ，那麼很自然的就會產生一個疑問，那我想存在別的地方可以嗎?
結果是可以的， Laravel 提供了不少配合的 Driver 可以配合儲存待辦的工作，大致上有以下幾種

**Queue Drive:**
- Database 
    - 普通的資料庫，如 MySQL 之類的
    - 如果使用這個選項要額外下[指令](https://laravel.com/docs/8.x/queues#database)去聲成對應的特殊表
- Amazon SQS
    - Amazon 的服務
    - 要另外安裝套件 : [aws/aws-sdk-php](https://github.com/aws/aws-sdk-php) ~3.0
- [Beanstalkd](https://beanstalkd.github.io/)
    - 需另外安裝套件 : [pda/pheanstalk](https://github.com/pheanstalk/pheanstalk) ~4.0
- Redis
    - 需另外安裝套件 : [predis/predis](https://github.com/predis/predis) ~1.0 

在此處，因為筆者比較熟悉 Redis 的關係，所以我們選用 Redis 去作為儲存的載體

首先，我們假設 redis 已經安裝完畢，而 Laravel 與 Redis 溝通需要依賴套件，所以我們先裝 [predis/predis](https://github.com/predis/predis) 套件

```
composer require predis/predis
```

再來我們要去將與 redis 連線的設定做更改，底下會將其他設定先做簡化，只專注在必要調整的部分

```php=
//config/database.php

'redis' => [
    
    // 將預設的 client 調整為 predis 
    // 或是在 env 新增 REDIS_CLIENT 參數並設定為 predis
    'client' => env('REDIS_CLIENT', 'predis'), 

    'default' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'password' => env('REDIS_PASSWORD', null),
        'port' => env('REDIS_PORT', '6379'),
        // database 參數可以指定 redis 內的資料庫
        // 一般來說會看到16個（0~15)
        // 這邊預設指向0
        'database' => env('REDIS_DB', '0'),
    ],

],
```

如此一來 Laravel 與 Redis 的溝通手段就準備好了

## 介紹 Laravel Queue Job

儲存的地方準備好了，接下來我們要做的是將 Job 這個物件創造出來並且在需要的時候放進 redis 內

### 如何新增

首先我們要新增一個 Job ，這部分 Laravel 的 artisan 幫我們事先準備好了指令，讓我們可以透過下指令的方式輕鬆的產生 Job 類別

```
php artisan make:job WriteLogJob
```

而新產生的 Job 大致上會長這樣

```php=
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldBeUnique;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class WriteLogJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /**
     * Create a new job instance.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }

    /**
     * Execute the job.
     *
     * @return void
     */
    public function handle()
    {
        //
    }
}
```

```handle()``` 的部分就是讓我們去定義自己想要做的工作的地方，遵循上面的例子來說，我今天想要讓他去寫一個特殊的 Log 所以我可以將 ```handle()``` 內改寫為以下示範內容

```php=
public function handle()
{
    echo "write special log \n";
}
```

### 如何派發

我們已經準備好了 Job 本體，接下來我們來看看要怎麼樣將它給派發進去 redis 內， Laravel 很貼心的為 Job 物件事先包裝好了派發的方法，為求方便，我直接在 ```web.php``` 寫派發的過程

```php=
<?php

use App\Jobs\WriteLogJob;
use Illuminate\Support\Facades\Route;

Route::get(
    '/',
    function () {
        WriteLogJob::dispatch();
    }
);
```

只要用上面的語法，就可以輕易的將 Job 給派發到 redis 內，派發完之後 redis 裡面的內容會像是這樣

![](https://i.imgur.com/BvuMc4e.png)

到這邊其實就算是派發完成了，接下來就等待這些工作被執行。

而在派發的這個工作上， Laravel 還提供了你可以指定 queue 名稱讓我們可以去做 Job 的管理與分流，使用起來也相當簡單，語法如下

```php=
use App\Jobs\WriteLogJob;
use Illuminate\Support\Facades\Route;

Route::get(
    '/',
    function () {
        WriteLogJob::dispatch();
        WriteLogJob::dispatch()->onQueue('A');
        WriteLogJob::dispatch()->onQueue('B');
    }
);
```

只要我們加上了 ```onQueue('QueueName')``` 就可以將 Job 派發到指定的地方，派發完的結果如下

![](https://i.imgur.com/zN4tcJ2.png)

另外，如果除了 redis 以外我們還有不同的儲存載體的話，我們也可以在派發的時候指定要連接的地方，語法如下

```php=
use App\Jobs\WriteLogJob;
use Illuminate\Support\Facades\Route;

Route::get(
    '/',
    function () {
        WriteLogJob::dispatch()->onConnection('sqs');
    }
);
```

這個部分主要做的事情就是將我們寫好的 Job 給放到 redis 內，像是下圖這樣

![](https://i.imgur.com/aZjo3K0.png)

### 如何執行

我們已經將 redis 準備好， Job 也成功的派發了進去等待被處理，那麼我們該怎麼處理呢？ Laravel 也幫你把處理的動作也寫好了，同樣的我們僅僅需要下簡單的指令便可以讓他有空的時候就去撿 Job 回來做

Laravel 的官方網站上提供了兩種可以執行的指令，分別是

```
php artisan queue:work
php artisan queue:listen
```

以上兩種指令都可以達到執行 Job 的結果，只是為什麼需要兩種不一樣的指令去做同一件事情呢？官網有寫出這兩種指令的差異性

- ```php artisan queue:work```
    - 比起 ```php artisan queue:listen``` 在效率上更好一些
    - 但如果有更新 Job 的話必須要重新下指令
- ```php artisan queue:listen``` 
    - 比起 ```php artisan queue:work``` 在效率上會比較差
    - 如果有更新 Job 的話不用重新下指令

大致上可以這樣思考，如果在開發階段為了方便不想每次更新 Job 都去重新下指令的話，可以使用 ```php artisan queue:listen``` 就好，而在正式的產品上則使用 ```php artisan queue:work``` ，只是官網有特別指出，如果在正式的環境上讓指令做背景執行，要搭配監控的手段比較妥當，否則除非產品出了問題有人來回報，不然你不會知道執行程序跳掉了。

接下來我們來介紹一些常用的可以接在指令後面的參數，可以讓你的指令更靈活

**queue**

在上面有提到我們可以透過 ```onQueue()``` 去指定要放到那個分流去，這個參數讓你可以單獨執行該分流的任務

```
php artisan queue:work --queue=A
php artisan queue:work --queue=B
```
**once**

只執行一次（會執行順位最前面的那個）

```
php artisan queue:work --once
```
**tries**

當任務失敗，可以指定重新嘗試的次數

```
//錯誤重新嘗試三次
php artisan queue:work --tries=3 
```

**timeout**

我們都害怕某項任務卡住而癱瘓了後面該做的事情，所以我們可以為每個執行設定一但超過多少時間就跳過他

```
php artisan queue:work --timeout=30
```

指令執行起來大致上會像下圖般運作

![](https://i.imgur.com/7otJg6g.png)

### 失敗了怎麼辦？

再來我們談論到一個肯定會面臨的狀況，「如果我的 Job 執行的內容失敗了怎麼辦？」
這方面 Laravel 自然也是幫你準備好配套方案了， Laravel 幫你準備了一張資料表，讓整個流程中失敗的 Job 的相關訊息可以被記錄在裡面
該怎麼做呢？首先我們先新增一個 Laravel 幫我們準備好的 migration，並且將資料表新增（此處筆者使用的 Laravel 8.16.1 似乎在創立專案的時候就幫我們把 migration 給新增好了，所以下指令的時候他會告訴你已經存在，這時候就不用下第一個指令，直接做第二個的 migrate 就好）

```
php artisan queue:failed-table
php artisan migrate
```

這時候如果執行結果失敗， Laravel 則會將失敗的相關訊息幫你存在 failed_jobs 中

## 將整個流程串起來吧

經過上面的介紹，運行的流程大概如下，假設我已經將 redis 的連線設定全部都搞定了，接下來我會先新增 Job

### 新增 Job

```
php artisan make:job WriteLogJob
```

然後撰寫 Job 的內容

```php=
namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldBeUnique;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class WriteLogJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /**
     * Create a new job instance.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }

    /**
     * Execute the job.
     *
     * @return void
     */
    public function handle()
    {
        echo "write special log \n";
    }
}
```

### 派發

我們將執行的過程中加入 dispatch 的動作

```php=
<?php

use App\Jobs\WriteLogJob;
use Illuminate\Support\Facades\Route;

Route::get(
    '/',
    function () {
        WriteLogJob::dispatch();
        WriteLogJob::dispatch();
        WriteLogJob::dispatch();
    }
);
```

執行完之後我們會看到 redis 裡面多了新的 Job 等待被執行，如圖

![](https://i.imgur.com/o25etM2.png)

到這邊就表示了我們已經將待辦的事情放到 redis 了

### 執行

接下來我們需要一個人時時刻刻去關心 redis ，去處理這些任務，所以我們在專案內下了下面的指令

```
php artisan queue:work
```

然後我們可以看到下面的畫面，的確我們寫在 handle() 內的工作有被正常執行

![](https://i.imgur.com/1KQ2MMK.png)

### 錯誤處理

而如果執行的過程中有錯誤，錯誤訊息就會被丟入 failed_jobs 資料表並且被記錄下來

以下我們將 Job 的 handle() 改成會出錯的內容

```php=
public function handle()
{
    echo 字串沒有括起來;
}
```

再執行一次指令之後，會看到錯誤的告知

![](https://i.imgur.com/hmCkAh1.png)

而且在資料庫的 failed_jobs 表中可以看到詳細的錯誤訊息

![](https://i.imgur.com/a2zxOQa.png)

總結來說整理的流程串接起來大概上會像下面這張圖一樣

![](https://i.imgur.com/2W8LeXW.png)

## Reference

* [Laravel Queue Job](https://laravel.com/docs/8.x/queues)
* [Redis](https://redis.io/)
* [Beanstalkd](https://beanstalkd.github.io/)
* [aws/aws-sdk-php](https://github.com/aws/aws-sdk-php)
* [pda/pheanstalk](https://github.com/pheanstalk/pheanstalk)
* [predis/predis](https://github.com/predis/predis)



