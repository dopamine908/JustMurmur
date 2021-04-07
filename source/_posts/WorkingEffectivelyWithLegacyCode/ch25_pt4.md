---
title: Working Effectively with Legacy Code - Ch25 - 解依賴技術 (4)
date: 2021-03-03 13:26:14
tags:
- 讀書會報告
- Legacy Code
- Test
- Refactoring
categories:
- 讀書會報告
cover: https://picsum.photos/id/178/800/600
---

# Working Effectively with Legacy Code - Ch25 - 解依賴技術 (4)

## 13. 連接替換

在物件導向的語言中有很多方法可以讓我們去替換物件，例如讓偽造的類別去實作與要替換的類別同一個物件，或是去繼承相同的基底類別，但在像是 C 這樣的程序式語言裡面卻沒辦法這麼做。

### before

例如下面這個函數，如果不用預處理手段，就完全沒辦法在編譯期將其替換為另一個函數

```c=
void account_deposit(int amount) 
```

### after

我們可以使用 **連接替換(Link Subsitution)** 來將他替換為另外一個函數。

首先要建立一個 **啞元函式庫(dummy library)** ，裡面放的假造函式簽章必須要跟原來的函式相同，並在偽造的函式內撰寫你用來感測的內容

```c=
void account_deposit(int amount) 
{
	struct Call *call =
		(struct Call *)calloc(1, sizeof (struct Cal));
	call->type = ACC_DEPOSIT; 
	call->arg0 = amount; 
	append(g_calls, call);
}
```

例如以上的程式碼，他建立了一個全域的列表，該列表裡包含了每一次函式被呼叫的相關資訊，在測試時我們便可以透過這個列表去確認函式是否照正確的順序被呼叫

## 14. 參數化建構子

如果你用建構子內去構築依賴關係，那最好的解依賴方式就是將建立的過程外部化

### before

像是下方例子，我們在 ```MailChecker``` 中建構 ```$Receiver``` ，而這種狀況在要加入測試的時候會相當棘手，我們很難去偽造 ```$Receiver```

**MailChecker**
```php=
class MailChecker
{
    private $Receiver;

    public function __construct()
    {
        $this->Receiver = new MailReceiver();
    }
}
```

### after

我們可以將建構的過程移動到另外一個 ```initialMailReceiver()``` 去，並將要綁定的實體給參數化，最後將建構子的內容也變成去委託 ```initialMailReceiver()``` 做生成的動作

這樣改的好處是我在測試中也可以透過 ```initialMailReceiver()``` 去將偽造的 ```MailReceiver``` 給注入以達到感測的效果，並且也不會影響到原本與該類別合作的所有依賴類別

**MailChecker**
```php=
class MailChecker
{
    private $Receiver;

    public function __construct()
    {
        $this->initialMailReceiver(new MailReceiver());
    }

    public function initialMailReceiver(MailReceiver $Receiver)
    {
        $this->Receiver = $Receiver;
    }
}
```

但這樣做也是有缺點的，今天當我們需要在建構子上加入新的參數的時候，可能會導致對參數類型的進一步依賴

## 15. 參數化方法

假設你有一個方法，該方法在內部建立了物件，而你需要透過替換物件來實作感測或分離，最簡單的辦法就是將在內部生成的物件改由從外部傳送進來

這邊使用的例子會跟書上有些不同，書上使用了一個在跑測試的狀況作為例子，但筆者覺得會有點混淆這個方法的主軸，所以將例子改為比較像產品程式碼的範例

### before

底下是一個畫圓的方法，我在方法內部生成了一張畫布 ```Canvas``` ，最後畫完了再將他回傳

**Draw**
```php=
class Draw
{
    public function drawCircle()
    {
        $Canvas = new Canvas();
        // draw circle on canvas
        return $Canvas;
    }
}
```

### after

我新增了另外一個 ```drawCircleOnCanvas()``` 方法，並將原本的實作內容複製過去，然後將我們想拿來實作感測的畫布 ```Canvas``` 改為用變數注入的方式，然後再將原本的 ```drawCircle()``` 的實作給轉發過去

**Draw**
```php=
class Draw
{
    public function drawCircle()
    {
        return $this->drawCircleOnCanvas(new Canvas());
    }

    public function drawCircleOnCanvas(Canvas $canvas)
    {
        // draw circle on canvas
        return $canvas;
    }
}
```

## 16. 樸素化參數

有時候我們會遇到將一個類別納入測試需要付出很大的成本的狀況，這時候為了獲得一些必要的分離，則可採用本章節的技術

### before

我們有一個 ```Sequence``` 類別，他代表音樂合成器中的一個音軌，我們期望加入一個 ```hasGapFor()``` 方法以便我們判斷有沒有可能在某個音軌的區間段放入另外一段音軌

