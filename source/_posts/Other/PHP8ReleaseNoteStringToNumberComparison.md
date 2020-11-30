---
title: PHP8 - Release Note - 字串與數字的比對行為修正
date: 2020-11-30 22:36:47
tags:
- PHP
- PHP8
- WTF PHP
categories: 
- 日常開發筆記
cover: https://picsum.photos/id/1042/800/600
---

# PHP8 - Release Note - 字串與數字的比對行為修正

## 前言

在 PHP8 官方的 release 網頁中，看到一個似乎更新風險相當大的一個改動，如圖

![](https://i.imgur.com/4gvt25C.png)

同樣的比對會回傳不一樣的結果，感覺很可能導致很多產品的比較或是套件內的比較失效。
我對於這樣的差異性感到好奇，照常理來說 PHP8 的結果會比較貼近於直覺，而 PHP8 以前為什麼會是那樣的結果呢? PHP8 做的這種更動，影響的範圍會不會比想像中還要更大呢?

所以我看了這個更新的 [RFC](https://wiki.php.net/rfc/string_to_number_comparison) ，以下會大致上以 RFC 的內容嘗試著去解釋差異性的前因後果。

## PHP8 之前的字串與數字比對

PHP8 以前的數字與字串比對是甚麼樣子的呢?是甚麼樣的運作會導致以下的結果呢?

```php=
0 == 'foobar'; // bool(true)
```

PHP 將比較的運算子分為兩種:

* Strict(嚴格) : === 、 !==
* Non-strict(寬鬆) :  == 、 != 、 > 、 >= 、 < 、 <= 、 <=>

在使用 Strict 的運算子的時候會進行型別的比較，而當我們將數字與字串使用 Non-strict 的運算子去進行比較的時候， PHP 會先將字串視為數字，再進行數字的比對。

```php=
/* step 1 */
0 == 'foobar';

/* step 2 , 因為 (int)'foobar' 為 (int)0 */
0 == 0 // true
```

這也就是為什麼在 PHP8 以前， ```0 == 'foobar'``` 會得到 ```true``` 的結果。
這個是數字與字串的例子，當然 PHP 中我們可以將很多不同型別的變數去進行比對，而各種不同的型別配對也都有自己不同的判別方式，詳細如下圖([來源](https://www.php.net/manual/en/language.operators.comparison.php))

![](https://i.imgur.com/wWmwXgL.png)

```
弱型別的語言(如:JS、PHP)在各種比對的運算上面都有一套自己預設的規則，
所以常常會出現一些不如預期的結果，
了解背後運行的原理對於理解這些結果來說相當有益
```
## PHP8 開始的字串與數字比對

PHP8 期望將比對的邏輯修正，當我們比較"數字字串"與數字的時候，可以使用數字的比對，而其餘的狀況， PHP8 則會將數字轉換為字串再去做字串比對。

> 數字字串 ( numeric string ) : 泛指像是 "42" 、 "0" 、 "0.0" 這樣子的字串

以下為在 PHP8 的修正前跟後的一些比較

| Comparison    | Before | After |
|---------------|--------|-------|
| 0 == "0"      | true   | true  |
| 0 == "0.0"    | true   | true  |
| 0 == "foo"    | true   | false |
| 0 == ""       | true   | false |
| 42 == "   42" | true   | true  |
| 42 == "42foo" | true   | false |

## PHP8 的字串與數字比對詳細規則

這份 RFC 中有明確定義甚麼樣的比較會用到這種新調整過的比較方式，如下

*  運算子 : <=> 、 == 、 != 、 > 、 >= 、 < 、 <=
*  array 相關的 function : in_array() 、 array_search() 、 array_keys() 當 $strict 設定為 false 的時候(預設)
*  排序相關的 function : sort() 、 rsort() 、 asort() 、 arsort() 、 array_multisort() 當 $sort_flags 設定為 SORT_REGULAR 的時候(預設)

底下使用 ```$int <=> $string``` 當範例，列出遇到各種狀況的時候 PHP8 會如何處理

* 如果 ```$string``` 是一個格式正確的 int numeric string ，則會將 ```$string``` 傳換為 int (下稱 ```$string_as_int```) 去做比較，並回傳 ```$int <=> $string_as_int``` 的結果
* 如果 ```$string``` 是一個格式正確的 float numeric string ，則會將 ```$string``` 傳換為 float (下稱 ```$string_as_float```) 去做比較，並回傳 ```(float)$int <=> $string_as_float``` 的結果
* 其餘的狀況，則會回傳 ```strcmp((string)$int, $string)``` 的結果

[strcmp()](https://www.php.net/manual/en/function.strcmp.php)

RFC 中有特別提到，當我的比較為 ```$string <=> $int``` (就是上面的反過來)的時候，則會回傳``` -($int <=> $string) ```的結果

而當我們想做 ```$float <=> $string``` 的比對時， PHP8 會如下處理

* 如果 ```$float``` 是 NAN, 則回傳 1 (此處應該是直接定義回傳)
* 如果 ```$string``` 是一個格式正確的 int numeric string，則會將 ```$string``` 傳換為 int (下稱 ```$string_as_int```) 去做比較，並回傳 ```$float <=> (float)$string_as_int``` 的結果
* 如果 ```$string``` 是一個格式正確的 float numeric string，則會將 ```$string``` 傳換為 float (下稱 ```$string_as_float```) 去做比較，並回傳 ```$float <=> $string_as_float``` 的結果
* 其餘的狀況，則會回傳 ```strcmp((string)$float, $string)``` 的結果

同樣的，如果我們做的比對為 ```$string <=> $float``` (就是上面的反過來)

* 如果 ```$float``` 是 NAN, 則回傳 1 (此處應該是直接定義回傳)
* 其餘的狀況，則會回傳``` -($float <=> $string) ```的結果

這邊有些更改前後的比較例子可以參考
```php=
// Before *and* after this RFC
var_dump(42 == "000042");        // true
var_dump(42 == "42.0");          // true
var_dump(42.0 == "+42.0E0");     // true
var_dump(0 == "0e214987142012"); // true

                         // Before | After | Type
var_dump(42 == "   42"); // true   | true  | well-formed
var_dump(42 == "42   "); // true   | false | non well-formed (*)
var_dump(42 == "42abc"); // true   | false | non well-formed
var_dump(42 == "abc42"); // false  | false | non-numeric
var_dump( 0 == "abc42"); // true   | false | non-numeric
// (*) Becomes well-formed if saner numeric strings RFC passes
```

> (*)特別註記了一個看起來還是怪怪的比對，上面的第8跟第9行會導致" 42" 不等於 "42 "的結果，而此處牽扯到另外一個RFC的內容([saner numeric strings RFC](https://wiki.php.net/rfc/saner-numeric-strings))，這個RFC主要指講述判斷變數是否為 numeric string 的實作所做出的修改，而上面的不自然之處則會在這個RFC通過之後將```var_dump(42 == "42   ")```改為true
> [saner numeric strings RFC](https://wiki.php.net/rfc/saner-numeric-strings) 裡面的狀態是呈現 Implemented in PHP 8.0 的，所以我想比對結果應該已經是 true 了，有待之後再去做驗證

## 特例

當然有些特殊的值的比對必須經由特別定義過後的規則去返回真假值，如下

```php=
                             // Before | After
var_dump(INF == "INF");      // false  | true
var_dump(-INF == "-INF");    // false  | true
var_dump(NAN == "NAN");      // false  | false
var_dump(INF == "1e1000");   // true   | true
var_dump(-INF == "-1e1000"); // true   | true
```

* INF
    * "INF" 跟 "-INF" 為 INF 跟 -INF 的字串表示，所以比對結果為 true
* NAN
    * 遵循 [IEEE-754](https://zh.wikipedia.org/wiki/IEEE_754) 的定義，NAN 不是數字，所以跟任何代表數字的值去比對都會回傳 false

## 設定

RFC 中也有提到，如果要維持 PHP8 以前的那種判斷模式，可以經由設定 php.ini 去做修改，範例如下

```php=
$float = 1.75;
 
ini_set('precision', 14); // Default
var_dump($float < "1.75abc");
// Behaves like
var_dump("1.75" < "1.75abc"); // true
 
ini_set('precision', 0); // Degenerate case
var_dump($float < "1.75abc");
// Behaves like
var_dump("2" < "1.75abc"); // false
```

## 延伸

在找資料的時候我找到了一位外國的工程師的 PPT (看不懂是甚麼國家的語言)，但他將一些 PHP 神奇的地方列了出來，其中除了本篇介紹的重點以外，也有許多相當特別的結果，附上[來源](https://pyrech.github.io/php-wtf/#/?_k=zyvjbp)給大家一同欣賞

## Reference

* [PHP8 release feature 官方](https://www.php.net/releases/8.0/en.php?lang=en#saner-string-to-number-comparisons)
* [PHP RFC: Saner string to number comparisons](https://wiki.php.net/rfc/string_to_number_comparison)
* [PHP RFC: Saner numeric strings](https://wiki.php.net/rfc/saner-numeric-strings)
* [IEEE 754](https://zh.wikipedia.org/wiki/IEEE_754)
* [Comparison Operators](https://www.php.net/manual/en/language.operators.comparison.php)
* [strcmp()](https://www.php.net/manual/en/function.strcmp.php)
* [PHP WTF](https://pyrech.github.io/php-wtf/#/?_k=zyvjbp)






