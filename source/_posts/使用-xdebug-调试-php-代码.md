---
title: 使用 xdebug 调试 php 代码
date: 2024-06-24 11:37:02
tags:
  - php
  - xdebug
  - homestead
---

## 环境

* php5.6
* xdebug 扩展
* phpstorm 编辑器
* homestead 虚拟机(注意要用超管用户启动)

本文内容基于以上环境编写，其他环境下配置也基本大同小异，注意这篇文章的调试只在传统的 `php` 方式下，并不包括 `swoole` 这种环境。

## 安装 xdebug 扩展

安装方式可以参考 `swoole` 扩展的安装，只需要确保扩展安装好并启用即可

{% post_link Homestead-添加-Swoole-扩展 %}

`cli`
```
php -m | grep xdebug
```

`fpm`
```
phpinfo();
```

<!-- more -->

## xdebug 配置

这里需要注意，如果 `php` 的 `fpm` 和 `cli` 方式下引入的扩展不一样，需要按需配置

```
zend_extension=xdebug.so
xdebug.mode=debug
xdebug.remote_enable = 1
xdebug.remote_connect_back = 1
; 这里填写编辑器监听的端口，phpstorm默认是9000
xdebug.remote_port = 9000
; 这里填写主机地址，也就是编辑器所在的地址
xdebug.remote_host = 192.168.56.1
xdebug.max_nesting_level = 512
; 这里看情况填写，vscode或者其他编辑器可能不一样，也是看情况调整
xdebug.idekey = PHPSTORM
```

## phpstorm 配置

1.
![](/images/使用-xdebug-调试-php-代码/1.png)

2.
这里需要注意 `host` 是我们要访问的虚拟域名，如果是在虚拟机或者一些非本机环境下，我们需要做一些目录映射
![](/images/使用-xdebug-调试-php-代码/2.png)

3.
配置 `php cli` 执行目录，因为我们这里用的 `homestead` ，所以需要调用到虚拟机中的可执行命令
![](/images/使用-xdebug-调试-php-代码/3.png)
![](/images/使用-xdebug-调试-php-代码/4.png)

4.
配置 `debug` 配置,服务选择第二步中所添加的
![](/images/使用-xdebug-调试-php-代码/5.png)

## 测试调试

### fpm 方式
开启监听并调试(浏览器需要安装 `xdebug` 插件)
![](/images/使用-xdebug-调试-php-代码/6.png)
![](/images/使用-xdebug-调试-php-代码/7.png)

### cli 方式
![](/images/使用-xdebug-调试-php-代码/8.png)


## 总结

* php需要安装xdebug扩展，并且设置编辑器监听的端口
* 编辑器设置需要监听的端口，php可执行文件，调试的服务，目录映射等
* 浏览器安装调试扩展


