---
title: Amazon Linux Install MySQL
date: 2025-02-15 13:52:38 +0800
categories: [Linux]
tags: [mysql]
mermaid: true
---

访问MySQL官方下载网站  [MySQL :: MySQL Community Downloads](https://dev.mysql.com/downloads/) 
Amazon Linux 使用的是YUM，所以选择MySQL YUM Repository 点击进入

# 安装MYSQL
## 使用命令安装

``` bash
wget https://repo.mysql.com/mysql84-community-release-el9-1.noarch.rpm
# 添加MYSQL YUM仓库
sudo dnf install mysql84-community-release-el9-1.noarch.rpm
sudo dnf update
sudo dnf install mysql-community-server

sudo systemctl start mysqld
sudo systemctl enable mysqld

# 检查服务状态：
sudo systemctl status mysqld
```

## 设置MYSQL安全项

初始时这并非必需，但如果您计划商业上使用数据库服务器，公共用户将通过它访问数据，则建议通过设置根密码、删除匿名用户、禁用远程根登录等方式来保护您的MySQL安装。



> 首先使用以下命令找到MySQL为根用户设置的默认密码

```bash
sudo grep 'temporary password' /var/log/mysqld.log
# 控制台会输出默认密码，将其复制保存

sudo mysql_secure_installation -p
# 输入刚刚保存的密码,并设置初始密码
```

设置一个安全密码，然后系统将要求您检查密码的强度以进行验证，您可以根据需要按Y或N。之后，按照向导进一步保护您在Amazon Linux 2023上的MySQL数据库服务器实例。

## 访问数据库

```bash
mysql -u root -p
# 输入密码进入
```

> root仅能在本地访问，无法远程访问

## 创建管理用户

```bash
CREATE USER 'aws'@'%' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON *.* TO 'aws'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```



# 安装管理工具

## 安装APACHE

安装完毕后使用 http://ec2-instance-public-ip 访问验证是否可以访问，记得**安全组**要添加80端口

```bash
sudo yum update
sudo yum install httpd
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl status httpd
```

## 安装PHP

安装php相关库

```bash
sudo dnf install -y php-fpm php-mysqli php-json php php-devel
# 测试是否安装成功
sudo php -v
```

设置文件权限

```bash 
sudo usermod -a -G apache ec2-user
```

退出重新登录验证

```
[ec2-user ~]$ exit
[ec2-user ~]$ groups
ec2-user adm wheel apache systemd-journal
```

修改文件夹 **/var/www** 组权限拥有者

```
sudo chown -R ec2-user:apache /var/www
```

设置文件夹 **/var/www** 及其子文件夹的写权限

```bash
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
```

设置文件夹 **/var/www** 及其子文件的写权限

```bash
find /var/www -type f -exec sudo chmod 0664 {} \;
```

## 安装PHPMYADMIN

安装依赖

```bash
sudo dnf install php-mbstring php-xml -y
```

重启Apach服务

```bash
sudo systemctl restart httpd
```

重启php-fpm

```bash
sudo systemctl restart php-fpm
```

切换至/var/www/html目录

```bash
cd /var/www/html
wget https://files.phpmyadmin.net/phpMyAdmin/5.2.2/phpMyAdmin-5.2.2-all-languages.tar.gz
mkdir phpMyAdmin && tar -xvzf phpMyAdmin-5.2.2-all-languages.tar.gz -C phpMyAdmin --strip-components 1
rm phpMyAdmin-5.2.2-all-languages.tar.gz

```

