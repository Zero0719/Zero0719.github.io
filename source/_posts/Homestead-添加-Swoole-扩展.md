---
title: Homestead 添加 Swoole 扩展
date: 2024-03-31 19:15:48
tags:
  - Swoole
  - Homestead
categories: PHP
---

源码安装方式适用于 `PHP` 所有的扩展

* php~7.4
* swoole~4.8.12

```bash
# 切换php为7.4版本
php74

# 确定php是否已经安装了swoole
php -m | grep swoole

# 下载swoole源码包
wget https://wenda-1252906962.file.myqcloud.com/dist/swoole-src-4.8.12.tar.gz

tar -xzvf swoole-src-4.8.12.tar.gz

cd swoole-src-4.8.12/

phpize

# 下面这些配置建议加入，省的后续需要重新编译扩展
# sudo apt-get install libcurl4-openssl-dev (--enable-swoole-curl 需要该支持)
./configure --enable-openssl --enable-http2 --enable-swoole-curl

sudo make && sudo make install

# 安装成功以后我们可以看到对应扩展目录中有 swoole.so 文件
ls /usr/lib/php/20190902/ | grep swoole

sudo vim /etc/php/7.4/mods-available/swoole.ini
```

`swoole.ini` 加入以下内容
```ini
extension=swoole.so
swoole.use_shortname=off
```

执行命令
```bash
cd /etc/php/7.4/cli/conf.d
sudo ln -s /etc/php/7.4/mods-available/swoole.ini 20-swoole.ini

# 出现 swoole 扩展，扩展到此结束，源码编译安装扩展的方式适用于任何 Linux 系统，也适用于其他的扩展，比如 redis，pdo 等等
php -m | grep swoole
```
