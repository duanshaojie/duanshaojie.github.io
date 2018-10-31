---
title: Kettle的安装和简单使用(Linux)
tags: kettle
categories: 开发工具
---

​	因为我本地是window的，但是最后需要布置到linux上，所以这里主要记录下Linux上的安装部署，与定时调度。

​	最近公司需要做一些数据抽取的需求，需要将mongodb内部分数据进行定时抽取，输入到MySQL中。我们的涛哥根据我们的实际情况选择了kettle，这个记录只是简单的使用说明，具体业务需求与配置就不多说了（我也是第一次接触，也没啥可说的，有不对的地方还望各位前辈指出），话不多说，开始吧！

<!--more-->

- 介绍

  ​	Kettle是一款国外开源的ETL工具，纯java编写，可以在Window、Linux、Unix上运行。Kettle的中文名称叫水壶，名字真的是够通俗易懂的，该项目的主程序员MATT 希望把各种数据放到一个壶里，然后以一种指定的格式流出。window系统上可以直接使用图形化界面来编写任务（我就是在本地配好任务上传执行的-。-）

  ​	Kettle家族目前包括4个产品：Spoon、Pan、CHEF、Kitchen。

  ​	**SPOON**** **允许你通过图形界面来设计ETL转换过程（Transformation）。

  ​	**PAN**** **允许你批量运行由Spoon设计的ETL转换 (例如使用一个时间调度器)。Pan是一个后台执行的程序，没有图形界面。

  ​	**CHEF **允许你创建任务（Job）。 任务通过允许每个转换，任务，脚本等等，更有利于自动化更新数据仓库的复杂工作。任务通过允许每个转换，任务，脚本等等。任务将会被检查，看看是否正确地运行了。

  ​	**KITCHEN**** **允许你批量使用由Chef设计的任务 (例如使用一个时间调度器)。KITCHEN也是一个后台运行的程序。

- 环境说明

  ​	kettle的部署使用之前，我介绍下现在系统中安装的东西吧！	

  ​	Mongodb

  ​	MySQL

  ​	Jdk

  这里具体安装和使用就不介绍了，我们讲的是kettle的使用嘛

- Kettle部署配置

  Linux中Kettle文件结构

  ​	将下载的Kettle解压到（data-integration）创建的Kettle文件夹下。这里需要进入data-integration目录下，并为目录下的.sh文件赋予权限

  1.cd /usr/local/kettle/data-integration  ###进入目录

  2.chmod +x *.sh ###赋予权限，结束后再键入./kitchen.sh 出现宝珠信息则表示成功

  ​	接下来在Kettle文件夹下接着创建几个空文件夹ktllog，ktlsh和project文件夹。这样，我们的基本配置算是结束了；

- 创建任务及相关文件

  1.创建执行文件（这里我是在本地的window下Kettle中创建好的:) ）

  ​	这里后面更新本地kettle使用过程吧，最后生成.ktr文件,并复制到前面创建的project文件夹中。

  2.创建job

  ​	在ktllsh文件夹中新建文件 test.sh （在终端键入touch test.sh）并赋予权限 chmod  +x  test.sh

  编辑test.sh文件

  ```
  #!/bin/sh
  source /etc/profile  
   
  ROOT_TOPDIR=/usr/local/kettle
  Export ROOT_TOPDIR
  $ROOT_TOPDIR/data-integration/pan.sh -file=/usr/local/kettle/project/test.ktr
  ```

  保存后在ktlsh目录下执行./test.sh，如果出现以下则表示成功了

  ```
  2017/08/25 14:34:57 - 字段选择.0 - Finished processing (I=0, O=0, R=1, W=1, U=0, E=0)
  2017/08/25 14:34:57 - MongoDB Input.0 - Finished processing (I=0, O=0, R=0, W=1, U=0, E=0)
  2017/08/25 14:34:57 - 表输出.0 - Finished processing (I=0, O=1, R=1, W=1, U=0, E=0)
  2017/08/25 14:34:57 - Pan - Finished!
  2017/08/25 14:34:57 - Pan - Start=2017/08/25 14:34:54.427, Stop=2017/08/25 14:34:57.422
  2017/08/25 14:34:57 - Pan - Processing ended after 2 seconds.
  2017/08/25 14:34:57 - demo mongo -  
  2017/08/25 14:34:57 - demo mongo - Step MongoDB Input.0 ended successfully, processed 1 lines. ( 0 lines/s)
  2017/08/25 14:34:57 - demo mongo - Step 字段选择.0 ended successfully, processed 1 lines. ( 0 lines/s)
  2017/08/25 14:34:57 - demo mongo - Step 表输出.0 ended successfully, processed 1 lines. ( 0 lines/s)
  ```

- 定时任务配置

  如果上面的都成功了，接下来的就来做任务调度

  如下

  ```
  crontab –e ###增加一行
  30 12 * * *  /usr/loca/kettle/ktlsh/test.sh >> /usr/local/kettle/ktllog/logtest    
  ###如果ktllog下面不存在logtest文件会自动创建
  ###配置完后重启crontab服务：
  Service crond restart
  ###查看crontab服务
  Service crond status
  查看crontab中的内容
  Crontab –1
  ```

  而调度时间的设置可自行Google或者百度