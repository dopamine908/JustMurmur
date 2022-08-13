---
title: 單元測試的藝術 - Ch1 單元測試基礎 - Sec. 1.5 ~ 1.8
date: 2021-02-24 13:42:52
tags:
- Unit Test
- Test
- 讀書會報告
  categories:
- 讀書會報告
  cover: https://picsum.photos/id/153/800/600
---

- # 單元測試的藝術 - Ch5 隔離框架 - Sec. 5.4 ~ 5.7

## 5.4 測試事件相關的活動

事件是雙向的，所以可以從兩個方面去進行測試

- 測試監聽事件的那一方
- 測試觸發事件的那一方

### 測試事件監聽者

我們想檢查**一個物件是否有註冊到另一個物件的事件**

如果物件被註冊到另外一個物件，我們比較直覺的會想到去檢查物件內部狀態使否有該物件被註冊，但作者認為這是比較不好的方式

比較好的選擇是，檢查監聽物件是否對於發生的事件做出某種反應

```csharp=
class Presenter
{
    private readonly IView _view;
    
    public Presenter(IView view)
    {
        _view = view;
        this._view.Loaded += OnLoaded; //註冊事件(OnLoaded)
    }
    
    private void OnLoaded()
    {
        _view.Render("Hello World");
    }
}

public interface IView
{
    event Action Loaded;
    void Render(string text);
}

//------ TESTS
[TestFixture]
public class EventRelatedTests{
    [Test]
    public void ctor_WhenViewIsLoaded_CallsViewRender()
    {
        var mockView = Substitute.For<IView>();
        Presenter p = new Presenter(mockView);
        mockView.Loaded += Raise.Event<Action>();
        mockView.Received() //使用 NSubstitute 觸發事件，並驗證 view.Render() 是否被呼叫
            .Render(Arg.Is<string>(s => s.Contains("Hello World")));
    } 
}
```

請留意下列幾點：

- 這個模擬物件同時也是一個虛設常式（你用它來模擬一個事件）
- 要觸發一個試驗，你需要在測試中註冊這個事件

底下給了另外一個情境，現在我們有兩個相依物件，一個 ```Log``` 物件，一個 ```
View``` 物件。下列程式碼所呈現的測試，確保了 ```Presenter``` 在虛設常式得到一個物件時，寫入 ```Log```

```csharp=
[Test]
public void ctor_WhenViewhasError_CallsLogger()
{
    var stubView = Substitute.For<IView>();
    var mockLogger = Substitute.For<ILogger>();
    Presenter p = new Presenter(stubView, mockLogger);
    stubView.ErrorOccured += Raise.Event<Action<string>>("fake error"); // 模擬錯誤發生
    mockLogger.Received() // 使用模擬物件確認 Log 有被正確呼叫
        .LogError(Arg.Is<string>(s => s.Contains("fake error")));
}
```

請注意，這個地方你使用了虛設常式觸發事件，然後再用一個模擬物件去驗證是否正確呼叫 Log 的服務

### 測試事件是否觸發

測試事件的另外一個簡單的方式，就是在測試方法中使用一個匿名委派，手動註冊這個方法。

```csharp=
[Test]
public void EventFiringManual()
{
    bool loadFired = false;
    SomeView view = new SomeView();
    view.Load+=delegate
                   {
                     loadFired = true;
                   };
    view.DoSomethingThatEventuallyFiresThisEvent();
    Assert.IsTrue(loadFired);
}
```

這個委派只記錄了這個事件是否被觸發
## 5.5 現有的 ~~.NET~~ PHP 隔離框架

原本書上的內容是在介紹 .NET 有什麼測試框架，但因為筆者自己是使用 PHP ，所以這邊我去找了人家列出來相對多人使用的 PHP 測試框架

- PHPUnit (Unit Test)
- Codeception (Unit Test、Integration Test)
- Behat (Integration Test、Behaviour Test)
- PhpSpec (Integration Test、Behaviour Test)
- Storyplayer (Unit Test、Integration Test、Behaviour Test)
- Peridot (Integration Test、Behaviour Test)
- Atoum (Unit Test)
- Kahlan (Integration Test、Behaviour Test)
- Selenium (Integration Test、Behaviour Test)

## 5.6 隔離框架的優缺點

### 優點

- 更容易驗證參數
- 更容易驗證一個方法被多次呼叫
- 更容易建立假物件

### 缺點
- 測試程式可讀性變差
- 驗證了錯誤的東西
    - 容易驗證是否正確呼叫某個方法，但卻不能保證該方法的行為是正確的
- 一個測試中有多個模擬物件
    - 多個模擬物件就意味著你驗證不止一件事情
- 過度指定的測試
    - 如果在使用模擬物件的時候有過多的預期，測試可能就會變得越發脆弱

#### 避免過度指定
- 盡量使用非嚴格模擬物件（後面章節會詳細講解）
- 盡量使用虛設常式，而非模擬物件
- 極力避免將虛設常式當作模擬物件使用
    - 但也有無法避免的狀況，例如 5.4 提到的事件的測試

## 5.7 小結

如果有超過 5% 的測試使用模擬物件（而非虛設常式），那你可能就過度指定了。
在測試中如果過度指定，會導致測試程式變得不易讀、易失敗、脆弱性。
