---
title: Working Effectively with Legacy Code - Ch25 - 解依賴技術 (6)
date: 2021-03-27 11:27:13
tags:
- 讀書會報告
- Legacy Code
- Test
- Refactoring
categories:
- 讀書會報告
cover: https://picsum.photos/id/61/800/600
---

# Working Effectively with Legacy Code - Ch25 - 解依賴技術 (6)


## 21. 子類別化並覆寫方法

「子類別化並覆寫方法」是物件導向程式中姐依賴的核心技術，這個手法就是讓我們可以在測試環境下，利用繼承將我們不關心的行為架空，或是覆寫你在乎的行為讓他可以達到感測的目的，事實上這本書中其他的很多手法都是這個手法的變形。

### before

```MessageForwarder``` 上有許多方法，假設我們今天要加入測試的方法是 ```sendMessage()``` ，而他的過程中會呼叫一個私有方法 ```createForwardMessage()``` 去生成一個新的訊息，但過程中用到了 ```Session``` 物件，可是測試過程我們沒辦法對真正的 ```Session``` 去做操作。

**MessageForwarder**
```php=
class MessageForwarder
{

    public function sendMessage()
    {
        $session = new Session();
        $message = $this->createForwardMessage($session, new Message());
        // do something
        // do something
        // do something
        $this->send($message);
    }

    private function createForwardMessage(Session $session, Message $message): Message
    {
        $forward = new MimeMessage ($session);
        $forward->setFrom();
        $forward->addRecipients();
        $forward->setSubject();
        $forward->setSentDate();
        $forward->addHeader();
        $forward = $this->buildForwardContent($message, $forward);
        return $forward;
    }

    private function buildForwardContent(Message $message, MimeMessage $forward): Message
    {
        // build forward content
        return $forward;
    }

    private function send(Message $message)
    {
        // send message
    }
}
```
### after

為了讓 ```createForwardMessage()``` 的過程與 ```Session``` 解依賴，我們將 ```createForwardMessage()``` 設定為 ```proteted``` ，並新增一個 ```MessageForwarder``` 的子類別 ```TestingMessageForwarder``` ，然後將 ```createForwardMessage()``` 給覆寫，讓他直接回傳我們偽造過後的 ```FakeMessage```

**MessageForwarder**
```php=
class MessageForwarder
{

    public function sendMessage()
    {
        $session = new Session();
        $message = $this->createForwardMessage($session, new Message());
        // do something
        // do something
        // do something
        $this->send($message);
    }

    protected function createForwardMessage(Session $session, Message $message): Message
    {
        $forward = new MimeMessage ($session);
        $forward->setFrom();
        $forward->addRecipients();
        $forward->setSubject();
        $forward->setSentDate();
        $forward->addHeader();
        $forward = $this->buildForwardContent($message, $forward);
        return $forward;
    }

    private function buildForwardContent(Message $message, MimeMessage $forward): Message
    {
        // build forward content
        return $forward;
    }

    private function send(Message $message)
    {
        // send message
    }
}
```

**TestingMessageForwarder**
```php=
class TestingMessageForwarder extends MessageForwarder
{
    protected function createForwardMessage(Session $session, Message $message): Message
    {
        return new FakeMessage();
    }
}
```

在這個新建立的 ```TestingMessageForwarder``` 子類別中，我們可以架空我們用不到的功能，並且可以達成感測與分離的目的

## 22. 替換實例變數

有些語言不支援在建構子中呼叫虛擬函式，例如 C++

### before

**Pager**
```cpp=
class Pager 
{
public:
    Pager() { 
        reset();
        formConnection(); 
    }

    virtual void formConnection() {
        assert(state == READY);
        // nasty code that talks to hardware here ...
    }
	
    void sendMessage(const std::string& address, 
                     const std::string& message) {
        formConnection();
    }
 }
```

**TestingPager**
```cpp=
class TestingPager : public Pager 
{
public:
    virtual void formConnection() {
    } 
}
```

**Test**
```cpp=
TEST(messaging,Pager) {
    TestingPager pager;
    pager.sendMessage("5551212", "Hey, wanna go to a party? XXXOOO");
    LONGS_EQUAL(OKAY, pager.getStatus()); 
}
```

以上是一個 C++ 的例子，我對它實施了「子類別化並覆寫方法」的手段，但這種方式在 C++ 中會出問題，當我們將 ```TestingPager``` 給實例化的時候， ```Pager``` 的 ```formConnection()``` 會被執行，而不是 ```TestingPager``` 的 ```formConnection()``` ，這使得我法成功運用「子類別化並覆寫方法」去取代原本的 ```formConnection()``` 行為。

