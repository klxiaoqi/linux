# 1. 静态IP设置

``` bash
sudo vi  /etc/network/interfaces
```

``` ini
auto eno4
iface eno4 inet static
address 10.8.30.176
netmask 255.255.255.0
gateway 10.8.30.1
dns-nameserver 114.114.114.114
```

``` bash
/etc/init.d/networking restart

route add default gw 10.8.30.1                 
```

# 2. 修改hosts

``` bash
sudo vi /etc/hosts

10.8.30.176     anxinyun-m1
10.8.30.177     anxinyun-n1
10.8.30.179     anxinyun-n2

```

# 3. 安装ssh

``` bash
sudo apt-get update
sudo apt-get install openssh-server
sudo apt-get install openssh-client
# 测试是否安装成功
ssh -l anxinyun 10.8.30.179
```

# 4. 配置免密登录

```
修改 /etc/ssh/sshd_config:
RSAAuthentication yes  (启用RSA认证)
PubkeyAuthentication yes (启用公钥私钥配对认证)
AuthorizedKeysFile      %h/.ssh/authorized_keys (公钥文件路径)
```

``` bash
# 重启服务
service ssh restart
```

``` bash
# 生成密钥对
ssh-keygen -t rsa -P ""
# 输出到authorized_keys文件
cat .ssh/id_rsa.pub >> .ssh/authorized_keys
# 设置authorized_keys权限
chmod 600 .ssh/authorized_keys
```

``` bash
# 需要免密登录 哪台主机，就把公钥注册到哪台主机
# 复制 n1 公钥到 m1
scp anxinyun@10.8.30.179:/home/anxinyun/.ssh/id_rsa.pub .
# 追加到 authorized_keys
cat id_rsa.pub >> .ssh/authorized_keys
# 删除 n1 公钥
rm id_rsa.pub

# 在 n1、n2中重复上述命令
```

##  M1 远程免密

``` 
用xshell 用户密钥生成工具生成密钥对
把公钥追加到 .ssh/authorized_keys 
私钥和密钥自己妥善保存

# 禁用ssh用户密码登录
修改 /etc/ssh/sshd_config:
  PasswordAuthentication no
```

# 5. 安装JDK

# 6. 安装Docker

# 7. 安装 PostgreSQL 

```
# 新建 文件 /etc/apt/sources.list.d/pgdg.list 并添加行: 
deb http://apt.postgresql.org/pub/repos/apt/ xenial-pgdg main
```

``` bash
sudo wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

sudo apt-get update

sudo apt-get install postgresql-10

# 安装完毕后自动创建了系统和数据库的postgres用户
# 初始默认数据目录/var/lib/postgresql/9.6/main.  
# 配置文件目录/etc/postgresql/9.6/main.

sudo su     # 切到 root 用户
su postgres # 切到 postgres 用户

```

``` bash
# 设置密码
psql
 \password


psql
>Postgres =# create user "FashionAdmin" with password 'Fas123_';
>Postgres =# create database "AnxinCloud" owner "FashionAdmin";
>Postgres =# GRANT ALL PRIVILEGES ON DATABASE exampledb TO dbuser;

```

``` sql
# 常用命令
\c # 切换数据库
\l # 列出数据库

DROP DATABASE dataBaseName; # 删除数据库
drop role userRole; # 删除用户

grant role1 to role2; # 把用户角色role1 对当前数据库的 权限给用户角色 role2

```

``` bash
# 备份配置文件
cp postgresql.conf postgresql.conf.bak
cp pg_hba.conf pg_hba.conf.bak

# 修改 postgresql.conf
vi postgresql.conf
# listen_addresses = '*'


# 修改 pg_hba.conf
vi pg_hba.conf
# Host   all   all   0.0.0.0/0   md5

```

``` bash
service postgresql restart  # 重启服务
service postgresql stop     # 停止服务
service postgresql start    # 启动服务
```

# 8. 安装 ambari

> 安装 ambari 需要 root 用户
> 以下操作均在 root 用户下进行
> (sudo su)

## 准备工作

* ambari 离线安装包 10G

```
/hdp
|---ambari-2.5.1.0-ubuntu16.tar.gz
|---/hdp
|   ---HDP-2.6.1.0-ubuntu16-deb.tar.gz
|   ---HDP-UTILS-1.1.0.21-ubuntu16.tar.gz

```

* Maximum Open Files Requirements

* 免密登录

