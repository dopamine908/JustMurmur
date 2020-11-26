---
title: SQL達人的現場工作筆記 - Ch4 - 三元邏輯運算與NULL
date: 2020-11-04 10:14:29
tags:
- SQL
- 讀書會報告
categories: 
- 讀書會報告
cover: https://picsum.photos/id/1047/800/600
---

# SQL達人的現場工作筆記 - Ch4 - 三元邏輯運算與NULL

## Intro

一般來說，普遍上的程式語言都會有布林值的資料類型(bool、boolean)，可以用作二元邏輯的判斷或是運算，而 SQL 除了 true 、 false 以外還另外追加了 unknown 這個第三值。所以 SQL 體系中，我們稱這種邏輯體系為三元邏輯(three-valued logic)。

起因是由於關連式資料庫的體系中具有 NULL 這個值，所以運算的時候必須將其納入考慮，也因為三元邏輯的判斷有時候相當詭異，所以常常會伴隨著一些很怪的結果。

## 理論篇

### 你知道嗎？其實有兩種NULL。那到底是三元邏輯還是四元邏輯？

兩種 NULL 分別是

1. Unknown 未知值
    > 我們無法得知某個戴太陽眼鏡的人的眼睛顏色，這種叫做「未知值」，他有這個屬性，但我們不知道
3. Not Applicable, Inapplicable 不適性值
    > 像是冰箱的眼睛顏色，由於他本身並不賦予這種屬性，我們無論如何都無法得知，這種叫做「不適性值」
    > 常常在表格中看到的 N/A 符號，就是 Not Applicable 的縮寫

雖然說有兩種 NULL ，但 SQL 中其實只有一種（書中並未提及確切是哪一種，但依前後文的語意判斷大概是囊括上面兩種NULL）


### 為什麼不寫成「=NULL」？必須寫成「is NULL」？

**tb1_A**

| id | col_1 |
|----|-------|
| 1  | test  |
| 2  | NULL  |

當我在 tb1_A 資料表用下方 SQL 查詢的時候，會返回空的結果，而不是 id=2 的資料

```sql=
SELECT * FROM `tb1_A` WHERE `col_1` = NULL
```

我必須將 SQL 改為使用```IS NULL```才能如期找到 id=2 的資料

```sql=
SELECT * FROM `tb1_A` WHERE `col_1` IS NULL
```

這是因為 where 語句只會回傳條件判斷結果為 true 的列而已（例如：如果要找尋``` `col_1` = 'test' ```，那在 id=1 那列， SQL 評斷‘ col_1 的值等於 test ’為 true ，所以 id=1 則會被回傳），而如果比較結果為```false```或者```unknown```的話是不會被回傳的

```sql=
-- 以下的比對都會回傳 unknown 的結果
1 = NULL
2 > NULL
3 < NULL
4 <> NULL
NULL = NULL
NULL > NULL
NULL < NULL
NULL <> NULL
```

以 NULL 為對象的比對通常都不會為 true ，因為其實**NULL不是值也不是變數**， NULL 的意思是「這裡沒有值」，只是一個符號或是標誌的概念。能夠使用比對述詞的只有值，而 NULL 不是值，所以才總是回傳 unknown 。

### unknown、第三真假值

unknown 為導入 NULL 之後在三元邏輯運算中新增進來的**第三真假值**

而要注意的一點是，這裡的**unknown**跟NULL的**UNKNOWN**是不一樣的，**unknown**為SQL三元邏輯運算中的布林值，而**UNKNOWN**既非值也非變數，**unknown**是可以做比較的，而**UNKNOWN**既然非值，自然就不能做比較。

我們將這些放在 SQL 中去做真值運算會得到以下結果

```sql=
-- 單純的真假值比對
unknown = unknown <-- true

-- 簡單來說，這裡表示「NULL = NULL」
UNKNOWN = UNKNOWN <-- unknown
```

接下來我們來看看在 SQL 中的三元運算真值表

#### 三元運算真值表(NOT)

| X | NOT X |
|---|-------|
| t | f     |
| u | u     |
| f | t     |

