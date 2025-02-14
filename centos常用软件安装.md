## centos7安装redis6单实例以及集群

### 一、单实例安装

#### 1、安装依赖以及编译安装
  
```bash
# 安装gcc
yum -y install gcc tcl

# 查看gcc版本是否在5.3以上，centos7.6默认安装4.8.5
gcc -v

# 升级到gcc 9.3：
yum -y install centos-release-scl yum -y install devtoolset-9-gcc devtoolset-9-gcc-c++ devtoolset-9-binutils 

# 需要注意的是scl命令启用只是临时的，退出shell或重启就会恢复原系统gcc版本。
scl enable devtoolset-9 bash

# 如果要长期使用gcc 9.3的话：
echo -e "\nsource /opt/rh/devtoolset-9/enable" >>/etc/profile
cd /usr/local/ tar -zxvf redis-6.0.9.tar.gz cd redis-6.0.9 make && make test && make install

# 编译安装到指定目录下
make PREFIX=/usr/local/redis install
mkdir /etc/redis cp redis.conf /etc/redis/6379.conf
cd utils
cp redis_init_script /etc/init.d/redis_6379
chmod 777 /etc/init.d/redis_6379

# 修改配置文件
vim /etc/redis/6379.conf
```
  
配置文件如下：

```conf
# 将这行代码注释，监听所有的ip地址，外网可以访问
#bind 127.0.0.1 
# 把yes改成no，允许外网访问 
protected-mode no 
# 把no改成yes，后台运行 
daemonize yes 
# 开启aof备份
appendonly yes
```

#### 2、配置服务以及开放端口
  
```bash
# 1. 将redis服务添加到开机自启
chkconfig --add redis_6379
# 2. 设置redis开机自启
chkconfig redis_6379 on
# 3. 查看redis 有没有设置为开机启动
chkconfig --list | grep redis 
# 4. 启动redis服务
systemctl start redis_6379.service 
# 5. 开放端口
firewall-cmd --add-port=6379/tcp --permanent
firewall-cmd --reload
```

### 二、安装集群
#### 1 、创建配置，数据，日志文件
  
```bash
# 创建相应的文件目录
mkdir run log data conf

# 创建数据对应rdb文件每个端口对应目录
cd data mkdir 6381 6382 6383 6384 6385 6386

# 复制配置文件
cp redis.conf ../conf/redis_6381.conf

# 修改配置文件
vi redis_6381.conf
```
  
```conf
# 一.允许远程访问
# 注释掉下面代码，或者改为 bind 0.0.0.0，第75行
bind 0.0.0.0

# 关闭保护模式，第94行
protected-mode no

# 二.通用配置
# 开启守护进程，第247行
daemonize yes

# 持久化类型
appendonly yes appendfilename "appendonly-6381.aof"

### 三.集群配置
#配置端口，第98行
port 6381

# 开启集群，第1363行 
cluster-enabled yes 

# 集群节点配置文件，第1371行 
cluster-config-file nodes-6381.conf 

# 进程文件ID对应文件，第279行
pidfile /root/data/redis/run/redis_6381.pid 

# 集群节点超时时间，超过这个时间，集群认为该节点故障，如果是主节点，会进行相应的主从切换，第1377行
cluster-node-timeout 5000

# 四.配置对应目录
logfile /root/data/redis/log/redis_6381.log ###日志文件，第292行
```

####  2、 配置文件以及安装服务
  
```bash
# 将当前6381的配置文件复制为其他端口的配置文件，6382、6383、6384、6385、6386

# 复制配置文件
cp redis_6381.conf redis_6382.conf;cp redis_6381.conf redis_6383.conf;cp redis_6381.conf redis_6384.conf;cp redis_6381.conf redis_6385.conf;cp redis_6381.conf redis_6386.conf

### 创建服务文件
vim /etc/systemd/system/redis-6381.service
```
  
vim批量替换指令

```vim
%s/6381/700*/g
```

服务配置文件如下：