``` bash
# ambari 默认是使用root，用户，如果不使用root，也可以使用其他用户，要求 sudo 时不需要输入密码
# 这里使用的 root 用户
sudo su # 切换用户到 root
cd ~ # 切换目录到 root 主目录下
ssh-keygen # 生成密钥对  私钥：id_rsa 公钥：id_rsa.pub
cd .ssh
# 同时在 anxinyun-n1 和 anxinyun-n2 中执行下面命令
cat id_rsa.pub >> authorized_keys

```

* 安装 NTP

``` bash
apt-get install ntp
update-rc.d ntp defaults
```

* 检查DNS

``` bash
hostname -f
```

* 配置 iptables

``` bash
sudo ufw disable
sudo iptables -X
sudo iptables -t nat -F
sudo iptables -t nat -X
sudo iptables -t mangle -F
sudo iptables -t mangle -X
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT
```

* 关闭 SELinux

``` bash
# ubuntu 没有SELinux, 但是有 AppArmor, 它是一个安全扩展（类似于SELinux） 禁用它
service apparmor stop
update-rc.d -f apparmor remove
apt-get remove apparmor apparmor-utils
```

## 设置本地库

* 新建一个http server

``` bash
sudo apt install apache2
# 启动服务
sudo service apache2 start
```

* 创建一个根目录

``` bash
mkdir -p /var/www/html/
```

``` others
http//:10.8.30.176 # http://anxinyun-m1
```

* 解压

``` bash
# 解压 ambari 到 httpd server 根目录
tar -avxf ambari-2.5.1.0-ubuntu16.tar.gz -C /var/www/html/
tar -avxf ./hdp/HDP-2.6.1.0-ubuntu16-deb.tar.gz -C /var/www/html/hdp
tar -avxf ./hdp/HDP-UTILS-1.1.0.21-ubuntu16.tar.gz -C /var/www/html/hdp

# 得到 ./ambari、./hdp/HDP 、./hdp/HDP-UTILS-1.1.0.21 目录
# http//:10.8.30.176/ambari # http://anxinyun-m1/ambari
# http//:10.8.30.176/hdp # http://anxinyun-m1/hdp
```

* 配置本地仓库源

``` bash
cp /var/www/html/ambari/ubuntu16/ambari.list /etc/apt/sources.list.d/ambari.list
vi /etc/apt/sources.list.d/ambari.list
## deb http://anxinyun-m1/ambari/ubuntu16/ Ambari main

```

* 安装 ambari-server

``` bash
cd /var/www/html/ambari/ubuntu16/pool/main/a/ambari-server
dpkg -i ambari-server_2.5.1.0-159_amd64.deb

# 可能会报错 找不到 postgresql。执行以下命令来解决依赖关系
apt-get -f install
# 然后再执行 上面的命令

```

* 初始化 ambari-server

``` bash
ambari-server setup

# Customize user account for ambari-server daemon [y/n] (n)? y
# Enter user account for ambari-server daemon (root):
# Adjusting ambari-server permissions and ownership...
# Checking firewall status...
# Checking JDK...
# Do you want to change Oracle JDK [y/n] (n)? y
# [1] Oracle JDK 1.8 + Java Cryptography Extension (JCE) Policy Files 8
# [2] Oracle JDK 1.7 + Java Cryptography Extension (JCE) Policy Files 7
# [3] Custom JDK
# ==============================================================================
# Enter choice (1): 
# ...
# Enter advanced database configuration [y/n] (n)? y
# Configuring database...
# ==============================================================================
# Choose one of the following options:
# [1] - PostgreSQL (Embedded)
# [2] - Oracle
# [3] - MySQL / MariaDB
# [4] - PostgreSQL
# [5] - Microsoft SQL Server (Tech Preview)
# [6] - SQL Anywhere
# [7] - BDB
# ==============================================================================
# Enter choice (4): 4
# Hostname (localhost):
# Port (5432):
# Database name (ambaridb): ambari
# Postgres schema (ambari):
# Username (ambari):
# Enter Database Password (ambari_):
# Re-enter password:
# ...
# WARNING: Before starting Ambari Server, you must run the following DDL against the database to create the schema: /var/lib/ambari-serDL-Postgres-CREATE.sql
# Proceed with configuring remote database connection properties [y/n] (y)? 
# Extracting system views...
# ............
# Adjusting ambari-server permissions and ownership...
# Ambari Server 'setup' completed successfully.

# 根据提示 执行数据库初始化脚本

# ambari 设置完成

# 开启服务
ambari-server start

# http://anxinyun-m1:8080
# 后面在web界面根据提示安装服务

```
