---
title: Design Patterns - Structural - Composite Pattern 組合模式
date: 2020-09-11 15:24:55
tags:
- Design Patterns
- Structural Patterns
categories: 
- 讀書會報告
cover: https://picsum.photos/id/1078/800/600
---

# 設計模式 - Composite Pattern 組合模式

## Intro

假設我們今天要建構一個可以代表公司與部門之間結構的關係，公司可以包含分公司、可以包含部門，而部門可以包含子部門或是員工。

## 實作

我們先建立一個Component類別，讓它具有可以擴充及移除的功能，且期望所有他的子類別都能夠有固定的行為(operation)

```php=
abstract class Component
{
    protected $name;

    public function __construct(string $name)
    {
        $this->name = $name;
    }
    
    abstract public function add(Component $component);

    abstract public function remove(Component $component);

    abstract public function operation();
}
```

再來我們新增兩個Component的子類別
Composite : 可以繼續擴充
Leaf : 不能擴充

```php=
class Composite extends Component
{
    protected $children;

    public function __construct(string $name)
    {
        parent::__construct($name);
    }

    public function add(Component $component)
    {
        $this->children[] = $component;
    }

    public function remove(Component $component)
    {
        //remove component children
    }

    public function operation()
    {
        // do something for Composite
    }
}

class Leaf extends Component
{
    public function __construct(string $name)
    {
        parent::__construct($name);
    }

    public function add(Component $component)
    {
        throw new Exception('Leaf can not add');
    }

    public function remove(Component $component)
    {
        throw new Exception('Leaf can not remove');
    }

    public function operation()
    {
        // do something for Leaf
    }
}
```
使用方式大概如下

```php=
$MainCompany = new Composite('總公司');

$MainCompanyDepartmentA = new Composite('總公司的部門A');
$MainCompanyDepartmentAEmployeeA = new Leaf('總公司的部門A-員工A');
$MainCompanyDepartmentA->add($MainCompanyDepartmentAEmployeeA);
$MainCompany->add($MainCompanyDepartmentA);

$MainCompanyDepartmentB = new Composite('總公司的部門B');
$MainCompanyDepartmentBEmployeeA = new Leaf('總公司的部門B-員工A');
$MainCompanyDepartmentB->add($MainCompanyDepartmentBEmployeeA);
$MainCompany->add($MainCompanyDepartmentB);

$SubCompanyA = new Composite('分公司A');
$SubCompanyADepartmentA = new Composite('分公司A的部門A');
$SubCompanyADepartmentAEmployeeA = new Leaf('分公司A的部門A-員工A');
$SubCompanyADepartmentA->add($SubCompanyADepartmentAEmployeeA);
$SubCompanyA->add($SubCompanyADepartmentA);
$MainCompany->add($SubCompanyA);

$SubCompanyB = new Composite('分公司B');
$SubCompanyBDepartmentA = new Composite('分公司B的部門A');
$SubCompanyBDepartmentAEmployeeA = new Leaf('分公司B的部門A-員工A');
$SubCompanyBDepartmentAEmployeeB = new Leaf('分公司B的部門A-員工B');
$SubCompanyBDepartmentA->add($SubCompanyBDepartmentAEmployeeA);
$SubCompanyBDepartmentA->add($SubCompanyBDepartmentAEmployeeB);
$SubCompanyB->add($SubCompanyBDepartmentA);
$MainCompany->add($SubCompanyB);

$SubCompanyBDepartmentB = new Composite('分公司B的部門B');
$SubCompanyBDepartmentBEmployeeA = new Leaf('分公司B的部門B-員工A');
$SubCompanyBDepartmentB->add($SubCompanyBDepartmentBEmployeeA);
$MainCompany->add($SubCompanyBDepartmentB);
```

透過這樣的過程可以建構出像下圖的結構

![](https://i.imgur.com/Xy6KaDR.png)

### Plant UML

```
@startuml

node MainCompany
node Main_DepartmentA
actor Employee1
node Main_DepartmentB
actor Employee2

node SubCompanyA
node A_DepartmentA
actor Employee3

node SubCompanyB
node B_DepartmentA
actor Employee4
actor Employee5
node B_DepartmentB
actor Employee6

MainCompany --> SubCompanyA
MainCompany --> SubCompanyB
MainCompany --> Main_DepartmentA
MainCompany --> Main_DepartmentB
Main_DepartmentA --> Employee1
Main_DepartmentB --> Employee2
SubCompanyA --> A_DepartmentA
A_DepartmentA --> Employee3
SubCompanyB --> B_DepartmentA
SubCompanyB --> B_DepartmentB
B_DepartmentA --> Employee4
B_DepartmentA --> Employee5
B_DepartmentB --> Employee6

@enduml

```

## 時機

- 希望對結構整體或部分甚至是單一物件都可以使用同樣地行為操作時
- 當你要時作的對象擁有的結構類似於"樹狀"時

## 目的

將物件組合成「樹狀」結構以表示「部分—全體」的層次結構。讓外界以一致性的方式對待個別物件和整體物件

## 類別圖

![](https://i.imgur.com/OhnzkDl.png)

### Plant UML

```
@startuml

skinparam classAttributeIconSize 0

class Component {
{method} + add(Component $component)
{method} + remove(Component $component)
{method} + operation()
}

class Composite {
{method} + add(Component $component)
{method} + remove(Component $component)
{method} + operation()
}

class Leaf {
{method} + add(Component $component)
{method} + remove(Component $component)
{method} + operation()
}

Leaf -up-|> Component
Composite -up-|> Component
Composite o-up-> Component : children

@enduml
```

## 優點

- 容易加入新地Component，擴充容易
- 對於使用者來說，不必在乎他使用的對象是Leaf或是Component，他們擁有相同地對外行為

## 缺點

- Component、Leaf管理上缺乏良好的機制
