# PostgreSQL

## 安装前准备

在集群中每台机器上执行

### 服务裁剪

```shell
# 关闭不必要的OS服务
chkconfig --list|grep on  
# 关闭不必要的,例如 
chkconfig iscsi off
```



### 防火墙设置

```shell
# 仅支持内部网段访问
# 私有网段 配置范例
iptables -A INPUT -s 192.168.0.0/16 -j ACCEPT
iptables -A INPUT -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -s 172.16.0.0/16 -j ACCEPT

# 内部测试环境可以直接关闭防火墙
systemctl disable firewalld
systemctl stop firewalld
systemctl status firewalld

# systemctl enable firewalld
```

### 禁用selinux

```shell
# CentOS 7
sed -i s/"SELINUX=enforcing"/"SELINUX=disabled"/g /etc/selinux/config

# CentOS 6
sed -i s/"SELINUX=enforcing"/"SELINUX=disabled"/g /etc/sysconfig/selinux
# 说明：CentOS7中 该文件为软链接，指向/etc/selinux/config，所以不能直接修改软链接。

# 检查是否修改成功
sestatus
# 重启服务器
reboot
```

### 系统内核参数优化

```shell
vi /etc/sysctl.conf

fs.aio-max-nr = 1048576
fs.file-max = 76724600
kernel.core_pattern= /data/pgsql/core/core_%e_%u_%t_%s.%p         
# /data/pgsql/core事先建好，权限777，如果是软链接，对应的目录修改为777
kernel.sem = 4096 2147483647 2147483646 512000    
# 信号量, ipcs -l 或 -u 查看，每16个进程一组，每组信号量需要17个信号量。
kernel.shmall = 107374182      
# 所有共享内存段相加大小限制(建议内存的80%)
kernel.shmmax = 274877906944   
# 最大单个共享内存段大小(建议为内存一半), >9.2的版本已大幅降低共享内存的使用
kernel.shmmni = 819200         
# 一共能生成多少共享内存段，每个PG数据库集群至少2个共享内存段
net.core.netdev_max_backlog = 10000
net.core.rmem_default = 262144       
# The default setting of the socket receive buffer in bytes.
net.core.rmem_max = 4194304          
# The maximum receive socket buffer size in bytes
net.core.wmem_default = 262144       
# The default setting (in bytes) of the socket send buffer.
net.core.wmem_max = 4194304          
# The maximum send socket buffer size in bytes.
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_keepalive_intvl = 20
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_mem = 8388608 12582912 16777216
net.ipv4.tcp_fin_timeout = 5
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syncookies = 1    
# 开启SYN Cookies。当出现SYN等待队列溢出时，启用cookie来处理，可防范少量的SYN攻击
net.ipv4.tcp_timestamps = 1    
# 减少time_wait
net.ipv4.tcp_tw_recycle = 0    
# 如果=1则开启TCP连接中TIME-WAIT套接字的快速回收，但是NAT环境可能导致连接失败，建议服务端关闭它
net.ipv4.tcp_tw_reuse = 1      
# 开启重用。允许将TIME-WAIT套接字重新用于新的TCP连接
net.ipv4.tcp_max_tw_buckets = 262144
net.ipv4.tcp_rmem = 8192 87380 16777216
net.ipv4.tcp_wmem = 8192 65536 16777216
net.nf_conntrack_max = 1200000
net.netfilter.nf_conntrack_max = 1200000
vm.dirty_background_bytes = 409600000       
#  系统脏页到达这个值，系统后台刷脏页调度进程 pdflush（或其他） 自动将(dirty_expire_centisecs/100）秒前的脏页刷到磁盘
vm.dirty_expire_centisecs = 3000             
#  比这个值老的脏页，将被刷到磁盘。3000表示30秒。
vm.dirty_ratio = 95                          
#  如果系统进程刷脏页太慢，使得系统脏页超过内存 95 % 时，则用户进程如果有写磁盘的操作（如fsync, fdatasync等调用），则需要主动把系统脏页刷出。
#  有效防止用户进程刷脏页，在单机多实例，并且使用CGROUP限制单实例IOPS的情况下非常有效。  
vm.dirty_writeback_centisecs = 100            
#  pdflush（或其他）后台刷脏页进程的唤醒间隔， 100表示1秒。
vm.mmap_min_addr = 65536
vm.overcommit_memory = 0     
#  在分配内存时，允许少量over malloc, 如果设置为 1, 则认为总是有足够的内存，内存较少的测试环境可以使用 1 .  
vm.overcommit_ratio = 90     
#  当overcommit_memory = 2 时，用于参与计算允许指派的内存大小。
vm.swappiness = 0            
#  关闭交换分区
vm.zone_reclaim_mode = 0     
# 禁用 numa, 或者在vmlinux中禁止. 
net.ipv4.ip_local_port_range = 40000 65535    
# 本地自动分配的TCP, UDP端口号范围
fs.nr_open=20480000
# 单个进程允许打开的文件句柄上限
net.ipv4.tcp_max_syn_backlog = 16384
net.core.somaxconn = 16384

# 以下参数请注意
# vm.extra_free_kbytes = 4096000
# vm.min_free_kbytes = 2097152 # vm.min_free_kbytes 建议每32G内存分配1G vm.min_free_kbytes
# 如果是小内存机器，以上两个值不建议设置
# vm.nr_hugepages = 66536    
#  建议shared buffer设置超过64GB时 使用大页，页大小 /proc/meminfo Hugepagesize
# vm.lowmem_reserve_ratio = 1 1 1
# 对于内存大于64G时，建议设置，否则建议默认值 256 256 32

```

### 配置OS资源限制

