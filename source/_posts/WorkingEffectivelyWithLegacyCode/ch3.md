---
title: Working Effectively with Legacy Code - Ch3 - 感測和分離
date: 2020-11-25 20:56:15
tags:
- 讀書會報告
- Legacy Code
- Test
- Refactoring
categories: 
- 讀書會報告
cover: https://picsum.photos/id/168/800/600
---

# Working Effectively with Legacy Code - Ch3 - 感測和分離

通常來說，我們想要將測試安置到位，有兩個理由去解除依賴:

1. **感測**：有時候我們想測試的類別會對其他類別做出影響，我們就需要透過解除依賴去「感測」這些影響
2. **分離**：當我們無法將一部份的程式碼放入測試工具中去執行時，通常需要透過解除依賴將這塊程式碼「分離」出來

通常來說，這兩個面向是伴隨著彼此而出現的，試想因為一部分的程式碼無法被測試，所以我就將這部分給**分離**出來，而他獨立出來後，或許class之間的互動就成為了新的測試議題，我們便會想要**感測**分離出來的class是否被正確的影響。

本書提及了相當多的解除依賴技術，而在這之中最主要的一項技術是：**偽裝成合作者**(fake object)

## 偽裝成合作者

### 偽物件

偽物件就是那些在測試中，用來代替被測試的類別的「合作者」的物件。底下我們看一個例子。

假設在一個POS系統中有一個 Sale 類別(如下圖)，Sale中含有一個 scan 的 function，當我們要掃條碼的時候就會呼叫這個 function，並且呼叫之後會在收銀機上顯示商品的名稱及價格。

![](https://i.imgur.com/dih0v3p.png)


那麼我們該如何對這個function去作測試呢?如果收銀機的顯示也寫在這個 function 裡面(直接依賴於硬體)，那我們會很難感測功能是否正常，所以在這邊我們可以嘗試先將收銀機獨立出來，調整為下圖的樣子。

![](https://i.imgur.com/YFWHZlh.png)

我們新加入了一個 ArtR56Display 類別，該類別包含收銀機顯示器的所有功能程式碼。使用 ArtR56Display 的時候，我們只需要將我們想顯示的文字給 ArtR56Display 裡面負責顯示的 function 即可。

再來我們將 ArtR56Display 中負責顯示的 function 提取 interface 出來，改成下圖的設計

![](https://i.imgur.com/JviOp12.jpg)

如此一來，Sale 現在不只可以跟 ArtR56Display 合作，還可以跟任何有時作 Display 這個 interface 的類別合作，所以我們在這裡加上一個 FakeDisplay，讓他扮演偽物件的腳色。

底下示範了用 FakeDisplay 來取代 ArtR56Display 在測試中的寫法

```java=
public class Sale
{
    private Display diaplay;
    
    public Sale(Display display) {
        this.display = display;
    }
    
    public void sacn(String barcode) {
        ...
        String itemLine = /*do something get itemLine*/
        diplay.showLine(itemLine);
        ...
    }
}

public class FakeDisplay implements Display
{
    private String lastLine = '';
    
    public void showLine(String line) {
        lastLine = line;
    }
    
    public String getLastLine() {
        return lastLine;
    }
}

//Test
import junit.framwork.*
public class SaleTest extend TestCase
{
    FakeDisplay display = new FakeDisplay();
    Sale sale = new Sale(display);
    sale.scan('1');
    assertEquals('text that we expected', display.getLastLine())
}
```

### 偽物件的兩面性

把焦點放在 FakeDisplay 上，他提供了 showLine() 給 Sale 使用，同時也為測試提供了 getLastLine()，這使得我們在能夠如期使用功能的同時，還能夠去驗證。

### 偽物件手法的核心理念

偽物件的實作途徑有很多種，例如在物件導向的程式語言中，我們可以用像是上面的例子般的類別來實作。在非物件導向的程式語言中，得可以定義一個替代的函數來達成偽裝的目的。

### 仿物件

仿物件(mock object)，一種可以在內部進行斷言檢查的偽物件。如果我們將上面的測試改寫成用仿物件會長的像下面這樣。

```java=
import junit.framwork.*
public class SaleTest extend TestCase
{
    MockDisplay display = new MockDisplay();
    display.setException('showLine', 'item name and price');
    Sale sale = new Sale();
    sale.scan('1');
    display.verify();
}
```
