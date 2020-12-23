---
title: SOLID - SRP - 單一職責原則 （Single Responsibility Principle）
date: 2020-12-11 15:46:52
tags:
- SOLID
categories: 
- 日常開發筆記
cover: https://picsum.photos/id/594/800/600
---

# SOLID - SRP - 單一職責原則 （Single Responsibility Principle）

這篇的主角是 SOLID 原則中的 S : 單一職責原則 （Single Responsibility Principle），由 Robert C. Martin 在 1990 年代提出

## 定義

這邊我找到兩種定義，一種是在他出的書裡面提到的（[書的連結](https://www.amazon.com/Software-Development-Principles-Patterns-Practices/dp/0135974445)）

> A class should have one, and only one, reason to change.

另一個是我在 Martin 的個人部落格中提到的

> The Single Responsibility Principle (SRP) states that each software module should have one and only one reason to change.

兩種定義翻譯起來的意思其實大同小異

> 一個類別（軟體模組）應當只有唯一一個可以改變他的理由

這邊要特別提到，網路上有不少的文獻會將單一職責原則的定義解釋為「一個模組、類別或函式只做一件事情」，但我想這樣的解讀跟 Martin 原始敘述的語意是有些出入的。

## 職責？

定義中的敘述在解釋「職責」這個字眼用了 「 have one and only one reason to change 」去敘述

既然談到了改變的理由，那麼自然是各種狀況都可能導致修改發生，舉凡 bug 的修改、功能的修正、功能的新增都有可能造成各種改變的發生，而 Martin 也在他的文章內提到，這其實都取決於程式最終的服務對象（ who must the design of the program respond to? ），這聽起來也很抽象，簡單的說就是程式對於功能的實作，底下我們舉個例子來看看

```php=
Class Pay
{
    public function creditPay();
}
```

這是一個專門用來刷卡的 class ，如果今天情況相當單純，只有一個種類的銀行刷卡，那麼論改變來說，可能改變的原因只有該卡片刷卡行為的本身改變

但如果今天我們的 class 需要兼顧 A 卡跟 B 卡呢？

```php=
class Pay
{
    public function creditPay($card)
    {
        if ($card == 'A') {
            // 刷卡流程A
        }
        if ($card == 'B') {
            //刷卡流程 B
        }
    }
}
```

那麼可以改變他的原因就不止一種了

1. A 卡片刷卡流程的改變
2. B 卡片刷卡流程的改變

所以說，職責單不單一這件事情，我們還得考量他的實作以及他的合作對象，所以對於實踐單一職責原則的狀況可以說是每種人都有不同的解讀，並沒有一定的標準

像是我們上面講的兩個例子，我一開始的時候可能就真的只需要支援一張卡片的刷卡，那麼這樣的設計可以算是符合單一職責原則，但當後來需求增加了，我加入了第二張卡片，就變得不符合單一職責原則，所以說隨著功能的改變，對於有沒有符合單一職責原則的界定也需要重新的審視，加上適當的重構，才能夠盡量的保持符合單一職責原則的狀態。


<!-- ---
原始定義（Robert C. Martin）

什麼是職責（取決於實作

關注點

改變的理由只有一個，那麼跟他合作的人就是關鍵了，有可能因為跟他合作的方式不同導致改變的理由由一個變為多個

要做到改變的理由只有一個，要看實作的內容，可以對實作的內容去思考有什麼因素可以構成改變

這樣不就變成我一個class只能有一個function嗎？> 關注點

寫一個class，剛開始職責ＯＫ但是後來改變造成值則不分離，所以舊分離了
 -->
## Reference

- http://www.butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod
- http://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html
- https://www.jyt0532.com/2020/03/18/srp/
- https://medium.com/%E7%A8%8B%E5%BC%8F%E6%84%9B%E5%A5%BD%E8%80%85/%E4%BD%BF%E4%BA%BA%E7%98%8B%E7%8B%82%E7%9A%84-solid-%E5%8E%9F%E5%89%87-%E5%96%AE%E4%B8%80%E8%81%B7%E8%B2%AC%E5%8E%9F%E5%89%87-single-responsibility-principle-c2c4bd9b4e79
- https://drive.google.com/file/d/0ByOwmqah_nuGNHEtcU5OekdDMkk/view
- http://teddy-chen-tw.blogspot.com/2011/12/3.html