```
# vi /etc/security/limits.conf

# nofile超过1048576的话，一定要先将sysctl的fs.nr_open设置为更大的值，并生效后才能继续设置nofile.

* soft    nofile  1024000
* hard    nofile  1024000
* soft    nproc   unlimited
* hard    nproc   unlimited
* soft    core    unlimited
* hard    core    unlimited
* soft    memlock unlimited
* hard    memlock unlimited
```

### 其他系统设置

```shell
# 本地时区设置
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime 
# 字符集添加到环境变量
echo "export LANG=en_US.utf8" >> /root/.bash_profile 
```





### 数据库目录准备

1. 创建数据库base目录

   ```shell
   mkdir -p /data/pgsql
   ```

2. 物理机必须挂载单独的磁盘阵列给数据库分区，尽量不要与系统盘使用同一块盘。将分区mount到 /data/postgres 目录下，并将挂载命令放到fstab里面，开机自动调用。虚拟机无需挂载。

   ```shell
   # 挂载, 写入vi /etc/fstab文件
   mount  /sdb1 /data/pgsql
   ```

3. 创建数据库目录、日志目录等

   ```shell
   # 数据目录
   mkdir -p /data/pgsql/data
   # 备份目录,需要单独挂载盘，最好是网络存储，可以通过nfs 或者samba挂载
   mkdir -p /data/pgsql/backups
   # 归档日志目录,需要单独挂载盘，最好是网络存储，可以通过nfs 或者samba挂载
   mkdir -p /data/pgsql/archive_wals
   # core文件存放目录
   mkdir -p /data/pgsql/core
   ```



### 建立信任

在集群中每台机器上执行，首先执行一下ssh 保证known_hosts里面存在该ip

```shell
cd /home/rds_postgresql/packages
rpm -ivh sshpass-1.06-2.el7.x86_64.rpm
ssh-keygen -t rsa -P "" -f ~/.ssh/id_rsa
sshpass -p landleaf@123.com ssh-copy-id -i /root/.ssh/id_rsa.pub root@172.20.11.200
sshpass -p landleaf@123.com ssh-copy-id -i /root/.ssh/id_rsa.pub root@172.20.11.201
sshpass -p landleaf@123.com ssh-copy-id -i /root/.ssh/id_rsa.pub root@172.20.11.202
```





## 安装PostgreSQL12软件

在所有机器上执行

### 上传安装包并安装

将安装包rds_postgresql上传到集群中每个节点的 /home 目录

```shell
cd /home/rds_postgresql/packages
yum localinstall *.rpm
```





### 设置环境变量

```shell
vim /etc/profile.d/pg_env.sh

#export PS1="$USER@`/bin/hostname -s`-> "
export PGPORT=5432
export PGDATA=/data/pgsql/data
export LANG=en_US.utf8
export PGHOME=/usr/pgsql-12/
export LD_LIBRARY_PATH=$PGHOME/lib:/lib64:/usr/lib64:/usr/local/lib64:/lib:/usr/lib:/usr/local/lib:$LD_LIBRARY_PATH
export PATH=$PGHOME/bin:$PATH:.
export DATE=`date +"%Y%m%d%H%M"`
export MANPATH=$PGHOME/share/man:$MANPATH
export PGHOST=$PGDATA
export PGUSER=postgres
export PGDATABASE=postgres
alias rm='rm -i'
alias ll='ls -lh'

# 将环境变量文件加到postgres用户的环境变量文件
chmod 777 /etc/profile.d/pg_env.sh
echo "source /etc/profile.d/pg_env.sh" >> /var/lib/pgsql/.bash_profile
/etc/profile.d/pg_env.sh

```



### 初始化数据目录

```shell
# 修改数据目录权限
chown -R postgres.postgres /data/pgsql

# 初始化数据库
su - postgres -c "initdb -E utf8 --lc-collate=C --lc-ctype=en_US.utf8" 
```



### PostgreSQL参数动态优化

根据实际情况修改配置文件模版：

  

