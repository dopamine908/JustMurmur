---
title: Working Effectively with Legacy Code - Ch25 - 解依賴技術 (5)
date: 2021-03-12 17:36:24
tags:
- 讀書會報告
- Legacy Code
- Test
- Refactoring
categories:
- 讀書會報告
cover: https://picsum.photos/id/190/800/600
---

# Working Effectively with Legacy Code - Ch25 - 解依賴技術 (5)

## 17. 特性提升

有時候我想為類別中的某些方法加入測試，但我在測試中必須要構築的依賴卻又跟我想測試的方法毫無關聯，這樣我勢必得花上大量的成本去對付這些依賴，而這樣做對測試目標的方法毫無幫助。

當然，這個時候我們可以透過「暴露靜態方法」或是「分解出方法物件」去應對，本節介紹另外一個可以嘗試的方法

### before

我們有一個 ```Scheduler``` 類別，我們想要修改 ```getDeadTime()``` ，但不關心 ```updateScheduleItem()``` ，而 ```updateScheduleItem()``` 的實作就是伴隨著某些麻煩的依賴，與 ```getDeadTime()``` 的實作完全沒有關聯

**Scheduler**
```php=
class Scheduler
{
    /**
     * @var array
     */
    private $items;

    public function updateScheduleItem(ScheduleItem $item)
    {
        // do something...
        $this->validate($item);
        // do something...
    }

    private function validate(ScheduleItem $item)
    {
        // do something...
    }

    public function getDeadTime()
    {
        foreach ($this->items as $item) {
            // some process...
            $this->notSharedItem($item);
            $this->getClockTime();
            $this->getStandardFinish($item);
            // some process...
        }
    }

    private function notSharedItem($item)
    {
        // do something...
    }

    private function getClockTime()
    {
        // do something...
    }

    private function getStandardFinish($item)
    {
        // do something...
    }
}
```

### after

首先我們要先新增一個 ```SchedulingService``` 類別，並將我們會使用到的方法群給提升到 ```SchedulingService``` 類別，再讓原本的 ```Scheduler``` 類別去繼承 ```SchedulingService``` 類別

**SchedulingService**
```php=
class SchedulingService
{

    /**
     * @var array
     */
    protected $items;

    public function getDeadTime()
    {
        foreach ($this->items as $item) {
            // some process...
            $this->notSharedItem($item);
            $this->getClockTime();
            $this->getStandardFinish($item);
            // some process...
        }
    }

    protected function getStandardFinish($item)
    {
        // do something...
    }

    protected function getClockTime()
    {
        // do something...
    }

    protected function notSharedItem($item)
    {
        // do something...
    }
}
```

**Scheduler**
```php=
class Scheduler extends SchedulingService
{

    public function updateScheduleItem(ScheduleItem $item)
    {
        // do something...
        $this->validate($item);
        // do something...
    }

    private function validate(ScheduleItem $item)
    {
        // do something...
    }

}
```

此處在書上是將 ```SchedulingService``` 類別設定為一個抽象類別，但因為書上目前的做法只是將方法實作移動過去，而又因為 PHP 的抽象類別必須至少含有一個抽象方法，所以這樣的做法在 PHP 的規則下是無法將 ```SchedulingService``` 類別設定為一個抽象類別的，故此處先維持普通的類別，不過事實上讓他成為一個抽象類別會是比較好的選擇，如此一來 ```SchedulingService``` 類別的介面將會固定下來，結構上會更穩固些

接下來在測試的程式碼中我就可以用另外一個類別去代替 ```Scheduler``` ，我們把它稱為 ```TestingSchedulingService``` ，在 ```TestingSchedulingService``` 中我們除了可以安插方便我們做測試的輔助方法，還可以對某些需要偽造的方法去覆寫

**TestingSchedulingService**
```php=
class TestingSchedulingService extends SchedulingService
{
    // 輔助測試的方法
    public function addItem(ScheduleItem $item)
    {
        $this->items[] = $item;
    }

    protected function getClockTime()
    {
        //或是可以經由覆寫去輔助
    }

}
```

從基礎設計的角度來說，這樣的做法其實不甚理想，我們為了方便測試，有可能令本來應該是一組的特性跨越了兩個類別（父類別與子類別），這種方式其實是有可能帶來混亂的。

## 18. 依賴下推

當你的待測方法有許多依賴，通常我們有幾種方式可以選，包括之後會提到的「子類別化並覆寫方法」或是重複的運用介面提取來達成將依賴分離出來的目的，而這邊介紹另外一種方法：「依賴下推」

「依賴下推」的實作，首先要把目標類別設為抽象，然後建立一個子類別，最後把依賴的部分下推倒子類別中

### brfore

底下我們有一個 ```OffMarketTradeValidator``` 類別，我們想對 ```isValid()``` 加入測試，以便修改

但他的過程中會用到 ```showMessage()``` 方法，該方法是依賴於顯示相關的 API ，實際上與 ```isValid()``` 的驗證過程並不是那麼相關

