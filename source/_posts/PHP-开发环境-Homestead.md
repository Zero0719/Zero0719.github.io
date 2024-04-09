---
title: PHP 开发环境-Homestead
date: 2024-03-31 18:59:35
tags: Homestead
categories: PHP
---

`Homestead` 是一个可以在本地快速部署一个 `Laravel` 项目的虚拟机盒子，它基于 `ubuntu` 的基础上，内置了 `php-fpm`, `nginx`, `mysql` 等等的服务，事实上可以有很多个本地`php`的开发部署方案，比如 `phpstudy`, `docker` 等，但是用虚拟机部署，更接近生产环境，可以使我们平时开发时对虚拟机的操作，如果有必要可以以同样的方式同步到生产环境中，大部分 `Linux` 的操作大同小异。

虽然 `Homestead` 对外的官方搭档是 `Laravel` 项目，但是其实项目中内置了很多其他 `php` 框架对应的 `nginx` 配置，只需要修改一点配置文件，就可以使用 `proxy` 模式，或者生成适配 `thinkphp`, `yii` 等框架的配置，虚拟机中还部署了多个版本的 `php`，简单的指令就可以让你轻松切换 `php` 版本，所以该虚拟机盒子作为开发环境还是挺高效方便。

<!-- more -->

## 开始

### 软件

* [Virtualbox](https://www.virtualbox.org/)
* [Vagrant](https://www.vagrantup.com/)
* [Git](https://git-scm.com/)

### 拉取 Homestead 脚本
[laravel/homestead](https://github.com/laravel/homestead)

以下指令使用 `git bash` 操作
```bash
git clone https://github.com/laravel/homestead.git ~/Homestead

cd ~/Homestead

git checkout release

bash init.sh

# 如果用户还没有生成密钥对，则需要生成 rsa 密钥对
ssh-keygen -t rsa

# 启动 Homestead 虚拟机, 如果是首次执行，则会对应安装 laravel/homestead box，注意这里如果提示没有合适安装的版本，可在 Homestead.yaml 中指定 version: xxxxx
vagrant up

# ssh 连接至虚拟机
vagrant ssh
```

## Laravel/Homestaed 文件讲解

主要的核心文件是 `Homestead.yaml`，下面描述各个部分的配置块作用

`Homestead.yaml`

```yaml
---
# 定义虚拟机的IP
ip: "192.168.56.56"

# 虚拟机的内存
memory: 2048

# 虚拟机 cpu 核数
cpus: 2

# 虚拟机的的载体是 virtualbox
provider: virtualbox

# 本机的 id_rsa.pub 文件对应可 ssh 虚拟机中的权限
authorize: ~/.ssh/id_rsa.pub

keys:
    - ~/.ssh/id_rsa

# 目录映射关系
folders:
    - map: ~/code
      to: /home/vagrant/code

# nginx 站点配置
sites:
    - map: homestead.test
      to: /home/vagrant/code/public

# mysql 的数据库
databases:
    - homestead

# 可使用的功能
features:
    - mariadb: false
    - postgresql: false
    - ohmyzsh: false
    - webdriver: false

# 虚拟机中的服务
services:
    - enabled:
          - "mysql"
#    - disabled:
#        - "postgresql@11-main"

# 端口映射
#ports:
#    - send: 33060 # MySQL/MariaDB
#      to: 3306
#    - send: 4040
#      to: 4040
#    - send: 54320 # PostgreSQL
#      to: 5432
#    - send: 8025 # Mailpit
#      to: 8025
#    - send: 9600
#      to: 9600
#    - send: 27017
#      to: 27017
```

## 部署项目

比如我们需要新增一个 `Laravel` 开发的站点 `laravel.test`，我们需要修改 `Homestead.yaml` 中的内容
```yaml
sites:
    - map: homestead.test
      to: /home/vagrant/code/public
    - map: laravel.test
      to: /home/vagrant/code/laravel.test

databases:
    - homestead
    - laraveltest
```

修改完以后执行以下命令
```bash
cd ~/Homestead
vagrant provision
```

本地修改 `host` 即可
```
192.168.56.56 laravel.test
```

如果我们的项目是其他框架开发的，我们可以在 `Homestead/scripts/site-types` 中找到可用的类型，比如 `thinkphp`
```yaml
sites:
    - map: thinkphp.test
      to: /home/vagrant/code/thinkphptest
      type: thinkphp
```

更多详细用法可阅读 [官方文档](https://laravel.com/docs/11.x/homestead)

## 其他

`vagrant scp` 插件，将本地文件传输到虚拟机
```
vagrant plugin install vagrant-scp

# 首先需要确定传输的机器
vagrant global-status

# 会展示当前已有的所有虚拟机情况，我们需要用到 id 或者 name，这里建议用 id，name 可能会重复，传输成功以后目标文件就会在 vagrant 家目录中
vagrant scp  filename d4cfe67:/home/vagrant
```