```shell
##########################
###### 需要修改的项目 ######
#########################
shared_buffers = $SHARED_BUFFERS                      # 1/4 主机内存。例如：128GB
maintenance_work_mem = $MAIN_WORK_MEM                  # min( 2GB, (1/4 主机内存)/autovacuum_max_workers )
wal_buffers = $WAL_BUFFERS       # min( 2047MB, shared_buffers/32 ) = 512MB
max_wal_size = $MAX_WAL_SIZE       # 建议是SHARED BUFFER的2倍。例如：256GB 
min_wal_size = $MIN_WAL_SIZE        # max_wal_size/4
autovacuum_max_workers = $AUTOVACUUM_MAX_WORKERS     # CPU核多，并且IO好的情况下，可多点，但是注意16*autovacuum mem，会消耗较多内存
effective_cache_size = $EFFECTIVE_CACHE_SIZE    # 看着办，扣掉会话连接RSS，shared buffer, autovacuum worker, 剩下的都是OS可用的CACHE。

######################

vi postgresql.conf
 
listen_addresses = '0.0.0.0'
port = 5432
max_connections = 5000
unix_socket_directories = '.'
tcp_keepalives_idle = 60
tcp_keepalives_interval = 10
tcp_keepalives_count = 10
shared_buffers = $SHARED_BUFFERS                      # 1/4 主机内存。例如：128GB
maintenance_work_mem = $MAIN_WORK_MEM                  # min( 2G, (1/4 主机内存)/autovacuum_max_workers )
dynamic_shared_memory_type = posix
vacuum_cost_delay = 0
bgwriter_delay = 10ms
bgwriter_lru_maxpages = 1000
bgwriter_lru_multiplier = 10.0
bgwriter_flush_after = 0                    # IO很好的机器，不需要考虑平滑调度
max_worker_processes = 128
max_parallel_workers_per_gather = 2         #  如果需要使用并行查询，设置为大于1 ，不建议超过 主机cores-2
old_snapshot_threshold = -1
backend_flush_after = 0  # IO很好的机器，不需要考虑平滑调度, 否则建议128~256kB
wal_level = replica
synchronous_commit = off
full_page_writes = on   # 支持原子写超过BLOCK_SIZE的块设备，在对齐后可以关闭。或者支持cow的文件系统可以关闭。
wal_buffers = $WAL_BUFFERS       # min( 2047MB, shared_buffers/32 ) = 512MB
wal_writer_delay = 10ms
wal_writer_flush_after = 0  # IO很好的机器，不需要考虑平滑调度, 否则建议128~256kB
checkpoint_timeout = 30min  # 不建议频繁做检查点，否则XLOG会产生很多的FULL PAGE WRITE(when full_page_writes=on)。
max_wal_size = $MAX_WAL_SIZE       # 建议是SHARED BUFFER的2倍。例如：256GB 
min_wal_size = $MIN_WAL_SIZE        # max_wal_size/4
checkpoint_completion_target = 0.05          # 硬盘好的情况下，可以让检查点快速结束，恢复时也可以快速达到一致状态。否则建议0.5~0.9
checkpoint_flush_after = 0                   # IO很好的机器，不需要考虑平滑调度, 否则建议128~256kB
archive_mode = on
archive_command = 'test ! -f /data/pgsql/archive_wals/%f && lz4 -q -z %p /data/pgsql/archive_wals/%f.lz4 ; find /data/pgsql/archive_wals/ -type f -mtime +7 -exec rm -f {} \;'      #  后期再修改,默认保存7天的归档日志
max_wal_senders = 8
random_page_cost = 1.3  # IO很好的机器，不需要考虑离散和顺序扫描的成本差异
parallel_tuple_cost = 0
parallel_setup_cost = 0
effective_cache_size = $EFFECTIVE_CACHE_SIZE    # 看着办，扣掉会话连接RSS，shared buffer, autovacuum worker, 剩下的都是OS可用的CACHE。
force_parallel_mode = off
log_destination = 'csvlog'
logging_collector = on
log_truncate_on_rotation = on
log_checkpoints = on
log_connections = on
log_disconnections = on
log_error_verbosity = verbose
log_timezone = 'PRC'
vacuum_defer_cleanup_age = 0
hot_standby_feedback = off                             # 建议关闭，以免备库长事务导致 主库无法回收垃圾而膨胀。
max_standby_archive_delay = 300s
max_standby_streaming_delay = 300s
autovacuum = on
log_autovacuum_min_duration = 0
log_filename = 'postgresql.log.%y-%m-%d'
log_rotation_size = 100MB
log_statement=mod
log_min_duration_statement = 2s 
log_lock_waits=on
autovacuum_max_workers = $AUTOVACUUM_MAX_WORKERS  # CPU核多，并且IO好的情况下，可多点，但是注意16*autovacuum mem，会消耗较多内存  
autovacuum_naptime = 45s                               # 建议不要太高频率，否则会因为vacuum产生较多的XLOG。
autovacuum_vacuum_scale_factor = 0.1
autovacuum_analyze_scale_factor = 0.1
autovacuum_freeze_max_age = 1600000000
autovacuum_multixact_freeze_max_age = 1600000000
vacuum_freeze_table_age = 1500000000
vacuum_multixact_freeze_table_age = 1500000000
datestyle = 'iso, mdy'
timezone = 'PRC'
lc_messages = 'C'
lc_monetary = 'C'
lc_numeric = 'C'
lc_time = 'C'
default_text_search_config = 'pg_catalog.english'
shared_preload_libraries='pg_stat_statements'

## 如果你的数据库有非常多小文件（比如有几十万以上的表，还有索引等，并且每张表都会被访问到时），建议FD可以设多一些，避免进程需要打开关闭文件。
## 但是不要大于前面章节系统设置的ulimit -n(open files)
max_files_per_process=655360
```



### 配置pg_hba.conf



```shell
vi pg_hba.conf

host replication repl 0.0.0.0/0 md5  # 流复制，流复制用户:repl
host all postgres 0.0.0.0/0 reject # 拒绝超级用户从网络登录
host all all 0.0.0.0/0 md5  # 其他用户登陆
```



### 安装其他插件



```shell
# 安装PostGIS、pgRouting
cd /home/rds_postgresql/packages/extension/postgis/
yum localinstall *.rpm

# 安装TimescaleDB
cd /home/rds_postgresql/packages/extension
rpm -ivh timescaledb_12-1.7.0-1.rhel7.x86_64.rpm
# 安装Citus
cd /home/rds_postgresql/packages/extension
rpm -ivh citus_12-9.3.0-1.rhel7.x86_64.rpm
# 安装hll
cd /home/rds_postgresql/packages/extension
rpm -ivh hll_12-2.14-1.rhel7.x86_64.rpm

# 修改配置文件 的 shared_preload_libraries 配置项
vim /data/pgsql/data/postgresql.conf
# 设置内容
shared_preload_libraries='citus,timescaledb,pg_stat_statements'
```



### 启动数据库



```shell
 su - postgres -c "pg_ctl start"
 # 修改postgres用户密码
 su - postgres
 psql -c "ALTER USER postgres WITH PASSWORD 'landleaf@123.com'"
```



## 部署高可用流复制



PostgreSQL 12 的PGDATA中不再有recovery.conf文件，现在recovery.conf的所有参数现在都放在了postgresql.conf文件中。并且在备用服务器的群集数据目录中应为名为standby.signal的文件以触发备用模式。