**OffMarketTradeValidator**
```php=
class OffMarketTradeValidator
{
    private function showMessage()
    {
        // 依賴了UI介面的東西，但他不是 isValid() 驗證邏輯的重點
    }

    public function isValid()
    {
        // do something
        $this->showMessage();
        // do something
    }
}
```
### after

所以這邊我們將原本的 ```OffMarketTradeValidator``` 類別設為抽象，並把不好處理依賴的 ```showMessage()``` 方法給下推到我們新增的子類別 ```WindowsOffMarketTradeValidator``` 中

**OffMarketTradeValidator**
```php=
abstract class OffMarketTradeValidator
{
    abstract protected function showMessage();

    public function isValid()
    {
        // do something
        $this->showMessage();
        // do something
    }
}
```

**WindowsOffMarketTradeValidator**
```php=
class WindowsOffMarketTradeValidator extends OffMarketTradeValidator
{

    public function showMessage()
    {
        // 依賴了UI介面的東西，但他不是 isValid() 驗證邏輯的重點
    }
}
```

至此，當我們想測試的時候我們可以建立一個測試用的子類別 ```TestingOffMarketTradeValidator```

**TestingOffMarketTradeValidator**
```php=
class TestingOffMarketTradeValidator extends OffMarketTradeValidator
{

    protected function showMessage()
    {
        // TODO: Implement showMessage() method.
    }
}
```

這樣一來我們就擁有了一個可測試，但又不依賴任何 UI 介面的類別

不過要注意的是，這樣的繼承絕對不是一個最理想的方案，甚至很多時候這樣的方式會高機率的違反開放封閉原則，但這樣的方式最大的好處是可以協助我們將一些邏輯納入測試，這是通往理想的架構設計的第一步。

## 19. 換函數為函數指標

在程序式語言解依賴，沒有物件導向那麼多選擇，但面對規模比較小的依賴，前面介紹的「連接替換」或者「定義補全」又有點殺雞用牛刀，這時候可以嘗試看看本節要介紹的「換函數為函數指標」

### before

這是一個網路應用程式，封包資訊被保存在一個線上資料庫中，你透過以下的呼叫與該資料庫互動

**db.h**
```c=
void db_store(struct receive_record *record, struct time_stamp receive_time);
```

### after

首先我們找出想要替換為指標函數的函數

**db.h**
```c=
void db_store(struct receive_record *record, struct time_stamp receive_time);
```

接著宣告一個與他同名的指標函數

**db.h**
```c=
void db_store(struct receive_record *record, struct time_stamp receive_time);
void (*db_store)(struct receive_record *record, struct time_stamp receive_time);
```

再來重新命名原函數

**db.h**
```c=
void db_store_production(struct receive_record *record, struct time_stamp receive_time);
void (*db_store)(struct receive_record *record, struct time_stamp receive_time);
```

然後在 C 語言的原始檔案中，初始化指標函數

**main.c**
```c=
extern void db_store_production(struct receive_record *record, struct time_stamp receive_time);

void initializeEnvironment() {
    db_store = db_store_production;
}

int main(int ac, char **av) { 
    initializeEnvironment(); 
    // doing other things
}
```

最後找到 ```db_store()``` 函數的定義，把它重新命名為 ```db_store_production()``` 就可以編譯和測試了

**db.c**
```c=
void db_store_production(struct receive_record *record, struct time_stamp receive_time) 
{ 
    // do something
}
```
## 20. 以獲取方法替換全域參照

想要解開類別中對於全域變數的依賴，方法之一就是在該類別中，對於全域變數的使用引入一個存取方法。有了這些方法，就可以透過「子類別化並覆寫方法」來讓他們返回測試用的物件，以便我們加入測試。

### before

底下有個 ```RegisterSale``` 類別，其中 ```Inventory``` 類別被當作全域的變數去使用（雖然他只是一個靜態方法的呼叫，但我們的使用意圖上還是把它偷偷當作全域的方法去使用）

**RegisterSale**
```php=
class RegisterSale
{
    /**
     * @var Item;
     */
    private $item;

    public function addItem(Barcode $code)
    {
        $newItem = Inventory::getInventory()->itemForBarcode($code);
        $this->item->add($newItem);
    }
}
```

### after

我們對 ```Inventory``` 的存取另外提取了一個 protected 方法，如此一來我們想要在測試的子類別中覆寫他也是可行的

**RegisterSale**
```php=
class RegisterSale
{
    /**
     * @var Item;
     */
    private $item;

    public function addItem(Barcode $code)
    {
        $newItem = $this->getInventory()->itemForBarcode($code);
        $this->item->add($newItem);
    }

    protected function getInventory(): Inventory
    {
        return Inventory::getInventory();
    }
}
```

如果我們今天想要偽造的是 ```Inventory``` 類別，我們也可以透過繼承的方式去偽造

**FakeInventory**
```php=
class FakeInventory extends Inventory
{
    public static function getInventory(): Inventory
    {
        // over write
    }

    public static function itemForBarcode($code): Inventory
    {
        // over write
    }

}
```