```service
[Unit]
Description=The redis-cluster-server-6381 Process Manager
After=syslog.target network.target

[Service]
Type=forking
ExecStart=/usr/local/bin/redis-server /usr/local/redis/conf/redis_6381.conf
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

#### 3、注册系统服务以及启动、自启动，开放端口设置
  
```bash
# 类似节点一，依次生成其余5个节点配置
vim /etc/systemd/system/redis-6382.service
vim /etc/systemd/system/redis-6383.service
vim /etc/systemd/system/redis-6384.service
vim /etc/systemd/system/redis-6385.service
vim /etc/systemd/system/redis-6386.service

# 重新加载
systemctl daemon-reload

# 系统对应的服务即可
systemctl start redis-6381 
systemctl start redis-6382 
systemctl start redis-6383 
systemctl start redis-6384
systemctl start redis-6385
systemctl start redis-6386 

# 自启动
systemctl enable redis-6381
systemctl enable redis-6382
systemctl enable redis-6383
systemctl enable redis-6384 
systemctl enable redis-6385 
systemctl enable redis-6386

# 开放端口
firewall-cmd --zone=public --add-port=6383/tcp --permanent 
firewall-cmd --zone=public --add-port=6384/tcp --permanent
firewall-cmd --zone=public --add-port=6381/tcp --permanent 
firewall-cmd --zone=public --add-port=6382/tcp --permanent
firewall-cmd --zone=public --add-port=6385/tcp --permanent 
firewall-cmd --zone=public --add-port=6386/tcp --permanent

# 如果是跨服务器的集群需要开放通信端口
firewall-cmd --zone=public --add-port=16381/tcp --permanent 
firewall-cmd --zone=public --add-port=16382/tcp --permanent
firewall-cmd --zone=public --add-port=16383/tcp --permanent
firewall-cmd --zone=public --add-port=16384/tcp --permanent
firewall-cmd --zone=public --add-port=16385/tcp --permanent
firewall-cmd --zone=public --add-port=16386/tcp --permanent
firewall-cmd --reload

# 创建集群
redis-cli --cluster create 192.168.100.247:6381 192.168.100.247:6382 192.168.100.247:6383 192.168.100.247:6384 192.168.100.247:6385 192.168.100.247:6386 --cluster-replicas 1 -a xsm@2024
```

 
## Centos 安装 mysql
### 概述
  
 一般在 `centos` 安装 `mysql` 可以选择以下 2 种方式进行安装：
- `yum install mysql-community-server`
- `rpm ivh mysql-community-**.rpm`
  
这2种方式在最终安装完成后实际上是一致的，都是基于rpm方式安装。在低版本的 `mysql5` 版本，会存在 `tls` 版本问题。
  
```mysql
show variables like '%version%';

