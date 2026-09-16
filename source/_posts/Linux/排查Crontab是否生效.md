---
title: 排查 Crontab 是否生效：从执行日志到 spool 目录
date: 2026-09-16 10:00:00
author: chengjt
summary: 记录一次 Linux 日志清理定时任务的排查过程，用任务本和调度员的比喻讲清 crond、crontab、cron.d、周期脚本目录与 spool，并区分调度触发和执行成功。
tags:
    - Linux
    - crontab
    - 运维
    - 日志
categories:
    - Linux
---

一次日志清理任务的排查，让我重新梳理了几个容易混淆的问题：写进 crontab 就一定会执行吗？看到 `CMD` 日志就代表命令成功了吗？`crond`、`crontab`、`/etc/crontab` 和 `/etc/cron.d/` 分别是什么？`spool` 又是什么意思？

这篇文章记录实际观察到的现象，也整理了一套可以复用的检查顺序。示例环境使用 `crond`，启动日志显示版本为 Cronie 1.5.5；其他发行版的服务名和文件路径可能不同。

文中的业务名称、内部路径、日志文件名和进程编号均已替换为通用示例。命令中的示例路径需要按自己的环境调整。

<!-- more -->

## 事情的起点：想定时清空应用日志

服务的输出文件是：

```text
/opt/example-app/logs/app.out
```

为了快速验证定时任务是否生效，先将执行频率设置为每分钟一次：

```cron
* * * * * /usr/bin/truncate -s 0 /opt/example-app/logs/app.out
```

`truncate -s 0` 会把文件长度设为零，原有日志内容随之丢失。这里的每分钟配置只用于短时间测试，正式使用时需要调整频率。

接下来观察调度服务的日志：

```bash
journalctl -u crond -f
```

当时看到的执行记录只有另一项检查任务。以下摘录省略主机名，并使用通用示例标识：

```text
9月 16 09:34:01 CROND[10001]: (root) CMD (bash /opt/example-app/bin/service-check check >/dev/null 2>&1)
9月 16 09:35:01 CROND[10002]: (root) CMD (bash /opt/example-app/bin/service-check check >/dev/null 2>&1)
9月 16 09:36:01 CROND[10003]: (root) CMD (bash /opt/example-app/bin/service-check check >/dev/null 2>&1)
```

这说明 `crond` 正在触发任务，但在这段观察窗口里，没有看到目标清理命令的调度记录。

**单凭这些日志，还不能断言它只执行 `/etc/cron.d/`，或者完全不读取 root 的 crontab。** 日志没有标出配置来源，需要继续查看实际文件和服务配置。

完整排查记录还提供了几项证据：root 的 `crontab -l` 能看到清理任务，spool 文件权限为 `600`，`cat -A` 显示行尾有换行；服务状态则显示 `active (running)`，当时的启动命令为 `/usr/sbin/crond -n`。

手动执行清理命令后，文件从原先的 44M 变为检查时的 1.5M：

```text
-rw-r--r-- 1 root root 44M  9月 16 09:24 app.out
-rw-r--r-- 1 root root 1.5M 9月 16 09:26 app.out
```

这支持“手动清理产生了效果”，清理后继续有进程写入是合理解释。它不能直接证明 cron 环境下也一定成功，后者仍需单独验证。

另外，搜索任务配置时找到了检查命令：

```bash
grep -R "service-check" /etc/cron* /var/spool/cron 2>/dev/null
```

```text
/etc/cron.d/service-check:* * * * * root bash /opt/example-app/bin/service-check check >/dev/null 2>&1
```

因此，`service-check` 在 `/etc/cron.d/service-check` 中有对应配置是已确认的；这仍不能推出所有用户任务都被忽略了。

## 用最通俗的方式理解 crond 和 crontab

可以把这套机制想成“调度员看任务本，到点安排执行”：

| 名称 | 通俗理解 | 实际作用 |
| --- | --- | --- |
| `cron` | 定时办事这套机制 | 泛指定时调度程序或系统 |
| `crond` | 一直值班的调度员 | 后台运行，按配置触发命令 |
| crontab 任务表 | 写着时间和事情的任务本 | 保存任务规则 |
| `crontab` 命令 | 编辑任务本的工具 | 查看、编辑和安装用户任务表 |

也就是说，`crontab -e` 是在修改任务，`systemctl status crond` 是在查看调度员是否值班。任务表存在和服务运行，是两件需要分别检查的事。

再把不同配置位置看成不同的任务本：root 的用户任务本、系统公共任务表，以及按用途分开的公共任务文件。无论写在哪一种任务表里，最后都由调度服务安排执行。

