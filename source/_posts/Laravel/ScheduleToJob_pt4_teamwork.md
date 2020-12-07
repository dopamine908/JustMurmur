---
title: 晚點再說啦三部曲之四 - 精美的合作
date: 2020-12-07 16:06:52
tags:
- PHP
- Laravel
- Queue
- Job
- Command
- Schedule
- Crontab
categories: 
- Laravel
cover: https://picsum.photos/id/444/800/600
---

# 晚點再說啦三部曲之四 - 精美的合作

本文章使用的開發環境為

PHP : 7.3.11
Laravel : 8.17.2
Redis : 6.0
MySQL : 8.0

---

## 前言

老實說我不確定大家是不是會跟我有一樣的困擾，有的時候單獨的功能即便看懂了，也不知道該如何使用，更何況有時候的狀況是我根本連功能介紹都看不懂。即便我們好不容易知道了功能的用途，卻不知道該在什麼樣的情境下能夠聯想到可以使用。所以我在撰寫前面的文章的時候很努力的想要創造一個合理的情境，讓讀的人可以有一個想像力能夠去聯想到在什麼時候可以連結這些功能去運用。再者，筆者我在學習這一系列功能的初期根本不能夠理解這些東西的運作及個別的配合，也是經過了摸索及研究，好不容易才看得清這些功能大致上的全貌，所以我認為有時候我們必須將眼光拉遠一點，先去知道一個流程中哪個部分是負責什麼事情，再進而去理解每個部分如何達成這件事情，這樣對於學習上來說會比較容易一點。

所以呢，就像四大天王一定有五個人一樣，三部曲也是會有第四個部分，我們來將之前的三個部分給串連起來。

所以說我們來做一個「每分鐘自動寄 EDM 給所有使用者」的功能吧～

我知道這功能聽起來很愚蠢，但你可以將「每分鐘」替換成「每周日」、「每個月初」、「每天午夜」，讓你可以在適合的時間點觸發，也可以將「自動寄 EDM 給所有使用者」替換成任何你想要做的動作，例如「寄送報表給相關人員」、「寄送系統 Log 給工程主管」等，我們可以保留架構，從而替換架構中每個節點的實作，就可以創造很多種可能性，那麼我們開始一步一步將功能建構出來吧！

## Job

使用者很多，我們當然不可能一次性的寄出那麼多的信件，我們得利用之前介紹過的 Job 來輔助，讓這些工作能夠被放到 redis 內讓 server 慢慢的去處理

所以我們首先下個指令讓 Job 的模板跑出來讓我們可以編輯吧

```
php artisan make:job SendEmailJob
```

然後我們在 ```app/Jobs``` 內就可以發現我們的 ```SendEmailJob.php```

下面我將它快速改寫成模擬寄信的樣子

```php=
<?php

namespace App\Jobs;

use App\Models\Demo;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class SendEmailJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    private $message;

    /**
     * Create a new job instance.
     *
     * @return void
     */
    public function __construct(string $message)
    {
        $this->message = $message;
    }

    /**
     * Execute the job.
     *
     * @return void
     */
    public function handle()
    {
        $DemoAction = new Demo();
        $DemoAction->action = $this->message;
        $DemoAction->save();

        echo "執行 Job -> 將訊息：'" . $this->message . "' 存進資料庫\n";
    }
}

```

這邊我新增了一個 ```Demo``` 資料表模擬客戶接收到 email 的行為，底下附上這張 ```Demo``` 資料表的 ```migration``` 

```php=
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreateDemoTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('Demo', function (Blueprint $table) {
            $table->id();
            $table->string('action');
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('Demo');
    }
}

```

既然新增了資料表，那麼我們也該新增一個 ```Model``` 來跟這張表做溝通，所以我們先下指令把 ```Model``` 新增出來

```
php artisan make:model Demo
```

更改一下 ```Model``` 的設定，讓他綁定 ```Demo``` 資料表

```php=
// app/Models/Demo.php

<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Demo extends Model
{
    use HasFactory;

    protected $table = 'Demo';
}

```

## Config

接下來我們來設定一下跟 redis 溝通的一些設定，我們要看的是 ```config``` 資料夾內的 ```database.php``` 、 ```queue.php```

```php=
// database.php

'redis' => [

    'client' => env('REDIS_CLIENT', 'predis'),

    'default' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'password' => env('REDIS_PASSWORD', null),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_DB', '0'),
    ],

],

```

確認 ```client``` 參數抓到的值會是 ```predis``` ，這個部分可以直接在 ```php``` 檔裡面寫死，或是在 ```.env``` 新增 ```REDIS_CLIENT``` 參數並賦值。

然後確認 ```default``` 底下的連線設定有調整到你要的連線設定，這部分賦值的邏輯同上，可以直接在 ```php``` 檔裡面寫死，或是在 ```.env``` 新增對應參數並賦值。

不過筆者的習慣是都會設定讓程式去找 ```.env``` 的參數，這樣在部署不同環境的時候只要替換不同環境的 ```.env``` 檔就可以套用新的參數了，不用再去改程式。

```database.php``` 主要是掌管與資料庫連線有關的一些設定，除了這邊講到的 redis 的設定以外，我們也可以新增好幾個 mysql 的設定，這部分以後有機會可以再談

```php=
// queue.php

<?php

return [

    'default' => env('QUEUE_CONNECTION', 'redis'),

    'connections' => [
        'redis' => [
            'driver' => 'redis',
            'connection' => 'default',
            'queue' => env('REDIS_QUEUE', 'default'),
            'retry_after' => 90,
            'block_for' => null,
        ],
    ],

    'failed' => [
        'driver' => env('QUEUE_FAILED_DRIVER', 'database-uuids'),
        'database' => env('DB_CONNECTION', 'mysql'),
        'table' => 'failed_jobs',
    ],

];

```