# ....
# tls_version TLSv1,TLSv1.1
# ....
```
  
如上所示，低版本的 `mysql5` 不支持 `tlsv1.2` , 而目前无论是高版本的 `jdk` 或者是 `mysql-connector`，基本放弃对低版本的 `tls` 的支持。而使用二进制（`binary`）方式安装时，由于该 tar 包自带了高版本的 openssl 库，此时高版本的 `mysql-connector` 也可以正常连接。

### Binary 安装

#### 准备二进制包
  
进入 [官网下载地址](https://downloads.mysql.com/archives/community/)，这里我选择 5.7.27，选择 `mysql-5.7.27-el7-x86_64.tar.gz` 进行下载。

#### 安装
  
从上一步获取到的 tar 包，先进入到  `/usr/local`，解压，获取二进制的 mysql 包，结构如下

```shell
cd /usr/local
tar zxf mysql-5.7.27-el7-x86_64.tar.gz
```

| Directory       | Contents of Directory                                                                   |
|-----------------|-----------------------------------------------------------------------------------------|
| `bin`           | [**mysqld**](https://dev.mysql.com/doc/refman/8.3/en/mysqld.html "6.3.1 mysqld — The MySQL Server") server, client and utility programs |
| `docs`          | MySQL manual in Info format                                                             |
| `man`           | Unix manual pages                                                                       |
| `include`       | Include (header) files                                                                  |
| `lib`           | Libraries                                                                               |
| `share`         | Error messages, dictionary, and SQL for database installation                           |
| `support-files` | Miscellaneous support files                                                             |

 - mysql 需要特定的用户与组，先创建 mysql 用户与组

```shell
groupadd mysql
# 创建一个只为了提供mysql执行的用户，不需要登录权限
useradd -r -g mysql -s /bin/false mysql
```

- 创建 `mysql-files`，权限为 750，该目录可以作为secure_file_priv系统变量的值使用，该变量限制了导入和导出操作的目录范围。

```shell
mkdir mysql-files
chmod 750 mysql-files
```

- 将 mysql 文件夹的所属权修改为mysql组下的mysql用户

```shell
chown mysql:mysql /usr/local/mysql
```

- 创建mysql 的配置文件 , `vim /etc/my.cnf` （启动后再创建会报错，处理较麻烦，故提前创建）
  
```cnf
[mysqld]
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data
port=3306
#InnoDB缓冲池大小
innodb_buffer_pool_size = 2G
#InnoDB缓冲池实例个数 innodb_buffer_pool_size / innodb_buffer_pool_instances = 1G进行设置
innodb_buffer_pool_instances = 1 
#InnoDB日志文件大小
innodb_log_file_size = 300M
#InnoDB事务日志刷盘时机
innodb_flush_log_at_trx_commit = 1 
max_allowed_packet=10M
#最大连接数
max_connections = 5000
#innodb数据文件及redo log的打开、刷写模式，提升写入性能。
innodb_flush_method = O_DIRECT
#关闭指定datadir之外的分区或目录存储
symbolic-links=0
disable_log_bin # 禁用 bin log
lower_case_table_names= 1
server_id = 1 
log-error = /usr/local/mysql/data/mysqld.err
auto-increment-offset = 1 
sql-mode="STRICT_ALL_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_ZERO_IN_DATE"
log-bin-trust-function-creators=1
max_allowed_packet = 10G 
default-time-zone = '+08:00'
```

- 初始化mysql，ssl，启动mysql

```shell
bin/mysqld --initialize --user=mysql
# 8.0.34版本后无需手动调用，第一步初始化会自动创建证书文件，且下面命令已经被移除
# bin/mysql_ssl_rsa_setup 
bin/mysqld_safe --user=mysql &
# 将mysql bin 目录加入path
echo "export PATH=$PATH:/data/mysql/bin" >> /etc/profile && source /etc/profile
```

- 将 mysql 设置为开机启动，并设置系统服务

```shell
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
chkconfig --add mysqld
# 先关闭mysql服务
mysqladmin -uroot -p shutdown
# 使用服务模式启动mysql
systemctl start mysql
```

- 修改默认用户密码，创建用于远程连接的root用户
  
```sql
-- 本地客户端的root用户密码修改
alter user root@'localhost' identified by "123456";
      
-- 创建远程的root用户
create user root@'%' identified by '123456';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%';
```

- 修改 `/etc/my.cnf` , 启用binlog
  
```cnf
[mysqld]
# 开启binlog，配置binlog文件的前缀
log-bin=/opt/module/mysql/binlog/mysql-bin
server-id=1
```

### mysql 数据库导入导出
  
```shell
mysqldump --user root -p --databases origin_dict --routines > test.sql
mysql --default-character-set=utf8mb4 --database=db --user=root -p --host=10.1.139.42 --port=3306 < test.sql
```


##  Miniconda 安装
- 到[官网](https://docs.anaconda.com/free/miniconda/)下载 [`sh` 文件](https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh)

- 执行安装
  
```shell
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
# 安装完成，删除 sh 文件
rm -rf ~/miniconda3/miniconda.sh
```

- 初始化
  
```shell
~/miniconda3/bin/conda init bash
```