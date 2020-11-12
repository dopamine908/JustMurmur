---
title: 在 win10 上使用 php-cs-fixer 自動排版 php 程式碼為 psr-12 格式
date: 2020-03-04 13:24:25
tags:
- Coding Style
- PSR
- PHP
categories: 
- 日常開發筆記
cover: https://picsum.photos/id/618/800/600
---

# 在 win10 上使用 php-cs-fixer 自動排版 php 程式碼為 psr-12 格式

首先，我們先透過composer安裝[php-cs-fixer](https://github.com/FriendsOfPHP/PHP-CS-Fixer#usage)

```
composer global require friendsofphp/php-cs-fixer
```

然後我們便可以透過下面指令去重新排版資料夾內的檔案成預設的排版規則
```
php php-cs-fixer.phar fix /path/to/project 
```


在 windows 上，如果你想要透過絕對路徑去執行上方指令的話可以參考下方路經

其中UserName為自己的帳號名稱

``` 
C:\Users\UserName\AppData\Roaming\Composer\vendor\friendsofphp\php-cs-fixer\php-cs-fixer
```

所以我們就可以將指令改為透過絕對路徑去執行

```
php C:\Users\UserName\AppData\Roaming\Composer\vendor\friendsofphp\php-cs-fixer\php-cs-fixer fix /path/to/project
```
 
 接下來，我們可以將排版規則客製化
 例如我們可以選擇:psr-1,psr-2....etc
 我們只需要稍微調整一下指令讓他知道我們要使用的規則
```
php C:\Users\UserName\AppData\Roaming\Composer\vendor\friendsofphp\php-cs-fixer\php-cs-fixer fix /path/to/project --rules=@PSR2
```

但比較遺憾的是，在筆者寫文章的時間點 (2020.03.04) php-cs-fixer 還沒有支援 psr-12 的規範
所以我們要使用的是 [KorvinSzanto](https://github.com/KorvinSzanto/PHP-CS-Fixer) 寫的擴充
 
  擴充的檔案在這邊 > ['PHP-CS-Fixer/src/RuleSet.php'](https://github.com/KorvinSzanto/PHP-CS-Fixer/blob/feature/psr12/src/RuleSet.php#L60)
  我們只要將 ```RuleSet.php``` 替換成 KorvinSzanto 寫的檔案
  最後再執行一次指令
  我們就可以將程式碼風格編排成 psr-12 的格式了

