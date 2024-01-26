---
title: 使用定时任务清理docker日志
date: 2023-06-21 20:26:00 +0800
categories: [Docker]
tags: [docker]
mermaid: true
---

上午正在写代码的时候，突然运营人员说系统无法上传文件了。很纳闷，文件系统当时是做了一个微服务，运行了两年了从来没有出过什么错误。就自己用测试账号试了以下，发现确实无法上传。把日志拉下来看了一下找到了问题

> System.IO.IOException: There is not enough space on the disk.

好奇怪，明明前不久看到磁盘还有将近100个G呢，为什么突然就满了。

使用 **df** 命令查看一下

|Filesystem|Size|Used | Avail |Use%  |Mounted on|
| ---- | ---- | ---- | ---- | ---- | ---- |
|devtmpfs|3.8G|0|3.8G| 0%| /dev|
|tmpfs|3.8G|0|3.8G| 0%| /dev/shm|
|tmpfs|3.8G|406M|3.4G|11%| /run|
|tmpfs|3.8G|0|3.8G| 0%| /sys/fs/cgroup|
|/dev/sda3|64G| 64G| 0|100%| /|
|/dev/sda5|32G|901M| 31G| 3%| /home|
|/dev/sda1|1014M|270M|745M|27%| /boot|
|overlay|64G| 64G| 0|100%| /var/lib/docker/overlay2/d9e48 |
|overlay|64G| 64G| 0|100%| /var/lib/docker/overlay2/c2eb0 |
|overlay|64G| 64G| 0|100%| /var/lib/docker/overlay2/25317 |
|overlay|64G| 64G| 0|100%| /var/lib/docker/overlay2/e80be |
|overlay|64G| 64G| 0|100%| /var/lib/docker/overlay2/98932 |
|overlay|64G| 64G| 0|100%| /var/lib/docker/overlay2/44c3e |
|overlay|64G| 64G| 0|100%| /var/lib/docker/overlay2/25571 |
|overlay|64G| 64G| 0|100%| /var/lib/docker/overlay2/363e7 |
|overlay|64G| 64G| 0|100%| /var/lib/docker/overlay2/8ec8d |
|overlay|64G| 64G| 0|100%| /var/lib/docker/overlay2/c0b3e |
|overlay|64G| 64G| 0|100%| /var/lib/docker/overlay2/90c8d |
|tmpfs|368M|0|368M| 0%| /run/user/0 |
|overlay|64G| 64G| 0|100%| /var/lib/docker/overlay2/ee9a0 |

哦吼，发现docker容器全是满的，猜了一下应该是docker日志满了。

cd 到每个容器下检查下日志，路径在 ***/var/lib/docker/containers/容器id***  

发现日志都已经达到了10个G以上，这玩意太能造了😅

![20240126163740.png (974×135) (raw.githubusercontent.com)](https://raw.githubusercontent.com/liugt34/imagegallery/main/20240126163740.png)



找到问题就好办了，决定使用定时任务对这些日志进行清理。

先做个定时清理的批处理命令

```sh
touch auto_delete_docker_logs.sh

vim auto_delete_docker_logs.sh;

#编辑文件
#!/bin/sh
echo "======== start clean docker containers logs ========"
logs=$(find /var/lib/docker/containers/ -name *-json.log)
for log in $logs
    do
        echo "clean log : $log"
        cp $log /home/app/logs/$log
        cat /dev/null > $log
    done
echo "======== end clean docker containers logs ========"
```

为文件添加可执行权限

```sh
chmod +R auto_delete_docker_logs.sh

#上面的命令centos会报错，使用以下命令
chmod +x auto_delete_docker_logs.sh 
```

使用 **crontab** 任务管理器实现定时删除

```sh
crontab -e

# m  h  dom  mon  dow  command
# 50 0 * * * /var/tasks/auto_delete_app_logs.sh
# 每月1号凌晨 02:10分执行命令
10 2 1 * *   /var/tasks/auto_delete_docker_logs.sh
```

使用命令 ***crontab -l*** 查看是否成功，每天凌晨 00:50 执行该命令

| m    | h    | dom  | mon  | dow  | command |
| ---- | ---- | ---- | ---- | ---- | ------- |
| 分钟 | 小时 | 日   | 月   | 周   | 命令    |
| 50 | 0 | * | * | * | /var/tasks/iotlogs.sh |