理想的情況下我們應該要將 ```hasGapFor()``` 方法放入 ```Sequence``` 類別，並為他編寫測試去做感測，但不幸的是 ```Sequence``` 類別是一個依賴了過多類別的「黑洞類別」

**SequenceTest**
```php=
class SequenceTest extends TestCase
{
    /**
     * @test
     */
    public function hasGapFor()
    {
        //Arrange
        $baseSequence = new Sequence(); // 依賴過多
        $baseSequence->push_back(1);
        $baseSequence->push_back(0);
        $baseSequence->push_back(0);

        $pattern = new Sequence(); // 依賴過多
        $pattern->push_back(1);
        $pattern->push_back(2);

        //Assert
        $this->assertTrue($baseSequence->hasGapFor($pattern));
    }
}
```

**Sequence**
```php=
class Sequence
{
    public function __construct(/*假設在建構過程中依賴了相當多的類別*/)
    {
        // mass assign dependent
        // 過多依賴導致感測與分離困難
    }

    public function push_back(int $int)
    {
        // do something
    }
}
```

### after

為了在這種情況下加入新的功能並佐以測試，我們先新增一個新的 ```SequenceHelper``` 類別，並期望將 ```hasGapFor()``` 的實作委託到```SequenceHelper``` 類別中的 ```SequenceHasGapFor()``` 方法，當然我們必須要先在原本的 ```hasGapFor()``` 將一些依賴的狀況給整理好，讓```SequenceHelper``` 類別中的 ```SequenceHasGapFor()``` 方法實作可以具有可測試性

**SequenceTest**
```php=
class SequenceTest extends TestCase
{
    /**
     * @test
     */
    public function hasGapFor()
    {
        //Arrange
        $baseSequence = [1, 2, 3];
        $pattern = [1, 2];

        //Actual
        $SequenceHelper = new SequenceHelper();
        $actual = $SequenceHelper->SequenceHasGapFor($baseSequence, $pattern);
        //Assert
        $this->assertTrue($actual);
    }
}
```

**Sequence**
```php=
class Sequence
{
    public function __construct(/*假設在建構過程中依賴了相當多的類別*/)
    {
        // mass assign dependent
        // 過多依賴導致感測與分離困難
    }

    public function push_back(int $int)
    {
        // do something
    }

    public function hasGapFor(Sequence $pattern): bool
    {
        $baseRepresentation = $this->getDurationCopy();
        $patternRepresentation = $pattern->getDurationCopy();
        $SequenceHelper = new SequenceHelper();
        return $SequenceHelper->SequenceHasGapFor($baseRepresentation, $patternRepresentation);
    }

    private function getDurationCopy(): array
    {
        $result = [];
        // 處理資料的部分
        return $result;
    }
}
```

**SequenceHelper**
```php=
class SequenceHelper
{

    public function SequenceHasGapFor($baseSequence, $pattern): bool
    {
        // do something
    }
}
```

使用這樣的模式去修改後，確實可以用符合我們期待的成本加入測試並實作我們想要的特性了，但這個做法其實伴隨了不少缺點，以上述的例子來說

1. 暴露了 ```Sequence``` 類別的內部表示
    - ```getDurationCopy()``` 整理的過程會直接或間接透露```Sequence``` 類別的內部資訊
2. 令 ```Sequence``` 類別的實作更難以理解，因為我們將本應該在 ```Sequence``` 職責的方法拿到另外一個地方去撰寫
3. 寫了一些沒有測試覆蓋的程式碼
    - 上述例子來說，我們無法對 ```getDurationCopy()``` 加以測試
4. 在系統中存在重複的資料
    -  ```getDurationCopy()``` 的資料整理過程或許會將某些不應該有兩份的資料複製出去
5. 拖延了問題，事實上我們並沒有解開 ```Sequence``` 類別對於其他物件的依賴

> 作者認為這個方法算是不得已中的選擇，當構築依賴的成本真的相當昂貴而不值得馬上投資的時候則可以用此方法，如果狀況允許，可以優先使用別種手法已達到可加入測試又能解依賴的效果


## 小記

在 14 、 15 小節中提及的方法，筆者有將他針對 PHP 去做特化，所以對於方法的名稱會經過重新命名，而在有些程式語言中，如 Java 、 C++ 、 C# ，可以使用相同方法名稱但不同的簽章代表不同的方法，原文中的例子也是使用這樣的形式去達成參數化的目的，如下面範例

```java=
// before

class MailChecker {
    public MailChecker(int checkPeriodSecond){
    
    }
}

// after

class MailChecker {
    public MailChecker(int checkPeriodSecond){
    
    }
    
    public MailChecker(MailReceiver receiver, int checkPeriodSecond){
    
    }
}
```
