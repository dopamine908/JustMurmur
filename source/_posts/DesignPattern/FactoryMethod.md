---
title: Design Patterns - Creational - Factory Method Pattern 工廠方法模式
date: 2020-07-22 15:14:50
tags:
- Design Patterns
- Creational Patterns
categories: 
- 讀書會報告
cover: https://picsum.photos/id/133/800/600
---

# 設計模式 - Factory Method Pattern 工廠方法模式

## Intro

假設我們要製作一個計算機，如果使用間單工廠的話，程式碼應該長得如下

```php=

class OperationFactory
{
    public function createOperator($type): Operation
    {
        switch ($type) {
            case'+':
                return new Add();
                break;
            case'-':
                return new Minus();
                break;
        }
    }
}

class Add extends Operation
{
    public function operate(float $num1, float $num2): float
    {
        return $num1 + $num2;
    }
}

class Minus extends Operation
{
    public function operate(float $num1, float $num2): float
    {
        return $num1 - $num2;
    }
}

abstract class Operation
{
    abstract public function operate(float $num1, float $num2): float;
}

```

```php=

$OperatorFactory = new OperationFactory();
$Operator = $OperatorFactory->createOperator('+');
$Operator->operate(1, 2);

```

假設我們因應需求要新增一個運算符號‘乘法’，那我除了新增一個Product類別以外，我勢必還需要對OperationFactory作出修改
如此一來，這種修改方式其實違反了‘開放封閉原則’

## 實作

將上面的例子改為工廠方法模式

```php=

abstract class Operation
{
    abstract public function operate(float $num1, float $num2): float;
}

class Add extends Operation
{
    public function operate(float $num1, float $num2): float
    {
        return $num1 + $num2;
    }
}

class Minus extends Operation
{
    public function operate(float $num1, float $num2): float
    {
        return $num1 - $num2;
    }
}

class Product extends Operation
{
    public function operate(float $num1, float $num2): float
    {
        return $num1 * $num2;
    }
}

interface IOperationFactory
{
    public function createOperator(): Operation;
}

class AddFactory implements IOperationFactory
{
    public function createOperator(): Operation
    {
        return new Add();
    }
}

class MinusFactory implements IOperationFactory
{
    public function createOperator(): Operation
    {
        return new Minus();
    }
}

class ProductFactory implements IOperationFactory
{
    public function createOperator(): Operation
    {
        return new Product();
    }
}

```

```php=

$AddFactory = new AddFactory();
$Operator = $AddFactory->createOperator();
$Operator->operate(1, 2);

```

如此一來，我要新增新的運算類別的時候，我只需要對原本的設計做擴充，而不用修改

## 時機

- 當你希望能用不同的生成邏輯或流程去生成物件時
- 當類別希望讓子類別去指定欲生成的物件類型時

## 目的

定義一個建立物件的介面，讓子類別決定該實體化哪一個類別。工廠方法使一個類別的實例化延遲到其子類別。

## 類別圖

![](https://i.imgur.com/IWkEej1.png)
![](https://i.imgur.com/4wtEDiU.png)

### Plant UML

```=
@startuml

skinparam classAttributeIconSize 0

class Creator{
{method} +createProduct():Product
}

class ConcreteCreatorA{
{method} +createProduct():Product
}

class ConcreteCreatorB{
{method} +createProduct():Product
}

class Product
class ConcreteProductA
class ConcreteProductB

Product <|-- ConcreteProductA
Product <|-- ConcreteProductB
Creator <|-- ConcreteCreatorA
Creator <|-- ConcreteCreatorB
Product <-- Creator : <<Create>>

@enduml
```

```=
@startuml

skinparam classAttributeIconSize 0

class OperationFactory{
{method}+createOperator(): Operator
}
class AddFactory{
{method}+createOperator(): Operator
}
class MinusFactory{
{method}+createOperator(): Operator
}
class InnerProductFactory{
{method}+createOperator(): Operator
}

class Operator{
{method} +operate(): float
}
class Add{
{method} +operate(): float
}
class Minus{
{method} +operate(): float
}
class InnerProduct{
{method} +operate(): float
}

Operator <|-- Add
Operator <|-- Minus
Operator <|-- InnerProduct
OperationFactory <|-- AddFactory
OperationFactory <|-- MinusFactory
OperationFactory <|-- InnerProductFactory
Operator <-- OperationFactory : <<Create>>

@enduml
```

## 優點

- 可以避免創造者與產品間的緊密耦合(OperationFactory & Add/Minus)
- 符合開放封閉原則，可以在不修改原本設計的狀況下去引入新的產品生成
- 替生成的子類別預留掛鉤(hook)，使生成過程更具有彈性

## 缺點

- 此設計模式將帶來超大量的子類別