這部分要確認的是 ```default``` 、 ```connections.redis``` 、 ```failed``` 的設定是否如你期望

```queue.php``` 主要是掌管 queue 功能會需要用到的設定，包括 queue 運作要參照的資料庫設定以及失敗的錯誤處理要參照的資料庫設定，而這部分的設定還會去參考剛剛 ```database.php``` 裡面的設定。

建構這部分的功能常常會遇到 Job 新增不進去 redis ，或是新增進去了但是 queue 指令下完卻沒反應的狀況，這時候通常第一個都會檢查這部分的設定有沒有設定錯誤或是沒有套用，通常修正過後重新執行指令問題就會得到解決。

## 測試一下

接下來我們隨便寫一個觸發的功能，然後確認一下功能是否正常，這個地方的確認筆者通常會分為兩個步驟

1. 檢查 Job 是否可以成功的被新增進去我們指定的 redis 資料庫
2. 在 Job 被成功新增到 redis 的狀況下，下指令是否可以正常執行

### 新增 Job

首先我們在 ```route/web.php``` 寫一隻測試觸發用的 route

```php=
//route/web.php

<?php

use App\Jobs\SendEmailJob as SendEmailJob;
use Illuminate\Support\Facades\Route;

Route::get(
    '/test_job_dispatch',
    function () {
        SendEmailJob::dispatch();
    }
);
```

然後我們瀏覽這個網址測試一下觸發的效果如何

![](https://i.imgur.com/vx8ozg8.png)

不錯，看起來有成功的被新增進去了

### queue 執行測試

然後我們要執行一下指令，確定這個 Job 真的有被執行

依照我們剛剛寫的內容，執行成功的話 ```Demo``` 資料表內應該會有一筆資料，而且黑窗內也會印出 ```執行 Job -> 將訊息：'test message' 存進資料庫``` 

接下來我們輸入指令並執行

```
php artisan queue:work --tries=1
```

在開發時候，筆者會很習慣的使用 ```--tries=1``` 這個設定，這樣才不會導致當初錯的時候 queue 一直 retry 

![](https://i.imgur.com/lSDeySF.png)

![](https://i.imgur.com/qkigYce.png)

測試成功！

## Artisan Command

接下來我們要寫一個可以觸發上面流程的 Artisan Command ，我期望我們下了指令之後，就可以把 Job 派發到 redis 等著 queue 執行

所以我們先用指令新增一個 Artisan Command

```
php artisan make:command SendEmailCommand
```

新增好的 Artisan Command 檔案在 ```app/Console/Commands``` ，一樣我們把它改寫成我們要的形式

```php=
// app/Console/Commands/SendEmailCommand.php

<?php

namespace App\Console\Commands;

use App\Jobs\SendEmailJob;
use Illuminate\Console\Command;

class SendEmailCommand extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'edm:send';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = '寄送 edm 給所有使用者';

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
        SendEmailJob::dispatch('寄送 edm 給所有使用者');
        $this->info('已經寄信的任務派發到 Job 中等待執行');
    }
}

```

再來別忘記在 ```app/Console/Kernel.php``` 中註冊

```php=
<?php

namespace App\Console;

use App\Console\Commands\SendEmailCommand;
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
        SendEmailCommand::class
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

接下來我們就可以在 Artisan 內看到註冊過的 Command 了

![](https://i.imgur.com/GwzgYfv.png)

## 再測試一下

接下來我們要實際下指令測試看看，在 ```php artisan queue:work --tries=1``` 有啟動運作的時候，下完指令之後應該會有一個 Job 被派發到 redis 並且會立刻被 queue 給拿去處理

![](https://i.imgur.com/jevHUH3.png)

![](https://i.imgur.com/m9afn3q.png)

不錯，一切都有如我們預期的運作

## Schedule

接下來我們要來寫 Schedule ，讓他可以每分鐘去觸發 ```php artisan edm:send``` 的指令

我們在 ```app/Console/Kernel.php``` 裡面的 ```schedule()``` 註冊要執行的指令及時間頻率

```php=
// app/Console/Kernel.php

<?php

namespace App\Console;

use App\Console\Commands\SendEmailCommand;
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
        SendEmailCommand::class
    ];

    /**
     * Define the application's command schedule.
     *
     * @param \Illuminate\Console\Scheduling\Schedule $schedule
     * @return void
     */
    protected function schedule(Schedule $schedule)
    {
        $schedule->command('edm:send')->everyMinute();
        echo '['.Carbon::now()->toDateTimeString().'] '."已執行 edm:send 指令 \n";
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

我們註冊了每分鐘去執行一次 ```php artisan edm:send``` 的指令

## 最後測試

接下來我們要測試看看整個流程是否安好且順暢，我們要用 ```php artisan schedule:work``` 代替 ```php artisan schedule:run``` 來模擬觸發，並期望每分鐘 ```edm:send``` 的指令都被觸發

![](https://i.imgur.com/GLcZXFH.png)

![](https://i.imgur.com/o5Wk4Rp.png)

放置了五分鐘之後看起來結果相當成功，到這邊整個流程算是已經完成。

## 小結

花了不少功夫去建立整個功能，最後我們來看一張整個流程的示意圖，讓一切可以更明朗一些

![](https://i.imgur.com/ZGpTS6H.png)