这里的 `crontab` 既可以指任务表，也可以指管理命令；`/etc/crontab` 则是一个具体文件。执行 `crontab -e` 编辑的是指定用户的任务表，**不会打开 `/etc/crontab`**。管理命令的说明见 [Cronie 的 crontab 命令手册](https://github.com/cronie-crond/cronie/blob/master/man/crontab.1)。

## 先分清两种 crontab 格式

### 用户 crontab：不写执行用户

以 root 身份执行：

```bash
crontab -l
crontab -e
```

分别用于查看和编辑 root 的定时任务。用户 crontab 中，执行身份由所属用户决定，因此格式是：

```text
分钟 小时 日 月 星期 命令
```

例如每天凌晨两点清空日志：

```cron
0 2 * * * /usr/bin/truncate -s 0 /opt/example-app/logs/app.out
```

本例环境中，root 的用户 crontab 保存在 `/var/spool/cron/root`。日常修改优先使用 `crontab -e`，避免直接改 spool 文件时遗漏安装检查或文件属性处理。

如果管理员使用 `crontab -u mysql -e` 编辑任务，任务属于 mysql 用户，执行身份也是 mysql。因此更准确的说法是“按任务表所属用户执行”，而不是“谁敲下编辑命令，就由谁执行”。

### 系统任务：需要写执行用户

`/etc/crontab` 和 `/etc/cron.d/` 中的系统任务多一个用户字段：

```text
分钟 小时 日 月 星期 用户 命令
```

例如 `/etc/cron.d/app-clear-log`：

```cron
0 2 * * * root /usr/bin/truncate -s 0 /opt/example-app/logs/app.out
```

这里的 `root` 不能省略，也不能原样复制到用户 crontab 中。这两种格式的区别可参考 [Cronie 的 crontab 手册](https://github.com/cronie-crond/cronie/blob/cronie-1.5.5/man/crontab.5)。

对于需要独立文件管理的运维任务，`/etc/cron.d/` 比较方便；用户 crontab 同样可以用于运维，两者都受支持，选择取决于管理方式。

## /etc/crontab 和几个 cron 目录分别放什么

### /etc/crontab：系统公共任务表

`/etc/crontab` 是一个系统任务配置文件。`/etc/cron.d/` 则是目录，里面可以有多个配置文件，分别管理备份、日志清理、服务检查等任务。

两者使用相同的系统任务格式，都需要指定执行用户。`crond` 分别读取它们，并不需要先在 `/etc/crontab` 中登记 `/etc/cron.d/` 的每一个文件。

例如，下面是一份说明格式的 `/etc/crontab` 示例，并非本次服务器的文件回读：

```cron
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin

01 * * * * root run-parts /etc/cron.hourly
```

它表示每小时的第 1 分钟，以 root 身份调用 `run-parts`，执行该目录中符合条件的脚本。系统任务的读取位置可参考 [Cronie 的 cron 手册](https://github.com/cronie-crond/cronie/blob/master/man/cron.8)。

`/etc/crontab` 也可以直接放业务命令，它不是专门的“系统初始化任务表”。一些发行版中的这个文件很简洁，周期维护任务可能由其他配置或 anacron 安排，不能拿一份示例当成所有系统的默认内容。

### cron.hourly 等目录：放脚本，不写时间表

`/etc/cron.hourly/`、`/etc/cron.daily/`、`/etc/cron.weekly/` 和 `/etc/cron.monthly/` 通常用于存放按相应周期执行的脚本。

它们和 `/etc/cron.d/` 的主要区别是文件内容：

- `/etc/cron.d/` 放“什么时间、哪个用户、执行什么命令”的任务配置。
- 周期目录放实际执行的脚本，脚本中不写五个 cron 时间字段。

例如，一个周期脚本可以是：

```sh
#!/bin/sh
/usr/bin/date >> /var/log/cron-hourly-example.log
```

脚本通常需要可执行权限，文件名也需要符合系统所用 `run-parts` 的筛选规则。不同实现的规则可能不同，例如 Debian 默认规则通常不接受文件名中的点，因此不能认为 `example.sh` 放进去就一定会运行。规则示例见 [Debian 的 run-parts 手册](https://manpages.debian.org/bookworm/debianutils/run-parts.8.en.html)。

这些目录名本身不会触发执行。必须有调用它们的调度配置；每日、每周、每月任务也可能通过 anacron 安排，实际时间可能包含延迟。相关配置见 [Cronie 的 anacrontab 手册](https://github.com/cronie-crond/cronie/blob/master/man/anacrontab.5)。需要固定每天 02:00 执行时，直接写明确的 cron 时间规则更直观。

### 放在一起比较

| 位置或入口 | 文件内容 | 如何确定执行用户 | 常见管理方式 |
| --- | --- | --- | --- |
| `crontab -e` → 本例的 `/var/spool/cron/<user>` | 时间规则和命令 | 任务表所属用户 | 使用 `crontab` 命令 |
| `/etc/crontab` | 时间规则、用户和命令 | 每条配置中的用户字段 | 编辑系统公共任务表 |
| `/etc/cron.d/<任务名>` | 时间规则、用户和命令 | 每条配置中的用户字段 | 按用途管理独立配置文件 |
| `/etc/cron.hourly/` | 可执行脚本 | 调用它的调度配置决定 | 管理小时周期脚本 |
| `/etc/cron.daily/`、`weekly/`、`monthly/` | 可执行脚本 | 调用它的 cron 或 anacron 配置决定 | 管理对应周期的脚本 |

对于这次日志清理，如果希望接手服务器的人按文件名就能找到用途，可以选 `/etc/cron.d/app-clear-log`。它是管理上的选择，不代表用户 crontab 不适合正式任务。

## 一套可以复用的排查顺序

下面这些检查是根据本次经历整理的排查方法，不代表当时已经执行过每一步。

### 1. 确认服务状态和时间

```bash
systemctl status crond
date
timedatectl
```

先确认服务在运行，再确认服务器的时间和时区。`0 2 * * *` 是按任务适用的时区匹配，不能直接理解为自己电脑上的凌晨两点。

### 2. 确认任务属于哪个用户、保存在哪里

```bash
crontab -u root -l
cat /etc/crontab
ls -l /etc/cron.d/
```

如果目标任务放在独立文件中，再查看对应文件：

```bash
cat /etc/cron.d/app-clear-log
```

重点检查时间字段、命令路径和用户字段。如果迁移了配置，需要移除旧位置的同一任务，避免重复执行。

### 3. 检查文件属性和格式

对于系统任务文件，可以检查：

```bash
ls -l /etc/cron.d/app-clear-log
cat -A /etc/cron.d/app-clear-log
```

常见问题包括文件允许其他用户写入、误加可执行权限、Windows 换行，以及最后一行缺少换行符。`cat -A` 中正常行尾通常显示 `$`，Windows 换行可能显示 `^M$`。

由 root 管理的这个系统任务文件可设置为：

```bash
chown root:root /etc/cron.d/app-clear-log
chmod 644 /etc/cron.d/app-clear-log
```

这些命令针对 `/etc/cron.d/` 中的文件，不要照搬去修改用户 spool 文件。Cronie 对文件属性和末尾换行的要求见 [crontab 手册的 CAVEATS 部分](https://github.com/cronie-crond/cronie/blob/cronie-1.5.5/man/crontab.5)。

### 4. 查看启动参数和加载错误

```bash
systemctl cat crond
cat /etc/sysconfig/crond
journalctl -u crond --since '10 minutes ago'
```

`/etc/sysconfig/crond` 是否存在取决于发行版。使用 `systemctl cat` 可以同时看到服务定义和覆盖配置。

这里有一个值得纠正的细节：在 Cronie 中，`crond -n` 表示前台运行，`crond -c` 表示启用集群支持，**不是指定 spool 目录**。这些参数不能直接当成“用户 crontab 不生效”的证据。

集群模式确实可能影响用户任务的执行，但必须结合实际参数和主机选择配置判断。本次回读的服务状态显示 `/usr/sbin/crond -n`，启动日志还显示启用了 inotify，没有证据证明故障由集群模式导致。参数含义及自动检测配置变化的机制见 [Cronie 的 cron 手册](https://github.com/cronie-crond/cronie/blob/master/man/cron.8)。

正常情况下，Cronie 会检测配置变化，修改任务后通常不需要重启服务。重启可以帮助重新读取配置，但“重启后恢复”并不能单独证明之前是 inotify 异常。

### 5. 把命令输出留下来

排查时不要一开始就把输出丢到 `/dev/null`。可以临时让清理任务记录时间、错误信息和退出码，例如在 `/etc/cron.d/app-clear-log` 中使用：

```cron
* * * * * root { /usr/bin/date; /usr/bin/truncate -s 0 /opt/example-app/logs/app.out; rc=$?; echo "exit=$rc"; } >> /var/log/app-clear-log-test.log 2>&1
```

然后查看结果：

```bash
tail -n 20 /var/log/app-clear-log-test.log
stat /opt/example-app/logs/app.out
```

`exit=0` 表示该次 `truncate` 返回成功。如果应用持续写入，检查时文件可能已经重新变大，因此不能要求它一直保持零字节。验证后应删除这条测试配置，换成正式任务。

## 本次恢复后的日志说明了什么

排查过程中提出了将任务放到 `/etc/cron.d/app-clear-log` 的方案。随后提供的日志显示 `crond` 被重启，下一分钟出现了目标命令；对话没有完整展示最终配置文件，因此配置是否迁移仍需回读确认：

```text
9月 16 09:37:17 systemd[1]: Started Command Scheduler.
9月 16 09:37:17 crond[10000]: (CRON) STARTUP (1.5.5)
9月 16 09:38:01 CROND[10004]: (root) CMD (/usr/bin/truncate -s 0 /opt/example-app/logs/app.out)
```

到这里，可以确认目标任务已经被调度触发。但还要区分三个层次：

| 层次 | 验证方式 | 本次已有证据 |
| --- | --- | --- |
| 配置存在 | 查看对应 crontab 或 cron.d 文件 | 最初的 root 用户任务已回读；恢复后的最终文件未完整回读 |
| 调度触发 | 查看目标命令的 `CMD` 日志 | 已确认 |
| 清理成功 | 查看退出码、错误输出和文件变化 | 手动执行后文件明显变小；定时执行结果尚未完整验证 |

日志中的 `(root)` 表示执行身份，不能据此判断任务来自 `/var/spool/cron/root` 还是 `/etc/cron.d/`。

因此，本次可靠的结论是：**调整后已观察到目标任务触发，最初没有看到触发记录的根因仍未完全确认。** 不能进一步认定“root crontab 已重新加载”或“命令没有权限问题”。

正式配置若保留在 `/etc/cron.d/app-clear-log`，频率可以改为：

```cron
0 2 * * * root /usr/bin/truncate -s 0 /opt/example-app/logs/app.out
```

## 顺便理解 spool 是什么

排查时接触到 `/var/spool/cron/root`，也就遇到了 `spool` 这个词。它常与“假脱机”有关，可以理解为把待处理的数据先存放起来，由后台服务处理。

最直观的例子是打印：应用提交打印任务后，任务进入打印队列，打印服务再逐个交给打印机。

[文件系统层次标准 FHS](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch05s14.html) 将 `/var/spool` 定义为存放等待后续处理的数据。邮件发送队列、打印任务等都属于这种用途。

不过，`/var/spool/cron/root` 更准确地说是 **root 用户的定时任务配置文件**。它保存时间规则和命令，不是每执行一次就消费、删除一条记录的普通任务队列。Cronie 根据配置按时触发命令。

另外，`/var/spool` 是目录，查看它应该用：

```bash
ls -l /var/spool
```

查看具体文件才使用：

```bash
cat /var/spool/cron/root
```

## 日志管理可以进一步改成轮转

每天直接清空日志能控制文件增长，但不会保留历史记录。如果后续需要追查故障，可以考虑日志轮转。

对于无法方便通知进程重新打开输出文件的服务，可以评估使用 `copytruncate`。例如创建 `/etc/logrotate.d/example-app`：

```conf
/opt/example-app/logs/app.out {
    daily
    rotate 7
    copytruncate
    compress
    missingok
    notifempty
}
```

这份配置按天轮转非空日志，保留七份历史文件并压缩。`copytruncate` 先复制，再原地清空文件；两步之间存在窗口，可能丢失少量日志。采用轮转方案后，应移除原来的直接清空任务。

不要把 `daily` 和 `size 100M` 简单叠加并理解成“每天轮转，超过 100M 也轮转”。`size` 与时间条件有优先级关系；需要时间条件加提前按大小轮转时，可以了解 `maxsize 100M`。

轮转条件只有在 `logrotate` 被调用时才会检查，文件超过阈值并不会自动触发一次调用。`rotate 7` 表示保留七份，不一定正好覆盖七天。上述行为可参考 [logrotate 官方手册](https://github.com/logrotate/logrotate/blob/3.21.0/logrotate.8.in)。

配置后可以先检查，不实际修改日志：

```bash
logrotate -d /etc/logrotate.d/example-app
```

同时确认服务器已经通过定时任务或 `logrotate.timer` 定期调用 logrotate。如果应用本身支持日志框架的滚动文件功能，也可以直接在应用侧配置。

这次排查最值得保留的习惯，是把“配置写好了”“调度触发了”“命令成功了”分别验证。每一步都有对应的证据，定位问题时才不会因为看到一条日志就过早下结论。