```shell
# 测试环境
#主节点：172.20.11.200
#备节点：172.20.11.201
#备节点：172.20.11.202

```



### 主节点上创建复制用户



```sql
-- 创建复制用户 repl，并有REPLICATION和LOGIN权限，与SUPERUSER相反，REPLICATION不允许修改任何数据。
su - postgres
psql
create user repl replication login password 'landleaf@123.com';
```

###  设置用户访问策略

```shell
vim /data/pgsql/data/pg_hba.conf
# 确保有以下行
host    replication     repl        172.20.11.200/32        md5
host    replication     repl        172.20.11.201/32        md5
host    replication     repl        172.20.11.202/32        md5
```



### **在主服务器上创建从库的复制slot**

```sql
-- 201 从库的复制槽
SELECT * FROM pg_create_physical_replication_slot('pg_slot_01');
-- 202 从库的复制槽
SELECT * FROM pg_create_physical_replication_slot('pg_slot_02');

```



### 在每台从库同步主库的基础备份

```shell
su - postgres
# -R 生成postgresql.auto.conf 文件，记录流复制配置信息。同时生成standby.signal文件
# -X s 指定在备份期间产生的wal日志以什么形式传送到备库，s 表示流式
# -U 指定流复制用户
# -S 当采用流复制的情况下，指定复制槽名,注意要指定从库自己的复制槽
# 201 从库上执行
pg_basebackup -h 172.20.11.200 -p 5432 -U repl -W -Fp -Xs -Pv -R -D /data/pgsql/data/  -S pg_slot_01
# 202 从库上执行
pg_basebackup -h 172.20.11.200 -p 5432 -U repl -W -Fp -Xs -Pv -R -D /data/pgsql/data/  -S pg_slot_02
```



### 启动从库节点

```shell
pg_ctl start
```



### 检查是否部署成功

```shell
#  主库上执行,查看是否启动walsender 进程
ps -ef | grep walsender

postgres 25081  3394  0 17:15 ?        00:00:00 postgres: walsender repl 172.20.11.202(52312) streaming 0/7023AC0
root     26102 22005  0 17:30 pts/2    00:00:00 grep --color=auto walsender

# 从库上执行，检查从库上是否启动 walreceiver 进程
root@dbtest02-> ps -ef | grep walreceiver
postgres  3333  3326  0 17:15 ?        00:00:01 postgres: walreceiver   streaming 0/7023C40
root      3640  2156  0 17:34 pts/1    00:00:00 grep --color=auto walreceiver

# 在主库上创建一张表，并插入数据，检查从库是否有同步
```



## 部署故障切换-Patroni

### 集群信息

**软件信息**

- 数据库: PostgreSQL 12.3
- Python: Python 3.8.2
- Etcd: etcd-v3.4.7
- Patroni: patroni 1.6.5

**部署规划**

|   主机   |      IP       |           组件            |  备注  |
| :------: | :-----------: | :-----------------------: | :----: |
| dbtest00 | 172.20.11.200 | PostgreSQL、Patroni、Etcd | 主节点 |
| dbtest01 | 172.20.11.201 | PostgreSQL、Patroni、Etcd | 备节点 |
| dbtest02 | 172.20.11.202 | PostgreSQL、Patroni、Etcd | 备节点 |

### 前置条件

#### 数据库流复制部署

部署patroni的前提条件是成功安装好postgreql12.3 ，且配置好主从流复制

#### 升级到python3

```shell
# 安装前置条件
yum -y install zlib*
yum -y install gcc
# 如果没有下载，执行下载，wget -c https://www.python.org/ftp/python/3.8.2/Python-3.8.2.tar.xz
cd /home/rds_postgresql/packages/patroni
tar xvf Python-3.8.2.tar.xz     
cd Python-3.8.2    
./configure
make
make install
```



创建软链接,如下:

```
rm -f /usr/bin/python
ln -s /usr/local/bin/python3 /usr/bin/python
```



python3安装后’yum’命令执行会报错，需要修改以下配置:

```
/usr/bin/yum: 将文件第一行改为/usr/bin/python2.7。(2.7.x也改为2.7)
/usr/libexec/urlgrabber-ext-down: 将文件第一行改为/usr/bin/python2.7。
```



####  安装virtualenv

```shell
yum install -y epel-release
yum install python3-pip
yum install python3-devel

#  安装virtualenv
pip3 install virtualenv
     
# 创建虚拟环境目录
mkdir -p /home/local_virtualenv/
cd /home/local_virtualenv/

# 创建虚拟环境
cp -r /home/rds_postgresql/packages/patroni/myEnv  /home/local_virtualenv/

# 切换到虚拟环境
source /home/local_virtualenv/myEnv/bin/activate

```

####  主库执行创建用来执行pg_rewind的用户

```shell
su - postgres
psql -c "create user rewind with superuser PASSWORD 'landleaf@123.com'"
# 修改pg_hba.conf，增加以下内容

host    all             rewind        172.20.11.200/32               md5
host    all             rewind        172.20.11.201/32               md5
host    all             rewind        172.20.11.202/32               md5

# 使修改生效
su - postgres -c 'pg_ctl reload'
```





### Etcd安装



#### 三台主机执行安装



##### 安装软件

```sh
cd /home/rds_postgresql/packages/patroni/
rpm -ivh etcd-3.3.11-2.el7.centos.x86_64.rpm

```



##### 修改配置文件

172.20.11.200

