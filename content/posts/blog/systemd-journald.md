---
title: "systemd-journal 占用内存的问题"
date: 2019-06-04T17:00:08+08:00
toc: false
images:
tags: 
  - Debian
  - Linux
---

# systemd-journal 占用内存的问题


最近发现部分 Debian 机器的 systemd-journal 占用了非常多内存。这和 Debian 对其的
错误配置有关系（查了一下其他发行版，有和 Debian 一样的配置的也有和 Debian 不一样
的配置的，说明这个配置有争议）。


## systemd-journal 简介

systemd-journal 是 systemd 引入的系统日志记录工具。其优势是：

* 使用二进制保存日志，有压缩，体积小
* 可以记录启动早期，磁盘还没挂载之前，rsyslog 还没启动时候的系统日志
* 有索引，可以快速搜索
* 索引包含了多种类型，可以方便使用多种维度，以及他们的组合，进行搜索，包含但不限
  于：
	* 时间
	* PID
	* 程序可执行文件路径
	* service 名称
	* 用户
	* 内核
	* 错误级别
* 显示的时候，可以针对不同等级做高亮，可以转换日志时间戳
* 可以针对日志设置用户访问权限控制
* 会对日志做校验，用户无法修改任何日志，日志也不能伪造用户、processid 等敏感信息
* 可以设置 rotate 和最大体积等各种限制，也可以比较方便地手工清理指定时间之前的日
  志
* 支持 syslog 的所有日志级别
* 支持复制日志并转发到 rsyslog

由于可以方便地过滤某个时间段的所有程序的日志，所以 journal 特别适合 debug 一些多
种环境下，会有多个日志源的复杂问题，可以按时间顺序将所有日志源共同打印出来，从而
清晰地观察到各种应用程序之间的交互顺序。

其缺点是：

* 不支持 rsyslog 的复制日志和转发过滤等功能

由于游戏需要 rsyslog 的转发过滤，因此我们一般都会打开 rsyslog，因此在 Debian 中，
日志会首先到达 systemd-journal，并且被保存为 journal 文件，同时再转一个副本给
rsyslog，由 rsyslog 控制写到 `/var/log/` 目录下，或者游戏项目自行设置的其他路径
下。

## systemd-journal 的配置

在 `/etc/systemd/journald.conf` 下面。支持的配置项还是比较多的。具体可以参考
`man 5 journald.conf`

## systemd-journal 的使用

使用 `journalctl` 命令。具体参数可以 `man 1 journalctl` 查看。


## systemd-journal 的坑


默认的配置文件，配置了 `Storage=auto`。含义为：


* 如果设置为 volatile，journal 将会保存在内存中，使用位于内存盘的
  `/run/log/journal` 目录（会自动创建）
* 如果设置为 persistent，journal 将会保存在磁盘中，使用 `/var/log/journal` 目录
  （会自动创建），如果自动创建失败，以及针对启动早期磁盘尚未挂载成功的部分日志，
  仍然记录在内存盘。
* 如果设置为 auto，那么，如果 `/var/log/journal` 目录存在，则使用该目录记录到磁
  盘，如果目录不存在（不会自动创建），则使用内存盘。
* 如果设置为 none，完全不记录任何日志（但是仍然可以转发给 rsyslog）
* 默认是 auto


而 Debian 默认并不会创建 `/var/log/journal` 目录（查了一下其他发行版，有创建的有
不创建的，看来不同发行版是有分歧的）。因此会导致默认配置情况下，journal 默认会将
日志全部保存在内存盘中。

在我们长期不关机的情况下，`/run/log/journal` 目录可能会变得非常大，从而导致占据
较多内存。

systemd 默认的配置，对总的存储空间做了上限。上限如下：

* 如果使用的是磁盘，那么上限默认为磁盘空间的 10% 和 4G 中较小的那个（由
  `SystemMaxUse` 控制）
* 如果使用的是内存，那么上限默认为内存空间的 15% 和 4G 中较小的那个（由
  `RuntimeMaxUse` 控制）

因此，极端情况下，journal 可能会消耗 4G 的内存。

## 清理 journal 的内存

* 清除到只剩下最新的 100M 空间：`journalctl --vacuum-size=100M`
* 清除到只剩下最近两小时：`journalctl --vacuum-time=2h`
* 将内存盘中的数据刷到硬盘：`journalctl --flush`
* 或者采用很黄很暴力的清除方法（不推荐）：`rm -rf /run/log/journal && systemctl
  restart systemd-journal`

## 建议的解决办法：

以下方法任选一种即可

* 方法一：创建 `/var/log/journal` 目录，然后使用 `journalctl --flush` 将内存盘中
  的数据刷到硬盘
* 方法二：修改 `/etc/systemd/journald.conf`，配置 `Storage=persistent`，然后重启
  `systemd-journal` 并使用 `journalctl --flush` 将内存盘中的数据刷到硬盘
* 方法三：修改 `/etc/systemd/journald.conf`，配置 `Storage=none`，然后重启
  `systemd-journal`