### after

如果在建構子內使用了某個物件，而我們想要將他替換掉，可以採用「替換實例變數」手法

```cpp=
BlendingPen::BlendingPen() 
{
	setName("BlendingPen");
	m_param = ParameterFactory::createParameter("cm", "Fade", "Aspect Alter"); 
	m_param->addChoice("blend");
	m_param->addChoice("add"); m_param->addChoice("filter");

	setParamByName("cm", "blend"); 
}
```

```BlendingPen``` 的建構子透過一個工廠來建立 ```Parameter``` 物件，我們可以新增一個方法，直接替換掉 ```Parameter```

```cpp=
void BlendingPen::supersedeParameter(Parameter *newParameter) 
{
	delete m_param;
	m_param = newParameter; 
}
```

在測試的時候，我們可以根據需要去建立 ```BlendingPen``` 物件，並在需要放入感測物件的時候呼叫 ```supersedeParameter()``` 方法。

## 23. 模板重定義

如果你的語言支援泛型以及類型別名，則可以使用「模板重新定義」來解依賴

### before

**AsyncReceptionPort.h**
```cpp=
class AsyncReceptionPort 
{
private:
    CSocket m_socket; 
    Packet m_packet; 
    int m_segmentSize; 
    ...
public:
    AsyncReceptionPort();
    void Run();
    ... 
};
```

**AsynchReceptionPort.cpp**
```cpp=
void AsyncReceptionPort::Run() 
{
    for(int n = 0; n < m_segmentSize; ++n) {
        int bufferSize = m_bufferMax; 
        if (n = m_segmentSize - 1)
            bufferSize = m_remainingSize; 
        m_socket.receive(m_receiveBuffer, bufferSize); 
        m_packet.mark(); 
        m_packet.append(m_receiveBuffer,bufferSize); 
        m_packet.pack();
    }
    m_packet.finalize(); 
}
```

對於以上的程式碼如果我想要修改 ```Run()``` 並加入測試，就必須面對 Socket 的依賴

### after

在 C++ 中我們可以透過把 ```AsyncReceptionPort``` 做成一個類別模板來避開這個問題

**AsynchReceptionPort.h**
```cpp=
template<typename SOCKET> class AsyncReceptionPortImpl 
{
private:
    SOCKET m_socket; 
    Packet m_packet; 
    int m_segmentSize; 
    ...
public: 
    AsyncReceptionPortImpl();
    void Run();
    ...
};

template<typename SOCKET>
void AsyncReceptionPortImpl<SOCKET>::Run() 
{
    for(int n = 0; n < m_segmentSize; ++n) { 
        int bufferSize = m_bufferMax;
        if (n = m_segmentSize - 1)
            bufferSize = m_remainingSize; 
        m_socket.receive(m_receiveBuffer, bufferSize); 
        m_packet.mark(); 
        m_packet.append(m_receiveBuffer,bufferSize); 
        m_packet.pack();
    }
    m_packet.finalize(); 
}
typedef AsyncReceptionPortImpl<CSocket> AsyncReceptionPort;
```

有了這個步驟，我們就可以在測試檔案中用一個 ```FakeSocket``` 來替換 ```Socket```

**TestAsynchReceptionPort.cpp**
```cpp=
#include "AsyncReceptionPort.h"

class FakeSocket 
{
public:
    void receive(char *, int size) { ... } 
};


TEST(Run,AsyncReceptionPort) 
{
    AsyncReceptionPortImpl<FakeSocket> port;
... 
}
```
## 24. 文字重定義

一些直譯式語言提供了很不錯的解依賴途徑，在直譯的時候，方法可以被重新定義

### before

假設我們想測試 ```Account``` 裡面的 ```deposit()``` ，但過程中的 ```report_deposit()``` 卻不是我們關心的對象

**Account.rb**
```ruby=
class Account
    def report_deposit(value) 
        # doing something
    end
    def deposit(value) 
        @balance += value 
        report_deposit(value)
    end
    def withdraw(value) 
        @balance -= value
    end 
end
```

### after

我們想要架空 ```report_deposit()``` 的行為，那麼我們只需要在測試檔案中重新定義

**AccountTest.rb**
```ruby=
require "runit/testcase" 
require "Account"
class Account
    def report_deposit(value) 
        # over write
    end
end

# tests start here
class AccountTest < RUNIT::TestCase
    # test content
end
```