```shell
cp /etc/etcd/etcd.conf /etc/etcd/etcd.conf.bak
# 创建etcd工作目录
mkdir -p /home/etcd/node1.etcd
chown -R etcd.etcd /home/etcd
# 修改配置
vim /etc/etcd/etcd.conf
# 填充以下内容
ETCD_DATA_DIR="/home/etcd/node0.etcd"
ETCD_LISTEN_PEER_URLS="http://172.20.11.200:2380"
ETCD_LISTEN_CLIENT_URLS="http://172.20.11.200:2379,http://127.0.0.1:2379"
ETCD_NAME="node0"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.20.11.200:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://172.20.11.200:2379"
ETCD_INITIAL_CLUSTER="node0=http://172.20.11.200:2380,node1=http://172.20.11.201:2380,node2=http://172.20.11.202:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```



172.20.11.201

```shell
cp /etc/etcd/etcd.conf /etc/etcd/etcd.conf.bak
# 创建etcd工作目录
mkdir -p /home/etcd/node1.etcd
chown -R etcd.etcd /home/etcd
# 修改配置
vim /etc/etcd/etcd.conf
# 填充以下内容
ETCD_DATA_DIR="/home/etcd/node1.etcd"
ETCD_LISTEN_PEER_URLS="http://172.20.11.201:2380"
ETCD_LISTEN_CLIENT_URLS="http://172.20.11.201:2379,http://127.0.0.1:2379"
ETCD_NAME="node1"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.20.11.201:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://172.20.11.201:2379"
ETCD_INITIAL_CLUSTER="node0=http://172.20.11.200:2380,node1=http://172.20.11.201:2380,node2=http://172.20.11.202:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```



172.20.11.202 

```shell
cp /etc/etcd/etcd.conf /etc/etcd/etcd.conf.bak
# 创建etcd工作目录
mkdir /home/etcd/node2.etcd
chown -R etcd.etcd /home/etcd
# 修改配置
vim /etc/etcd/etcd.conf
# 填充以下内容
ETCD_DATA_DIR="/home/etcd/node1.etcd"
ETCD_LISTEN_PEER_URLS="http://172.20.11.202:2380"
ETCD_LISTEN_CLIENT_URLS="http://172.20.11.202:2379,http://127.0.0.1:2379"
ETCD_NAME="node2"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.20.11.202:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://172.20.11.202:2379"
ETCD_INITIAL_CLUSTER="node0=http://172.20.11.200:2380,node1=http://172.20.11.201:2380,node2=http://172.20.11.202:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```





##### 修改系统服务配置文件

```shell
vim /usr/lib/systemd/system/etcd.service 

# 内容如下，修改WorkingDirectory 字段
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/home/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/bin/etcd --name=\"${ETCD_NAME}\" --data-dir=\"${ETCD_DATA_DIR}\" --listen-client-urls=\"${ETCD_LISTEN_CLIENT_URLS}\""
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```



#### 启动 etcd

依次启动 node0、node1、node2 节点的 etcd

```sh
systemctl start etcd.service
systemctl status etcd.service
systemctl enable etcd.service
```

```shell
# 验证 etcd

etcdctl cluster-health

# 列出成员
etcdctl member list
```

