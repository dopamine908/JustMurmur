---
title: Laravel 中的職責鏈模式 - Middleware 背後的魔法 及 在 Query 上的應用
date: 2021-01-21 14:25:44
tags:
- PHP
- Laravel
- Pipeline
- Middleware
- Query
- Design Pattern
categories:
- Laravel
cover: https://picsum.photos/id/660/800/600
---

# Laravel 中的職責鏈模式 - Middleware 背後的魔法 及 在 Query 上的應用

## 職責練模式

這篇會用到職責練模式的概念，但不會從頭講解職責練模式，如果對於這個設計模式還不熟悉，建議先理解一下再來閱讀會對理解上比較有幫助一些

**職責練模式** （[可以參考這邊](https://refactoring.guru/design-patterns/chain-of-responsibility)）

用簡單又較為口語化的方式來敘述職責練模式來說，我們讓一個處理的過程拆解成像是一個產線的感覺，這一站處理完換下一站處理，每一站都可以獨自判斷自己該運作或是根據某些條件 pass 給下一站

在設計上會讓大家共同實作某個讓執行者可以執行的介面，例如 ```handle()``` ， 底下看一下大致上的示意圖

![](https://i.imgur.com/KHBXlfD.png)

所以說，為什麼會提到職責練模式呢？

其實，本篇的主角是一個 Laravel 官方文件也沒提及的一個類別，他叫做 Pipeline ，雖然沒有當作一個獨立的功能記載在文件上，但在 Laravel 的生命週期中卻扮演著相當重要的角色，而今天我們就是來試圖去理解它，並且將 Pipeline 獨立應用，看會發生什麼有趣的事情

## Laravel Middleware

以下的程式範例會使用 Laravel 官方的 github 上的程式碼，我們將會使用 Laravel 8.0 當作範例

**Github Repository 來源**
- [Laravel](https://github.com/laravel/laravel/tree/v8.0.0)
- [Framework](https://github.com/laravel/framework/tree/v8.0.0)

那麼，在 Laravel 的生命週期中，Pipeline 究竟是扮演著一個什麼樣的角色呢？

我們要先從 Laravel 的生命週期進入點開始一步一步的看中間到底發生了什麼事情

### 生命週期的進入點 —— index.php

首先我們看到 ```public/index.php```

[**public/index.php**](https://github.com/laravel/laravel/blob/v8.0.0/public/index.php)

```php=
<?php

use Illuminate\Contracts\Http\Kernel;
use Illuminate\Http\Request;

define('LARAVEL_START', microtime(true));

if (file_exists(__DIR__.'/../storage/framework/maintenance.php')) {
    require __DIR__.'/../storage/framework/maintenance.php';
}

require __DIR__.'/../vendor/autoload.php';

$app = require_once __DIR__.'/../bootstrap/app.php';

$kernel = $app->make(Kernel::class);

$response = tap($kernel->handle(
    $request = Request::capture()
))->send();

$kernel->terminate($request, $response);

```

看到 14 行的位置
```
$app = require_once __DIR__.'/../bootstrap/app.php';
```
這邊將 Laravel 整個應用程式給載入進來，設定為 ```$app```

接著在 16 行的位置

```
$kernel = $app->make(Kernel::class);
```

這邊將 Laravel 運行的核心 ： ```Kernel::class``` 設定成 ```$kernel```

而這個 ```$kernel``` 的實體則是 [```app/Http/Kernel.php```](https://github.com/laravel/laravel/blob/v8.0.0/app/Http/Kernel.php)
這部分是因為在 [index.php](https://github.com/laravel/laravel/blob/v8.0.0/public/index.php) 中所使用的 ```Kernel::class``` 的來源就是在上方 use 的 [```Illuminate\Contracts\Http\Kernel```](https://github.com/laravel/framework/blob/v8.0.0/src/Illuminate/Contracts/Http/Kernel.php) ，但是 [```Illuminate\Contracts\Http\Kernel```](https://github.com/laravel/framework/blob/v8.0.0/src/Illuminate/Contracts/Http/Kernel.php) 其實只是一個介面而已，這時候我們要看到 14 行所提到的 [```bootstrap/app.php```](https://github.com/laravel/laravel/blob/v8.0.0/bootstrap/app.php) ，我們可以看到在這裡面有對 [```Illuminate\Contracts\Http\Kernel```](https://github.com/laravel/framework/blob/v8.0.0/src/Illuminate/Contracts/Http/Kernel.php) 去執行綁定實體的動作，如下方的 7~10 行的位置

[**bootstrap/app.php**](https://github.com/laravel/laravel/blob/v8.0.0/bootstrap/app.php)
```php=
<?php

$app = new Illuminate\Foundation\Application(
    $_ENV['APP_BASE_PATH'] ?? dirname(__DIR__)
);

$app->singleton(
    Illuminate\Contracts\Http\Kernel::class,
    App\Http\Kernel::class
);

$app->singleton(
    Illuminate\Contracts\Console\Kernel::class,
    App\Console\Kernel::class
);

$app->singleton(
    Illuminate\Contracts\Debug\ExceptionHandler::class,
    App\Exceptions\Handler::class
);

return $app;

```

也就是說，在 [public/index.php](https://github.com/laravel/laravel/blob/v8.0.0/public/index.php) 繼續往下執行到要取得 ```$response``` 的時候

[**public/index.php#L51**](https://github.com/laravel/laravel/blob/v8.0.0/public/index.php#L51)

```php=
<?php

$response = tap($kernel->handle(
    $request = Request::capture()
))->send();

```

我們會對 [```app/Http/Kernel.php```](https://github.com/laravel/laravel/blob/v8.0.0/app/Http/Kernel.php) 去執行一個名為 ```handle()``` 的 function

接下來我們要轉換我們關心的目標，我們來看看這個 ```handle()``` 做了哪些事情

### App/Http/Kernel.php

我們先來看看 App/Http/Kernel.php 裡面有什麼內容吧

[**App/Http/Kernel.php**](https://github.com/laravel/laravel/blob/v8.0.0/app/Http/Kernel.php)
```php=
<?php

namespace App\Http;

use Illuminate\Foundation\Http\Kernel as HttpKernel;

class Kernel extends HttpKernel
{
    protected $middleware = [
        // \App\Http\Middleware\TrustHosts::class,
        \App\Http\Middleware\TrustProxies::class,
        \Fruitcake\Cors\HandleCors::class,
        \App\Http\Middleware\PreventRequestsDuringMaintenance::class,
        \Illuminate\Foundation\Http\Middleware\ValidatePostSize::class,
        \App\Http\Middleware\TrimStrings::class,
        \Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull::class,
    ];

    protected $middlewareGroups = [
        'web' => [
            \App\Http\Middleware\EncryptCookies::class,
            \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
            \Illuminate\Session\Middleware\StartSession::class,
            // \Illuminate\Session\Middleware\AuthenticateSession::class,
            \Illuminate\View\Middleware\ShareErrorsFromSession::class,
            \App\Http\Middleware\VerifyCsrfToken::class,
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
        ],

        'api' => [
            'throttle:api',
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
        ],
    ];

    protected $routeMiddleware = [
        'auth' => \App\Http\Middleware\Authenticate::class,
        'auth.basic' => \Illuminate\Auth\Middleware\AuthenticateWithBasicAuth::class,
        'cache.headers' => \Illuminate\Http\Middleware\SetCacheHeaders::class,
        'can' => \Illuminate\Auth\Middleware\Authorize::class,
        'guest' => \App\Http\Middleware\RedirectIfAuthenticated::class,
        'password.confirm' => \Illuminate\Auth\Middleware\RequirePassword::class,
        'signed' => \Illuminate\Routing\Middleware\ValidateSignature::class,
        'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
        'verified' => \Illuminate\Auth\Middleware\EnsureEmailIsVerified::class,
    ];
}

```

可以看到他只定義了一些關於 middleware 的變數，並沒有看到我們要執行的 handle() ，所以我們往他繼承的父累別去找，果然我們在 [Illuminate\Foundation\Http\Kernel](https://github.com/laravel/framework/blob/v8.0.0/src/Illuminate/Foundation/Http/Kernel.php) 找到了 handle()

[**Illuminate\Foundation\Http\Kernel#L105**](https://github.com/laravel/framework/blob/v8.0.0/src/Illuminate/Foundation/Http/Kernel.php#L105)
```php=
public function handle($request)
{
    try {
        $request->enableHttpMethodParameterOverride();

        $response = $this->sendRequestThroughRouter($request);
    } catch (Throwable $e) {
        $this->reportException($e);

        $response = $this->renderException($request, $e);
    }

    $this->app['events']->dispatch(
        new RequestHandled($request, $response)
    );

    return $response;
}
```

我們從第 6 行的位置繼續往下追，找一下 ```sendRequestThroughRouter($request)```

[**Illuminate\Foundation\Http\Kernel#L130**](https://github.com/laravel/framework/blob/v8.0.0/src/Illuminate/Foundation/Http/Kernel.php#L130)

```php=
protected function sendRequestThroughRouter($request)
{
    $this->app->instance('request', $request);

    Facade::clearResolvedInstance('request');

    $this->bootstrap();

    return (new Pipeline($this->app))
                ->send($request)
                ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
                ->then($this->dispatchToRouter());
}
```

看到我們的目標了

[**Illuminate\Foundation\Http\Kernel#L138**](https://github.com/laravel/framework/blob/v8.0.0/src/Illuminate/Foundation/Http/Kernel.php#L138)

```php=
return (new Pipeline($this->app))
                ->send($request)
                ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
                ->then($this->dispatchToRouter());
```

大致上來解釋一下這邊發生了什麼事情，這裡我們經由 ```Pipeline``` 寄送了 ```$request``` ，中間我們要經過 ```$this->middleware``` ，最後將 ```$request``` 派發到對應的 route 去執行程式，取得 response ，最後就是回傳

在上方我們可以看到 ```Pipeline``` 這個類別掌管了從 request 經由 middleware 最後產生 response 的過程，所以說 ```Pipeline``` 在 Laravel 框架的生命週期中其實扮演著相當重要的角色，他可以將 request 拿去執行多個 middleware 之後，最後透過執行 ```dispatchToRouter()``` 進而取得 response

這部分的 middleware 有自己一套複雜決策的過程，這邊我們省略，但一般的狀況下 ```$this->middleware``` 就會是我們在 [App/Http/Kernel.php#L16](https://github.com/laravel/laravel/blob/v8.0.0/app/Http/Kernel.php#L16) 所宣告的這些 middleware

[**App/Http/Kernel.php#L16**](https://github.com/laravel/laravel/blob/v8.0.0/app/Http/Kernel.php#L16)
```php=
protected $middleware = [
    // \App\Http\Middleware\TrustHosts::class,
    \App\Http\Middleware\TrustProxies::class,
    \Fruitcake\Cors\HandleCors::class,
    \App\Http\Middleware\PreventRequestsDuringMaintenance::class,
    \Illuminate\Foundation\Http\Middleware\ValidatePostSize::class,
    \App\Http\Middleware\TrimStrings::class,
    \Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull::class,
];
```

## What's Pipeline

在探討什麼是 Pipeline 之前，我們先來看一個實際上使用 Pipeline 的例子

```php=
$pipeline = app(Pipeline::class)
        ->send($some_instance)
        ->through(
            [
                Step1::class,
                Step2::class,
                Step3::class,
            ]
        )->thenReturn();
```

接下來我們會針對上面這一段 code 去解析這個過程中間 Pipeline 幫助我們做了什麼事情

首先 Pipeline 原始的 namespace 為 [```Illuminate\Pipeline\Pipeline```](https://github.com/laravel/framework/blob/v8.0.0/src/Illuminate/Pipeline/Pipeline.php)

我們先來介紹幾個等等會用到的參數

[**Illuminate\Pipeline\Pipeline#L25**](https://github.com/laravel/framework/blob/v8.0.0/src/Illuminate/Pipeline/Pipeline.php#L25)

```php=
/**
 * The object being passed through the pipeline.
 *
 * @var mixed
 */
protected $passable;

/**
 * The array of class pipes.
 *
 * @var array
 */
protected $pipes = [];

/**
 * The method to call on each pipe.
 *
 * @var string
 */
protected $method = 'handle';
```

- ```$passable```
    - 要在這整個過程中傳遞的物件將會被設置在這個變數中(以上面的例子來說就是指 ```$some_instance```)
- ```$pipes```
    - 這個變數裡面會放要執行的過程中間的每一個步驟(以上面的例子來說就是```Step1::class, Step2::class, Step3::class```)
- ```$method```
    - 原始碼中直接將這個變數定義為 ```'handle'``` ，這個其實就是我們在職責練模式中每一個職責鏈物件所共同會去實作的 ```handle()``` ，由於這個抽象的 function 名稱可能因人而異，所以雖然預設為 ```'handle'``` ，但其實是可以修改設定的

接著我們照上面執行的 function 一步一步去看每個步驟做了什麼吧

在我們用 ```app(Pipeline::class)``` 將 Pipeline 實體化之後，我們執行了 ```send($some_instance)```

[**send($passable)**](https://github.com/laravel/framework/blob/v8.0.0/src/Illuminate/Pipeline/Pipeline.php#L58)
```php=
public function send($passable)
{
    $this->passable = $passable;
    return $this;
}
```

這裡會將我們所帶入的 ```$some_instance``` 設置到 ```$passable``` 變數中，而這個 ```$some_instance``` 就會成為我們在職責鏈模式中傳遞的那個物件

接著我們執行了

```php=
->through(
    [
        Step1::class,
        Step2::class,
        Step3::class,
    ]
)
```

所以我們來看一下 ```through()```

[**through($pipes)**](https://github.com/laravel/framework/blob/v8.0.0/src/Illuminate/Pipeline/Pipeline.php#L71)
```php=
public function through($pipes)
{
    $this->pipes = is_array($pipes) ? $pipes : func_get_args();
    return $this;
}
```

在這邊因為我丟進去的 ```[Step1::class, Step2::class, Step3::class]``` 是一個 array ，所以這邊會直接將 ```$pipes``` 設置為 ```[Step1::class, Step2::class, Step3::class]```，而 ```Step1::class, Step2::class, Step3::class``` 就是職責鏈模式中的職責物件，會去處理被傳遞的那個物件並讓他繼續往後傳遞

接下來執行了最後一個 function ，我們執行了 ```thenReturn()```

[**thenReturn()**](https://github.com/laravel/framework/blob/v8.0.0/src/Illuminate/Pipeline/Pipeline.php#L111) 、 [**then()**](https://github.com/laravel/framework/blob/v8.0.0/src/Illuminate/Pipeline/Pipeline.php#L97)
```php=
public function thenReturn()
{
    return $this->then(function ($passable) {
        return $passable;
    });
}

public function then(Closure $destination)
{
    $pipeline = array_reduce(
        array_reverse($this->pipes()), $this->carry(), $this->prepareDestination($destination)
    );
    return $pipeline($this->passable);
}
```

我們可以看到， ```thenReturn()``` 最終會去執行 ```then(Closure $destination)``` ，而這部分的執行內容有些複雜，但大致上來說就是將我們前面設定好的被傳遞物件 ```$passable``` (```$some_instance```) 拿去讓他在職責物件中(```$pipes=[Step1::class, Step2::class, Step3::class]```)傳遞並執行，而這個地方比較值得注意的是在上面 code 的第 11 行的 ```$this->carry()``` 內的內容

[**carry()**](https://github.com/laravel/framework/blob/v8.0.0/src/Illuminate/Pipeline/Pipeline.php#L140)
```php=
protected function carry()
{
    return function ($stack, $pipe) {
        return function ($passable) use ($stack, $pipe) {
            try {
                if (is_callable($pipe)) {
                    // If the pipe is a callable, then we will call it directly, but otherwise we
                    // will resolve the pipes out of the dependency container and call it with
                    // the appropriate method and arguments, returning the results back out.
                    return $pipe($passable, $stack);
                } elseif (! is_object($pipe)) {
                    [$name, $parameters] = $this->parsePipeString($pipe);

                    // If the pipe is a string we will parse the string and resolve the class out
                    // of the dependency injection container. We can then build a callable and
                    // execute the pipe function giving in the parameters that are required.
                    $pipe = $this->getContainer()->make($name);

                    $parameters = array_merge([$passable, $stack], $parameters);
                } else {
                    // If the pipe is already an object we'll just make a callable and pass it to
                    // the pipe as-is. There is no need to do any extra parsing and formatting
                    // since the object we're given was already a fully instantiated object.
                    $parameters = [$passable, $stack];
                }

                $carry = method_exists($pipe, $this->method)
                                ? $pipe->{$this->method}(...$parameters)
                                : $pipe(...$parameters);

                return $this->handleCarry($carry);
            } catch (Throwable $e) {
                return $this->handleException($passable, $e);
            }
        };
    };
}

```

這個部分有一點相當值得注意，看到上方的 27 行附近的地方

```php=
$carry = method_exists($pipe, $this->method)
            ? $pipe->{$this->method}(...$parameters)
            : $pipe(...$parameters);
```

這個地方設置了關於我們在每個職責物件(```$pipes```內的元素)需要去執行的 function ，在這邊會先去檢驗 ```$this->method``` 是否正常存在，如果沒有被刻意更改過， ```$this->method``` 在一開始我們是預設為 ```'handle'``` 的，所以預設的狀況下， Pipeline 會去執行每個職責物件的 ```handle()```

所以在我不改變執行方法的前提下，我必須要讓我的每個職責物件都去實作 ```handle()``` 才可以，這方面其實相當類似我們在職責鏈模式中會去提取一個強迫所有職責物件去實作 ```handle()``` 的抽象類別或是介面

當然 Laravel 有提供你可以去修改執行的 function 名稱，這部分可以參見 [```via($method)```](https://github.com/laravel/framework/blob/v8.0.0/src/Illuminate/Pipeline/Pipeline.php#L84) 方法

以上大致上介紹了 Pipeline 裡面的內容及運行的整個流程，簡單來說 Pipeline 扮演的就是一個在職責練模式中的 Client 的角色，只是 Laravel 將他封裝的更完整、功能性更齊全、同時也更容易擴展

我們可以傳遞任何的物件讓他流淌於各個職責物件中，同時也可以對於職責物件的先後順序輕易的調換，而且可以將職責物件的共同介面名稱修改的更具表達力，我們只要專注於每一個職責物件內的判斷及實作即可

## Run Query with Pipeline

接著我們來看一個實際的例子，我們來給定一個情境吧

假設我們有一張記載學生的資料表

有幾個主要的欄位

- 名字
- 性別
- 居住地
- 年齡

然後假設我們想要對這些學生提供幾種搜尋的方式

1. 依照性別搜尋
2. 依照居住地搜尋
3. 依照年齡搜尋

以上三種搜尋條件有可能只出現一個，或是出現某兩個，也可能三個條件都具備

首先我們先來為這樣的需求來做點準備吧

先建立 migration

```
php artisan make:migration create_students_table --create=students
```

在 migration 將欄位設定

```php=
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreateStudentsTable extends Migration
{
    
    public function up()
    {
        Schema::create(
            'students',
            function (Blueprint $table) {
                $table->id();
                $table->string('name');
                $table->enum('gender', ['boy', 'girl']);
                $table->enum('area', ['taipei', 'tainan', 'kaohsiung']);
                $table->tinyInteger('age');
                $table->timestamps();
            }
        );
    }

    public function down()
    {
        Schema::dropIfExists('students');
    }
}

```

設定完之後執行指令建立資料表

```
php artisan migrate
```

接著幫該資料表建立 Model

```
php artisan make:model Students
```

設定一下 Model

```php=
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Students extends Model
{
    use HasFactory;

    protected $table = 'students';
}
```

接著為生成假資料建立 Factory

```
php artisan make:factory Students
```

設定 Factory

```php=
<?php

namespace Database\Factories;

use App\Models\Students;
use Illuminate\Database\Eloquent\Factories\Factory;

class StudentsFactory extends Factory
{
    protected $model = Students::class;

    public function definition()
    {
        return [
            'name' => $this->faker->name,
            'gender' => $this->faker->randomElement(['boy', 'girl']),
            'area' => $this->faker->randomElement(['taipei', 'tainan', 'kaohsiung']),
            'age' => $this->faker->numberBetween($min = 10, $max = 20)
        ];
    }
}

```

在 Seeder 中寫入生成假資料的程式碼

```php=
<?php

namespace Database\Seeders;

use App\Models\Students;
use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run()
    {
        Students::factory(100)->create();
    }
}

```

下指令生成假資料

```
php artisan db:seed
```

### 一般做法

接下來我會先寫在一般的狀況下該如何去實作上述的需求，但是我會將關注點放在下搜尋條件這段程式碼上，所以我不會去拆分關於 Repository、Service 之類的架構

首先我在 ```web.php``` 內新增我的 router

**web.php**

```php=
<?php

use App\Http\Controllers\StudentsController;
use Illuminate\Support\Facades\Route;

Route::get('students',[StudentsController::class,'getSearch']);

```

然後將搜尋的功能寫在 ```StudentsController```

**StudentsController.php**

```php=
<?php

namespace App\Http\Controllers;

use App\Models\Students;
use Illuminate\Http\Request;

class StudentsController extends Controller
{
    public function getSearch(Request $request)
    {
        $query_builder = Students::query();

        if ($request->has('gender')) {
            $query_builder->where('gender', '=', $request->gender);
        }

        if ($request->has('area')) {
            $query_builder->where('area', '=', $request->area);
        }

        if ($request->has('age')) {
            $query_builder->where('age', '=', $request->age);
        }

        $result = $query_builder->get();
    }
}

```

上方的搜尋邏輯應該相當好理解，我去確認有沒有帶該條件，當有條件的時候我就將條件加入到搜尋用的 ```$query_builder``` ，最後再將結果取出

類似的事情應該或多或少會出現在你我的搜尋程式碼中，如果我們專注在根據條件加入搜尋的程式碼這件事情上，這樣做其實很明顯的違反了單一職責原則(SRP)，對於決定搜尋的過程來說，至少會有三種不同的因素可以導致這個 function 必須去修改

那麼問題來了，該怎麼辦呢？這樣的狀況又跟我們本篇所要介紹的 Pipeline 又有什麼關聯呢？

### Refactor by Using Pipeline

還記得 Pipeline 帶給我們什麼樣的功能嗎？他讓我們可以定義一條職責鏈，並將某個物件放到這條職責鏈上，讓每個職責物件去處理他，如果遇到不能處理的狀況也可以直接傳遞給下一個職責物件

沒錯，從這樣的功能面上我們有了一個靈感，如果我讓 ```$query_builder``` 成為被傳遞的物件，而每一個判斷是否加入新的搜尋條件的邏輯變成職責物件的工作呢？

以下我們開始我們的重構

首先我需要職責物件，而這每一個職責物件必須要有相同的一個 ```handle()``` ，所以我先新增一個 interface

**IQueryFilter.php**
```php=
<?php


namespace App\QueryFilter;


use Closure;

interface IQueryFilter
{
    public function handle($query_builder, Closure $next);
}

```

接下來我要讓我的每一個職責物件都去 implements 這個 interface，而我剛剛說我要將每一個判斷是否加入新的搜尋條件的邏輯變成職責物件，所以底下我們新增 3 個職責物件用於判斷是否加入新的搜尋條件

**Gender.php**
```php=
<?php


namespace App\QueryFilter;


use Closure;

class Gender implements IQueryFilter
{

    public function handle($query_builder, Closure $next)
    {
        if (request()->has('gender')) {
            return $next($query_builder)->where('gender', '=', request()->gender);
        }
        return $next($query_builder);
    }
}

```

**Area.php**
```php=
<?php


namespace App\QueryFilter;


use Closure;

class Area implements IQueryFilter
{

    public function handle($query_builder, Closure $next)
    {
        if (request()->has('area')) {
            return $next($query_builder)->where('area', '=', request()->area);
        }
        return $next($query_builder);
    }
}

```

**Age.php**
```php=
<?php


namespace App\QueryFilter;


use Closure;

class Age implements IQueryFilter
{

    public function handle($query_builder, Closure $next)
    {
        if (request()->has('age')) {
            return $next($query_builder)->where('age', '=', request()->age);
        }
        return $next($query_builder);
    }
}

```

這邊大致上就是三個把判斷條件獨立出來放到各別的職責物件內，當某個搜尋的條件存在的時候，我們就將需要傳遞的物件加上條件，然後繼續往下傳遞

我們將職責物件給製作完了，接下來我們要來改寫 ```StudentsController``` 內的 ```getSearch(Request $request)```

**StudentsController.php**
```php=
<?php

namespace App\Http\Controllers;

use App\Models\Students;
use App\QueryFilter\Age;
use App\QueryFilter\Area;
use App\QueryFilter\Gender;
use Illuminate\Http\Request;
use Illuminate\Pipeline\Pipeline;

class StudentsController extends Controller
{
    public function getSearch(Request $request)
    {
        $pipeline = app(Pipeline::class)
            ->send(Students::query())
            ->through(
                [
                    Gender::class,
                    Area::class,
                    Age::class
                ]
            )->thenReturn();
        $result = $pipeline->get();
    }
}

```

這樣就完成了，我們將所有的條件判斷拆分成各個職責物件，並用 Pipeline 與職責鏈模式的概念去改寫了原本的流程，同樣的，在根據條件加入搜尋的程式碼這件事情上，符合了單一職責原則，我們讓決定搜尋條件的過程本身獨立了出來，決定搜尋條件的判斷也獨立了出來

今天如果我要增加或是刪減某個搜尋條件，我只要將 Pipeline 中間的職責物件新增或移除即可

同樣的如果我要更改某個搜尋條件，例如我想把年齡的條件改為輸入某個數字則可以取得大於該年齡的學生的話，我們也只要專注在修改年齡搜尋的職責物件上就可以

## Reference

- https://refactoring.guru/design-patterns/chain-of-responsibility
- https://www.youtube.com/watch?v=7XqEJO-wt7s&ab_channel=Coder%27sTape