#### 三元運算真值表(AND)

| AND | t | u | f |
|-----|---|---|---|
| t   | t | u | f |
| u   | u | u | f |
| f   | f | f | f |

#### 三元運算真值表(OR)

| OR | t | u | f |
|----|---|---|---|
| t  | t | t | t |
| u  | t | u | u |
| f  | t | u | f |

以上真假值是有規律的，其判斷順序如下

* AND 的情況：false > unknown > true
* OR 的情況：true > unknown > false

比較的時候，較大的會吃掉較小的

可參考以下比對

```sql=
(2<5) AND (5>NULL) --> unknown
(2>5) OR (5<NULL) --> unknown
(2<5) OR (5<NULL)  --> true
NOT (5<>NULL)      --> unknown
```

## 實踐篇

### 比對述詞與NULL其一-排中率不會成立

甚麼是排中率呢?

排中率(Law of excluded middle)

> In logic, the law of excluded middle (or the principle of excluded middle) states that for any proposition, either that proposition is true or its negation is true.
>> 排中率指出，任何一個論述，他只有可能是對的，或是他的反面是對的
> [wiki](https://en.wikipedia.org/wiki/Law_of_excluded_middle)

這種思維在二元邏輯中相當顯而易見，對於某一句話，我們可以很直覺地去認定他是對的，或者他不是對的(反面是對的意即他是不對的)，但在三元邏輯中，這種邏輯思維則會出現問題。

**Students**

| id | name | age |
|----|------|-----|
| 1  | A    | 22  |
| 2  | B    | 19  |
| 3  | C    | NULL|
| 4  | D    | 21  |

若遵循排中率的結果，也就是比較偏向我們直覺的思維，我們會認為下面的SQL可以將所以的人都取出，畢竟人只有分成「20歲的」跟「不是20歲的」

```sql=
SELECT * FROM `students` WHERE `age` = 20 OR `age` <> 20
```
實際上我們拿到的資料會像是這樣，少了 C 的資料
| id | name | age |
|----|------|-----|
| 1  | A    | 22  |
| 2  | B    | 19  |
| 4  | D    | 21  |

這是因為 SQL 在逐列比對吻合結果的過程中發生了下面的事情

```sql=
-- 1. C的年齡為NULL
SELECT * FROM `students` WHERE NULL = 20 OR NULL <> 20

-- 2. NULL的比對會得到unknown的結果
SELECT * FROM `students` WHERE unknown OR unknown

-- 3. 「unknown OR unknown」會得到unknown的結果
SELECT * FROM `students` WHERE unknown
```

由於 SQL 只會返回結果為 true 的列，所以 C 會被排除
如果我們想要將全部的資料取出的話我們應該改寫為這樣

```sql=
SELECT * FROM `students` WHERE `age` = 20 OR `age` <> 20 OR `age` IS NULL
```
### 比對述詞與NULL其二-CASE陳述式與NULL

在 CASE 陳述式中，判斷的結果為 true 才會返回 THEN 的結果

```sql=
CASE `col_1` 
    WHEN 1 THEN 'O'
    WHEN NULL THEN 'X' --只回得到unknown的結果
END
```

上面的 CASE 陳述式中只返回 'X' 的 CASE 是永遠進不去的，如果想要正常運作的話必須改為下面的寫法

```sql=
CASE  
    WHEN `col_1` = 1 THEN 'O'
    WHEN `col_1` IS NULL THEN 'X'
END
```

### NOT IN 與 NOT EXISTS 並不相等

NOT IN 及 NOT EXISTS 是常用的效能調教技巧，但有可能因為 NULL 的出現，使得我們得到不同的結果

**Class_A**

| id | name | age | city |
|----|------|-----|------|
| 1  | A1   | 22  | 台北  |
| 2  | A2   | 19  | 新北  |
| 3  | A3   | 21  | 台北  |

**Class_B**

| id | name | age | city |
|----|------|-----|------|
| 1  | B1   | 22  | 台北  |
| 2  | B2   | 23  | 台北  |
| 3  | B3   | NULL| 台北  |
| 4  | B4   | 18  | 桃園  |
| 5  | B5   | 20  | 桃園  |
| 6  | B6   | 19  | 新竹  |

如上面兩張資料表，如果我們想要得到「年齡與 B 班住台北的學生不一致的 A 班學生」， SQL 大概會是這樣寫

```sql=
SELECT * FROM `class_a` 
    WHERE `age` NOT IN 
    (SELECT `age` FROM `class_b` WHERE `city` = '台北')
```

我們期望可以獲得 A2 及 A3 這兩位同學的資料，但事實上我們只能得到搜尋為空的結果

接下來讓我們來看看 SQL 執行的階段到底發生了甚麼事情吧

```sql=
-- 1. 執行子查詢，得到年齡的List
SELECT * FROM `class_a` 
    WHERE `age` NOT IN (22, 23, NULL)

-- 2. 將 NOT IN 換成NOT 與 IN，進行同值轉換
SELECT * FROM `class_a` 
    WHERE NOT `age` IN (22, 23, NULL)
    
-- 3. 將 IN 換成 OR，進行同值轉換
SELECT * FROM `class_a` 
    WHERE NOT (
    (`age` = 22) OR (`age` = 23) OR (`age` = NULL)
    )

-- 4. 利用迪摩根定理進行同值轉換
SELECT * FROM `class_a` 
    WHERE 
    NOT (`age` = 22)
    AND NOT (`age` = 23) 
    AND NOT (`age` = NULL)

-- 5. 以 <> 代替 NOT 與 = 進行同值轉換
SELECT * FROM `class_a` 
    WHERE (`age` <> 22) AND (`age` <> 23) AND (`age` <> NULL)

-- 6. 因為NULL無法進行述值比對，則得到 unknown 的結果
SELECT * FROM `class_a` 
    WHERE (`age` <> 22) AND (`age` <> 23) AND unknown
    
-- 7. AND 的運算中夾帶 unknown ，依之前的真值表，我們無法得到 true 的結果
SELECT * FROM `class_a` 
    WHERE false 或是 unknown
```

因為每一列的資料都進行了這樣的比對流程，且都不為 true ，所以任何一筆資料都不會被取出來

那麼底下我們來看看可以得到正確的結果的 NOT EXISTS ， SQL 如下

```sql=
SELECT * FROM `class_a` 
    WHERE NOT EXISTS (
        SELECT * FROM class_b 
        WHERE class_a.age = class_b.age 
        AND class_b.city = '台北'
    )
```

一樣我們來看在含有 NULL 那列，中間的過程發生了甚麼事情

```sql=
-- 1. 子查詢與 NULL 的比對
SELECT * FROM `class_a` 
    WHERE NOT EXISTS (
        SELECT * FROM class_b 
        WHERE class_a.age = NULL 
        AND class_b.city = '台北'
    )

-- 2. NULL 的比對結果會得到 unknown
SELECT * FROM `class_a` 
    WHERE NOT EXISTS (
        SELECT * FROM class_b 
        WHERE unknown
        AND class_b.city = '台北'
    )

-- 3. AND 的運算結果出現 unknown ，結果就不會是 true
SELECT * FROM `class_a` 
    WHERE NOT EXISTS (
        SELECT * FROM class_b 
        WHERE false 或是 unknown
    )

-- 4. 子查詢不會回傳任何結果，所以 NOT EXIST 將會回傳 true
SELECT * FROM `class_a` WHERE true
```
### 限定述詞與NULL

這邊要探討使用 ALL 的情形

**Class_A**

| id | name | age | city |
|----|------|-----|------|
| 1  | A1   | 22  | 台北  |
| 2  | A2   | 19  | 新北  |
| 3  | A3   | 21  | 台北  |

**Class_B**

| id | name | age | city |
|----|------|-----|------|
| 1  | B1   | 22  | 台北  |
| 2  | B2   | 23  | 台北  |
| 3  | B3   | NULL| 台北  |
| 4  | B4   | 18  | 桃園  |
| 5  | B5   | 20  | 桃園  |
| 6  | B6   | 19  | 新竹  |

我們想取得「比 B 班住台北的任何學生都還要年輕的 A 班學生」， SQL 如下

<font color="#dd0000">這邊書上的資料表跟搜尋結果有點怪怪的</font>

```sql=
SELECT * FROM `class_a` 
    WHERE age < ALL ( SELECT age 
                        FROM class_b 
                        WHERE city = '台北')
```

我們期望得到 A2 、 A3 的資料，但結果依然是空白的，什麼也沒有回傳
所以我們一樣來看看過程中發生了什麼事情

```sql=
-- 1. 執行子查詢，取得年齡的 List
SELECT * FROM `class_a` 
    WHERE age < ALL ( 22, 23, NULL )

-- 2. 以 AND 讓 ALL 述詞進行同質轉換
SELECT * FROM `class_a` 
    WHERE (age <  22) AND (age < 23) AND (age < NULL)
    
-- 3. NULL 的比對結果會得到 unknown
SELECT * FROM `class_a` 
    WHERE (age <  22) AND (age < 23) AND unknown
    
-- 3. AND 的運算結果出現 unknown ，結果就不會是 true
SELECT * FROM `class_a` WHERE false 或者 unknown
```
### 限定述詞與極值函數並非同值

遇到上一個例子的狀況，或許會有人想說那就使用極值函數去取代（如 MIN）

```sql=
SELECT * FROM `class_a` 
    WHERE age < ALL ( SELECT MIN(age) 
                        FROM class_b 
                        WHERE city = '台北')
```

搜尋出來的結果是符合我們預期的沒錯，只會選出 A2 、 A3 的資料

| id | name | age | city |
|----|------|-----|------|
| 2  | A2   | 19  | 新北  |
| 3  | A3   | 21  | 台北  |

這是因為，當我們使用極值函數在運算時，會排除 NULL 的資料項目，乍看之下可能會覺得使用極值函數就可以迴避掉 NULL 帶來的問題

但試想如果今天 class_b 沒有人住在台北的狀況呢？如下表

**Class_B_taipei_no_person**

| id | name | age | city |
|----|------|-----|------|
| 1  | B4   | 18  | 桃園  |
| 2  | B5   | 20  | 桃園  |
| 3  | B6   | 19  | 新竹  |

```sql=
-- 0. 上方的 SQL 則寫成這樣
SELECT * FROM `class_a` 
    WHERE age < ALL ( SELECT MIN(age) 
                        FROM class_b_taipei_no_person 
                        WHERE city = '台北')
                        
-- 1. 極值函數將傳回 NULL
SELECT * FROM `class_a` WHERE age < NULL

-- 2. NULL 的比對結果會得到 unknown
SELECT * FROM `class_a` WHERE unknown
```

則我們會得到一片空白的搜尋結果

當我們想比對的對象不存在的時候，有的人覺得要回傳所有的資料，有的人覺得要回傳空的資料，要使用哪一種做法是根據情況而定。但要注意在這種狀況之下，使用極值函數是不會回傳任何值的。

### 彙總函數與NULL

**彙總函數、聚集函數(aggregate function) ：AVG()、COUNT()、MAX()、MIN()、SUM()**

其實不只極值函數， COUNT 之外的彙總函數都會有上面例子那樣的結果，例如

```sql=
SELECT * FROM `class_a` 
    WHERE age <  ( SELECT AVG(age) 
                        FROM class_b_taipei_no_person 
                        WHERE city = '台北')
```

這個也會回傳完全空白的結果

## 總結

1. NULL 不是值
2. 不是值，所以無法套入述詞（比對結果會為 unknown）
3. 勉強套用只會產生 unknown 的結果
4. 當邏輯運算出現 unknown ， SQL 會以有違直覺的方式去運算
5. 為了解決這個問題，我們必須分段觀察 SQL 的運算過程

**最重要的是，解決 NULL 問題的最佳策略，莫過於在資料表上加上 NOT NULL 限制**
