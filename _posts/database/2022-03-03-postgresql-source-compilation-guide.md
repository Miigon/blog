---
title: "[SZU] 数据库内核课程 PostgreSQL 12.5 源码安装避坑 guide"
date: 2022-03-04 00:08:11 +0800
categories: [Backend, Database, SZUCourse]
tags: [guide]
---

> 课程：2022 年下学期，秦建斌老师的《数据库内核原理与实现》课程。  
> 示例环境：Ubuntu 20.04 LTS  
> PostgreSQL 版本： 12.5  

## 1. 准备 linux 环境/虚拟机/或Windows下使用wsl

Linux/Mac 用户可直接编译，Windows 用户自行搜索 wsl 教程配置后，剩余流程同 Linux 用户。

Mac 用户需要用其他方式（homebrew 或 macports）安装 configure 过程中缺少的包。

## 2. 登陆机器，拷贝 PGDev.zip 到用户文件夹`~`中
使用 ssh/scp，或执行指令现场下载：
```
wget https://raw.fastgit.org/Miigon/pgdev-zip/master/PGDev.zip -O PGDev.zip
```

![](/assets/img/database/pg-comp-guide/0.png)
## 3. 安装 unzip 和 libreadline-dev：
```
sudo apt install unzip libreadline-dev
```
![](/assets/img/database/pg-comp-guide/3.png)
## 4. 解压 PGDev.zip
```
unzip PGDev.zip
```
![](/assets/img/database/pg-comp-guide/4.png)

解压完成后，出现 PGDev 文件夹：

![](/assets/img/database/pg-comp-guide/5.png)
## 5. 进入 PGDev 文件夹，创建 pghome 与 data 文件夹，用于存储 PostgreSQL 本身以及 PostgreSQL 所产生的数据
```
cd PGDev/
mkdir pghome
mkdir data
```
![](/assets/img/database/pg-comp-guide/6.png)
## 6. 使用 nano (或 vim/gedit 等其他编辑器) 编辑 env-debug 文件：
> 方向键移动光标

![](/assets/img/database/pg-comp-guide/7.png)

* 将第一行的 `PGHOME=/Users/jqin/PGDev/target-debug` 整行改为 `PGHOME=~/PGDev/pghome`
* 将第三行的 `export PGHOST=$PGDATA` 改为 `export PGHOST=127.0.0.1`

修改后：

![](/assets/img/database/pg-comp-guide/7.1.png)

按 Control + X，下方提示 `Save modified buffer?`，按 Y，出现 `File Name To Write: env-debug` 直接按回车，保存退出。
## 7. 执行 source
```
source env-debug
```

这一步将前面配置好的路径导入当前 shell 的环境变量中，没有任何输出则表示执行成功

![](/assets/img/database/pg-comp-guide/7.2.png)

## 8. 执行 configure（autoconf）
cd 进入 `postgresql-12.5`，执行：
```
./configure --prefix=$PGHOME
```
![](/assets/img/database/pg-comp-guide/7.3.png)

（确保执行时路径正确）

如果一开始有装 `libreadline-dev`，这里应该能执行成功：

![](/assets/img/database/pg-comp-guide/7.4.png)

## 9. 构建 & 安装
执行：
```
make -j4
```
> 这里假设机器 4 核心，如果不是可自行更改。写错了也不会影响构建结果

等待构建完成，看到这一句代表构建完成：

![](/assets/img/database/pg-comp-guide/7.5.png)

将编译好的 PostgreSQL 安装到 pghome 中：
```
make install
```
安装成功的提示：

![](/assets/img/database/pg-comp-guide/7.6.png)

## 10. 运行

1. 执行 `initdb` 初始化数据库：  
![](/assets/img/database/pg-comp-guide/7.7.png)

2. 执行 `pg_ctl -D ~/PGDev/pghome/../data -l ~/PGDev/logfile start` 启动 PostgreSQL 服务：  
（**注意这里的指令和上图提示的指令不同**）  
![](/assets/img/database/pg-comp-guide/8.png)

3. 执行 `createdb` 创建数据库，再执行 `psql` 进行连接：  
（这两个指令后面都可带参数来指定数据库名，不带则默认同用户名，建议不带参数。）  
![](/assets/img/database/pg-comp-guide/10.png)

此时应该就可以正常使用了：
![](/assets/img/database/pg-comp-guide/12.png)


## 后记

1. 这样安装后，PostgreSQL 本体会在 `~/PGDev/pghome` 中，数据会在 `~/PGDev/data` 中。
2. **后续如果对源代码作出修改，只需要在 `~/PGDev/postgresql-12.5` 中再次执行 make 与 make install 即可。**
3. 建议使用 git 对 `postgresql-12.5` 文件夹进行版本管理，方便后续修改回退。
4. **如果关闭了当前终端，打开新终端后需要先执行一次 `source ~/PGDev/env-debug`，否则会提示找不到 psql 等错误。**