![](https://img2018.cnblogs.com/blog/1516162/201908/1516162-20190825163727533-1798072991.png)

#### 定时对时

各个节点配置crontab 定时同步时间，否则可能会出现node1 etcd[1657]: the clock difference against peer f63afbe816fb463d is too hig问题

```shell
cd /home/rds_postgresql/packages/patroni

yum install ntp*
crontab -e
0 1 * * * /usr/sbin/ntpdate 172.20.11.205 >> /var/log/ntpdate.log 2>&1 &
# 172.20.11.205  改成对时服务器的ip或者域名
```

　　

### Patroni 安装

#### 在三个节点上安装软件

```shell
# 将virtualenv 虚拟机环境中的启动脚本 软连接到 全局环境变量支持的目录
ln -s /home/local_virtualenv/myEnv/bin/patroni /usr/bin/patroni
ln -s /home/local_virtualenv/myEnv/bin/patronictl /usr/bin/patronictl

# 创建patroni 工作目录，并创建配置文件
mkdir -p /home/patroni/{conf,log}
chown -R postgres:postgres /home/patroni

cd /home/patroni/conf
vim patroni_postgresql.yml

#######################################################################################
#######											 填入一下内容172.20.11.200为例 			 				 #################
##      	仅需修改全局参数name、restapi模块的listen和connect_address参数			   	 ##########  
###       以及etcd模块的host参数，以及postgresql模块的connect_address参数。  	   ###########
###       注意 postgresql 部分的配置文件会覆盖数据库本身的配置，需要注意            ###########
#######################################################################################
scope: pg_ydtf
namespace: /service/
name: pg_ydtf00

restapi:
  listen: 172.20.11.200:8008
  connect_address: 172.20.11.200:8008

etcd:
  #Provide host to do the initial discovery of the cluster topology:
  host: 172.20.11.200:2379

bootstrap:
  # this section will be written into Etcd:/<namespace>/<scope>/config after initializing new cluster
  # and all other cluster members will use it as a `global configuration`
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    master_start_timeout: 300
    synchronous_mode: false
    #standby_cluster:
      #host: 127.0.0.1
      #port: 1111
      #primary_slot_name: patroni
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: locical
        hot_standby: "on"
        wal_keep_segments: 128
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        archive_mode: "on"
        # primary_conninfo: 'host=172.20.11.200 port=1921 user=repuser'
        hot_standby: on
        archive_timeout: 1800s
        synchronous_commit: on
        synchronous_standby_names: 'ANY 1 (pg_ydtf01, pg_ydtf02)'
      recovery_conf:
        restore_command: lz4 -f -q -d /data/pgsql/archive_wals/%f.lz4 %p

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 172.20.11.200:5432
  data_dir: /data/pgsql/data
  bin_dir: /usr/pgsql-12/bin
  config_dir: /data/pgsql/data
  #pgpass: /home/pg12/patroni/.pgpass
  authentication:
    replication:
      username: repl
      password: landleaf@123.com
    superuser:
      username: postgres
      password: landleaf@123.com
    rewind:  # Has no effect on postgres 10 and lower
      username: rewind
      password: landleaf@123.com

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false

```

#### 三个节点上执行启动

```shell
su - postgres
source /home/local_virtualenv/myEnv/bin/activate
cd /home/local_virtualenv/myEnv/bin
patroni /home/patroni/conf/patroni_postgresql.yml >> /home/patroni/log/patroni.log  2>&1 &

```



#### 常用命令



##### 查看集群状态

```sh
# 安装格式化工具
yum install jq

curl -s "http://172.20.11.201:8008/cluster" | jq .
{
  "members": [
    {
      "name": "pg_ydtf00",
      "role": "leader",
      "state": "running",
      "api_url": "http://172.20.11.200:8008/patroni",
      "host": "172.20.11.200",
      "port": 5432,
      "timeline": 1
    },
    {
      "name": "pg_ydtf01",
      "role": "replica",
      "state": "running",
      "api_url": "http://172.20.11.201:8008/patroni",
      "host": "172.20.11.201",
      "port": 5432,
      "timeline": 1,
      "pending_restart": true,
      "lag": 0
    },
    {
      "name": "pg_ydtf02",
      "role": "replica",
      "state": "running",
      "api_url": "http://172.20.11.202:8008/patroni",
      "host": "172.20.11.202",
      "port": 5432,
      "timeline": 1,
      "pending_restart": true,
      "lag": 0
    }
  ]
}

  
patronictl -c /home/patroni/conf/patroni_postgresql.yml list
+ Cluster: pg_ydtf (6846936437991638436) ------+----+-----------+-----------------+
|   Member  |      Host     |  Role  |  State  | TL | Lag in MB | Pending restart |
+-----------+---------------+--------+---------+----+-----------+-----------------+
| pg_ydtf00 | 172.20.11.200 | Leader | running |  1 |           |                 |
| pg_ydtf01 | 172.20.11.201 |        | running |  1 |         0 |        *        |
| pg_ydtf02 | 172.20.11.202 |        | running |  1 |         0 |        *        |
+-----------+---------------+--------+---------+----+-----------+-----------------+

```





##### 如何手动切换Leader

```shell
## ########## 执行命令 ##############
su - postgres
patronictl -c /home/patroni/conf/patroni_postgresql.yml switchover

##########################################
##### 						输出						#########
##########################################

[root@localhost log]# patronictl -c /home/patroni/conf/patroni_postgresql.yml switchover
Master [pg_ydtf00]: 
Candidate ['pg_ydtf01', 'pg_ydtf02'] []: pg_ydtf01
When should the switchover take place (e.g. 2020-07-08T15:18 )  [now]: 
Current cluster topology
+ Cluster: pg_ydtf (6846936437991638436) ------+----+-----------+-----------------+
|   Member  |      Host     |  Role  |  State  | TL | Lag in MB | Pending restart |
+-----------+---------------+--------+---------+----+-----------+-----------------+
| pg_ydtf00 | 172.20.11.200 | Leader | running |  1 |           |                 |
| pg_ydtf01 | 172.20.11.201 |        | running |  1 |         0 |        *        |
| pg_ydtf02 | 172.20.11.202 |        | running |  1 |         0 |        *        |
+-----------+---------------+--------+---------+----+-----------+-----------------+
Are you sure you want to switchover cluster pg_ydtf, demoting current master pg_ydtf00? [y/N]: y
2020-07-08 14:18:47.56676 Successfully switched over to "pg_ydtf01"
+ Cluster: pg_ydtf (6846936437991638436) ------+----+-----------+-----------------+
|   Member  |      Host     |  Role  |  State  | TL | Lag in MB | Pending restart |
+-----------+---------------+--------+---------+----+-----------+-----------------+
| pg_ydtf00 | 172.20.11.200 |        | stopped |    |   unknown |                 |
| pg_ydtf01 | 172.20.11.201 | Leader | running |  1 |           |        *        |
| pg_ydtf02 | 172.20.11.202 |        | running |  1 |         0 |        *        |
+-----------+---------------+--------+---------+----+-----------+-----------------+

####################################################

#######   							查看集群状态 					########
####################################################

curl -s "http://172.20.11.202:8008/cluster" | jq .
{
  "members": [
    {
      "name": "pg_ydtf00",
      "role": "replica",
      "state": "stopped",
      "api_url": "http://172.20.11.200:8008/patroni",
      "host": "172.20.11.200",
      "port": 5432,
      "lag": "unknown"
    },
    {
      "name": "pg_ydtf01",
      "role": "leader",
      "state": "running",
      "api_url": "http://172.20.11.201:8008/patroni",
      "host": "172.20.11.201",
      "port": 5432,
      "timeline": 2,
      "pending_restart": true
    },
    {
      "name": "pg_ydtf02",
      "role": "replica",
      "state": "running",
      "api_url": "http://172.20.11.202:8008/patroni",
      "host": "172.20.11.202",
      "port": 5432,
      "timeline": 1,
      "pending_restart": true,
      "lag": 16773992
    }
  ]
}


##### 稍等几分钟之后观察旧主机器状态改为running

curl -s "http://172.20.11.202:8008/cluster" | jq .
{
  "members": [
    {
      "name": "pg_ydtf00",
      "role": "replica",
      "state": "running",
      "api_url": "http://172.20.11.200:8008/patroni",
      "host": "172.20.11.200",
      "port": 5432,
      "timeline": 2,
      "pending_restart": true,
      "lag": 0
    },
    {
      "name": "pg_ydtf01",
      "role": "leader",
      "state": "running",
      "api_url": "http://172.20.11.201:8008/patroni",
      "host": "172.20.11.201",
      "port": 5432,
      "timeline": 2,
      "pending_restart": true
    },
    {
      "name": "pg_ydtf02",
      "role": "replica",
      "state": "running",
      "api_url": "http://172.20.11.202:8008/patroni",
      "host": "172.20.11.202",
      "port": 5432,
      "timeline": 2,
      "pending_restart": true,
      "lag": 0
    }
  ]
}






```



#### 其他命令

更多命令参考 https://patroni.readthedocs.io/en/latest/rest_api.html



## 分片Citus实践

### **部署规划**

|   主机   |      IP       |              组件              |    备注     |
| :------: | :-----------: | :----------------------------: | :---------: |
| dbtest00 | 172.20.11.200 | PostgreSQL、Citus、TimescaleDB | Coordinator |
| dbtest01 | 172.20.11.201 | PostgreSQL、Citus、TimescaleDB |   备节点    |
| dbtest02 | 172.20.11.202 | PostgreSQL、Citus、TimescaleDB |   Worker    |



### 引入插件

```sql
-- 在每个节点执行
create extension citus;

```



### 构建集群

```SQL
-- 在Coordinator节点执行,添加worker节点
postgres=# SELECT * from master_add_node('172.20.11.201',5432);
postgres=# SELECT * from master_add_node('172.20.11.202',5432);

postgres=# SELECT * FROM master_get_active_worker_nodes();  
   node_name   | node_port 
---------------+-----------
 172.20.11.201 |      5432
 172.20.11.202 |      5432
(2 rows)

```



### 创建分片表

```SQL
-- 在Coordinator节点执行

-- 创建表
create table tbserial(id serial,c1 int);
-- 指定为分片表
select create_distributed_table('tbserial','id');

-- 利用pgbench 模拟测试，创建测试脚本  bench_insert.sql
vim bench_insert.sql

\sleep 500 ms
\set ival random(1, 100000)
insert into tbserial(c1) values ( :ival);

--执行，模拟4个客户端，两个线程，运行30秒
pgbench -c 4 -j 2 -M prepared -n -s 500 -T 30   -f bench_insert.sql -h 127.0.0.1 -U postgres postgres

-- 通过执行计划查看ID= 539743 的数据落在了  202 这台节点
postgres=# explain analyse select * from tbserial where id = 539743;
                                                           QUERY PLAN                                                           
--------------------------------------------------------------------------------------------------------------------------------
 Custom Scan (Citus Adaptive)  (cost=0.00..0.00 rows=0 width=0) (actual time=5.517..5.518 rows=0 loops=1)
   Task Count: 1
   Tasks Shown: All
   ->  Task
         Node: host=172.20.11.202 port=5432 dbname=postgres
         ->  Seq Scan on tbserial_102033 tbserial  (cost=0.00..126.86 rows=1 width=8) (actual time=0.314..0.314 rows=0 loops=1)
               Filter: (id = 539743)
               Rows Removed by Filter: 893
             Planning Time: 0.071 ms
             Execution Time: 0.333 ms
 Planning Time: 0.615 ms
 Execution Time: 5.551 ms
(12 rows)

```



## 时序数据库TimescaleDB实践

### 部署规划

沿用上面citus的集群

### 引入插件

```sql
-- 在所有节点执行
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;
```

### 创建hypertable 表

```sql
CREATE TABLE tb_device_monit (
  device_id int,
  capturetime        TIMESTAMPT       NOT NULL,
  metric_name    TEXT              NOT NULL,
  metric_value DOUBLE PRECISION  NULL
);

-- 创建hypertable 表
SELECT create_hypertable('tb_device_monit', 'capturetime',);


```



## Citus混合TimescaleDB

```sql
-- http://docs.citusdata.com/en/v9.3/use_cases/timeseries.html
-- 注意，官方推荐使用pg自带的分区功能结合pg_partman来做自动分区功能
CREATE TABLE conditions (
  time        TIMESTAMPTZ       NOT NULL,
  location    TEXT              NOT NULL,
  temperature DOUBLE PRECISION  NULL,
  humidity    DOUBLE PRECISION  NULL
);

-- 创建hypertable 表
SELECT create_hypertable('conditions', 'time');

-- 创建citus分片表
select create_distributed_table('conditions','location');

-- 插入数据
INSERT INTO conditions(time, location, temperature, humidity)
  VALUES (NOW(), 'office', 70.0, 50.0);

SELECT * FROM conditions ORDER BY time DESC LIMIT 100;

--
```



## 备份恢复

### 定期全量备份

#### 前期准备

```shell
# 备份目录,需要单独挂载盘，最好是网络存储，可以通过nfs 或者samba挂载
mkdir -p /data/pgsql/backups
# 挂载举例
mount /sdb1 /data/pgsql/backups
chown -R postgres:postgres  /data/pgsql/backups

```

#### 备份脚本

在本机执行。

backup_full.sh

```shell
#!/bin/sh
# 创建备份脚本，注意修改host、port
backup_dir=/data/pgsql/backups
yesterday_dir=${backup_dir}/$(date -d "1 day ago" +"%Y%m%d")
today_dir=${backup_dir}/$(date +%Y%m%d)
mkdir -p $today_dir
host=172.20.11.200
port=5432
pg_basebackup -h $host -p $port -U repl -W -Ft -Xf -Pv -z -Z5 -D $today_dir
# 每次成功备份完之后，删除前一天的全量备份。保证只保留一份全量备份。也可以酌情修改保存天数。
if [ "$?" == "0" ]; then
  rm -rf $yesterday_dir
fi
```

执行完成后，在/data/pgsql/backups 目录下会有 20200701/base.tar.gz的子目录。

### 增量备份

开启归档模式，即开始增量备份。默认保存7天的归档日志。postgresql.conf中的相关配置如下：

```shell

archive_mode = on
archive_command = 'test ! -f /data/pgsql/archive_wals/%f && lz4 -q -z %p /data/pgsql/archive_wals/%f.lz4 ; find /data/pgsql/archive_wals/ -type f -mtime +7 -exec rm -f {} \;'      #  后期再修改,默认保存7天的归档日志
```



前期安装已经默认打开了归档模式



### 恢复

1. 安装pg软件，参考第二章节

2. 清空数据目录 

   ```
   rm - rm /data/pgsql/data/*
   ```

   

3. 拷贝最近的全量备份到数据目录

   ```shell
   scp -rf X.X.X.X:/AA/base.tar.gz /data/pgsql/
   tar -xvf /data/pgsql/base.tar.gz -c /data/pgsql/data
   rm -rf /data/pgsql/base.tar.gz
   ```

   

4. 拷贝所有增量备份文件到本机/data/pgsql/archive_wals目录

   ```shell
   scp -rf X.X.X.X:/AA/* /data/pgsql/archive_wals/
   ```

   

5. 编辑postgresql.conf文件，增加restore_command命令

   ```shell
   restore_command = 'lz4 -f -q -d /data/pgsql/archive_wals/%f.lz4 %p'
   recovery_target_timeline = 'latest'
   ```

   

6. 启动新数据库

   ```shell
   su - postgres
   pg_ctl start
   ```

   



## 定时任务配置	

### cron说明

***crontab时间说明***



```bash
# .---------------- minute (0 - 59) 
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ... 
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7)  OR
#sun,mon,tue,wed,thu,fri,sat 
# |  |  |  |  |
# *  *  *  *  *  command to be executed


 minute：代表一小时内的第几分，范围 0-59。
 hour：代表一天中的第几小时，范围 0-23。
 mday：代表一个月中的第几天，范围 1-31。
 month：代表一年中第几个月，范围 1-12。
 wday：代表星期几，范围 0-7 (0及7都是星期天)。
 who：要使用什么身份执行该指令，当您使用 crontab -e 时，不必加此字段。
 command：所要执行的指令。
```



------

***crontab服务状态***



```bash
service crond start     #启动服务
service crond stop      #关闭服务
service crond restart   #重启服务
service crond reload    #重新载入配置
service crond status    #查看服务状态
```

------

***crontab命令***
 重新指定crontab定时任务列表文件

```shell
crontab $filepath
```

***查看crontab定时任务***

```undefined
crontab -l
```

***编辑定时任务【删除-添加-修改】***

```undefined
crontab -e
```



***crontab时间举例***

```bash
# 每天早上6点 
0 6 * * * echo "Good morning." >> /tmp/test.txt //注意单纯echo，从屏幕上看不到任何输出，因为cron把任何输出都email到root的信箱了。

# 每两个小时 
0 */2 * * * echo "Have a break now." >> /tmp/test.txt  

# 晚上11点到早上8点之间每两个小时和早上八点 
0 23-7/2,8 * * * echo "Have a good dream" >> /tmp/test.txt

# 每个月的4号和每个礼拜的礼拜一到礼拜三的早上11点 
0 11 4 * 1-3 command line

# 1月1日早上4点 
0 4 1 1 * command line SHELL=/bin/bash PATH=/sbin:/bin:/usr/sbin:/usr/bin MAILTO=root //如果出现错误，或者有数据输出，数据作为邮件发给这个帐号 HOME=/ 

# 每小时（第一分钟）执行/etc/cron.hourly内的脚本
01 * * * * root run-parts /etc/cron.hourly

# 每天（凌晨4：02）执行/etc/cron.daily内的脚本
02 4 * * * root run-parts /etc/cron.daily 

# 每星期（周日凌晨4：22）执行/etc/cron.weekly内的脚本
22 4 * * 0 root run-parts /etc/cron.weekly 

# 每月（1号凌晨4：42）去执行/etc/cron.monthly内的脚本 
42 4 1 * * root run-parts /etc/cron.monthly 

# 注意:  "run-parts"这个参数了，如果去掉这个参数的话，后面就可以写要运行的某个脚本名，而不是文件夹名。 　 

# 每天的下午4点、5点、6点的5 min、15 min、25 min、35 min、45 min、55 min时执行命令。 
5，15，25，35，45，55 16，17，18 * * * command

# 每周一，三，五的下午3：00系统进入维护状态，重新启动系统。
00 15 * *1，3，5 shutdown -r +5

# 每小时的10分，40分执行用户目录下的innd/bbslin这个指令： 
10，40 * * * * innd/bbslink 

# 每小时的1分执行用户目录下的bin/account这个指令： 
1 * * * * bin/account

# 每天早晨三点二十分执行用户目录下如下所示的两个指令（每个指令以;分隔）： 
203 * * * （/bin/rm -f expire.ls logins.bad;bin/expire$#@62;expire.1st）　
```



### 配置举例

```shell
# 配置文件
vim /etc/cron.d/cron_postgres
# 输入以下内容
# 每周日凌晨1点01分开始全量备份
01 1 * * 0 postgres cd  /data/backups && sh /data/backups/backup_full.sh 

# 加载配置文件
crontab /etc/cron.d/cron_postgres

#  列出当前定时任务
crontab -l

# 查看定时任务执行状态
service crond status 
```



## 常用命令

### 创建用户

```sql
-- 创建用户
```

