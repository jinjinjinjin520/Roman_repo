## Welcome to GitHub Pages

You can use the [editor on GitHub](https://github.com/jinjinjinjin520/Roman_repo/edit/main/README.md) to maintain and preview the content for your website in Markdown files.

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

### Markdown

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/jinjinjinjin520/Roman_repo/settings/pages). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://support.github.com/contact) and we’ll help you sort it out.


# 如何搭建PostGress
## 一、环境
### 1. 安装常用包
```markdown
yum -y install coreutils glib2 lrzsz mpstat dstat sysstat e4fsprogs xfsprogs ntp readline-devel zlib-devel openssl-devel pam-devel libxml2-devel libxslt-devel python-devel tcl-devel gcc make smartmontools flex bison perl-devel perl-ExtUtils* openldap-devel jadetex  openjade bzip2
```
### 2.关闭SELINUX和防火墙
```
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config # 重启生效
systemctl stop firewalld.service  
systemctl disable firewalld.service
```

### 3.修改主机名
```
hostnamectl set-hostname sgpg

cat>>/etc/hosts<<EOF
10.1.21.149 sgpg
EOF
```
### 4.修改OS内核参数
```
vim /etc/sysctl.conf

fs.aio-max-nr = 1048576
fs.file-max = 76724600
kernel.sem = 4096 2147483647 2147483646 512000    
kernel.shmall = 3355443  #单位page,内存80%，参数= 内存*0.8*1024*1024*1024/PAGE_SIZE
kernel.shmmax = 8589934592 #单位字节，内存50%，参数=内存*0.5*1024*1024*1024
kernel.shmmni = 819200
net.core.netdev_max_backlog = 10000
net.core.rmem_default = 262144       
net.core.rmem_max = 4194304          
net.core.wmem_default = 262144       
net.core.wmem_max = 4194304          
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_keepalive_intvl = 20
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_mem = 8388608 12582912 16777216
net.ipv4.tcp_fin_timeout = 5
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syncookies = 1    
net.ipv4.tcp_timestamps = 1    
net.ipv4.tcp_tw_recycle = 0    
net.ipv4.tcp_tw_reuse = 1      
net.ipv4.tcp_max_tw_buckets = 262144
net.ipv4.tcp_rmem = 8192 87380 16777216
net.ipv4.tcp_wmem = 8192 65536 16777216
net.nf_conntrack_max = 1200000
net.netfilter.nf_conntrack_max = 1200000
vm.dirty_background_bytes = 409600000       
vm.dirty_expire_centisecs = 3000             
vm.dirty_ratio = 95                          
vm.dirty_writeback_centisecs = 100            
vm.mmap_min_addr = 65536
vm.overcommit_memory = 0     
vm.overcommit_ratio = 90     
vm.swappiness = 0            
vm.zone_reclaim_mode = 0     
fs.nr_open=20480000
net.ipv4.tcp_max_syn_backlog = 16384
net.core.somaxconn = 16384
```
* 立即生效
```
sysctl -p
```
* 内核参数说明
```
fs.aio-max-nr = 1048576
fs.file-max = 76724600
#kernel.core_pattern= /data01/corefiles/core_%e_%u_%t_%s.%p         
# /data01/corefiles事先建好，权限777，如果是软链接，对应的目录修改为777
kernel.sem = 4096 2147483647 2147483646 512000    
# 信号量, ipcs -l 或 -u 查看，每16个进程一组，每组信号量需要17个信号量。
kernel.shmall = 3355443
# 所有共享内存段相加大小限制(建议内存的80%)，单位PAGE，getconf PAGE_SIZE   参数= 内存*0.8*1024*1024*1024/PAGE_SIZE
kernel.shmmax = 8589934592
# 最大单个共享内存段大小(建议为内存一半), >9.2的版本已大幅降低共享内存的使用，单位字节
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

### 4.时区
```
# 查看时区
timedatectl status 
 Local time: Sun 2020-08-30 20:16:16 CST
  Universal time: Sun 2020-08-30 12:16:16 UTC
        RTC time: Sun 2020-08-30 12:16:16
       Time zone: Asia/Shanghai (CST, +0800)
     NTP enabled: no
NTP synchronized: no
 RTC in local TZ: no
      DST active: n/a
# 修改时区为上海
timedatectl set-timezone Asia/Shanghai
```
### 5.用户
```
#userdel -r postgres # 若已存在则删除
groupadd dba -g 2000
useradd postgres -g 2000 -u 2000
echo "postgres"|passwd --stdin postgres # 设置密码。也可以用passwd username去设置
```
### 6.目录
```
mkdir /opt/soft
mkdir -p /opt/pg/pg13/{pgsql,pgdata,pgwal,pgarchive,pglog,pgtbs}
chown -R postgres:dba /opt/soft /opt/pg
chmod 0700 /opt/soft /opt/pg
```
* 目录规划如下
<div class="wiz-table-container" style="position: relative; padding: 0px;" contenteditable="false"><div style="" class="wiz-table-body" contenteditable="false"><table style="width: 554px;"><tbody><tr><td align="left" valign="middle" style="width: 220px;">目录</td><td align="left" valign="middle" style="width: 213px;">说明</td></tr><tr><td align="left" valign="middle" style="width: 220px;">/opt/</td><td align="left" valign="middle" style="width: 213px;">第三方软件目录</td></tr><tr><td align="left" valign="middle" style="width: 220px;">/opt/pg/</td><td align="left" valign="middle" style="width: 213px;">PG根目录</td></tr><tr><td align="left" valign="middle" style="width: 220px;"><span style="">/opt/pg/pg13</span></td><td align="left" valign="middle" style="width: 213px;">软件版本</td></tr><tr><td align="left" valign="middle" style="width: 220px;"><span style="">/opt/pg/pg13/pgarchive</span></td><td align="left" valign="middle" style="width: 213px;">归档目录</td></tr><tr><td align="left" valign="middle" style="width: 220px;"><span style="">/opt/pg/pg13/data</span></td><td align="left" valign="middle" style="width: 213px;">数据目录</td></tr><tr><td align="left" valign="middle" style="width: 220px;"><span style="">/opt/pg/pg13/pgsql</span></td><td align="left" valign="middle" style="width: 213px;">软件安装目录</td></tr><tr><td style="width: 220px;"><span style="">/opt/pg/pg13/pgtbs</span></td><td style="width: 213px;">表空间目录</td></tr><tr><td style="width: 220px;"><span style="">/opt/pg/pg13/pgwal</span></td><td style="width: 213px;">WAL日志目录</td></tr><tr><td style="width: 220px;"><span style="">/opt/soft/</span></td><td style="width: 213px;">安装介质存放目录</td></tr></tbody></table></div></div>

### 7.设置用户limits
```
# vi /etc/security/limits.conf

# nofile超过1048576的话，一定要先将sysctl的fs.nr_open设置为更大的值，并生效后才能继续设置nofile.
# 如果想限制到具体用户可以把*改成postgres 
* soft    nofile  1024000
* hard    nofile  1024000
* soft    nproc   unlimited
* hard    nproc   unlimited
* soft    core    unlimited
* hard    core    unlimited
* soft    memlock unlimited
* hard    memlock unlimited
```
### 8.磁盘优化
#### 8.1 XFS逻辑卷优化部分
```
#创建PV前，将块设备对齐，前面1MB最好不要分配，从2048 sector开始分配。或者使用parted创建并对齐分区。
fdisk -c -u /dev/dfa
start  2048  
end + (2048*n) - 1  
```
#### 8.2 XFS挂载优化
```
allocsize 设置缓冲I / O端的文件预分配的大小时，延迟分配写出时。此选项的页面大小（通常4KiB）到1GiB，包括有效值，功率为2的增量。
nobarrier  写入屏障 :XFS文件系统默认在挂载时启用“写入屏障”的支持。该特性会一个合适的时间冲刷底层存储设备的回写缓存，特别是在XFS做日志写入操作的时候。这个特性的初衷是保证文件系统的一致性，具体实现却因设备而异——不是所有的底层硬件都支持缓存冲刷请求。在带有电池供电缓存的硬件RAID控制器提供的逻辑设备上部署XFS文件系统时，这项特性可能导致明显的性能退化，因为文件系统的代码无法得知这种缓存是非易失性的。如果该控制器又实现了冲刷请求，数据将被不必要地频繁写入物理磁盘。为了防止这种问题，对于能够在断电或发生其它主机故障时保护缓存中数据的设备，应该以nobarrier选项挂载XFS文件系统。
largeio 针对数据仓库，流媒体这种大量连续读的应用  
nolargeio 针对OLTP  
logbsize=262144   指定 log buffer  
logdev=  指定log section对应的块设备，用最快的SSD。  
noatime,nodiratime  
swalloc  条带对齐  
#mount -t xfs -o allocsize=16M,inode64,nolargeio,logbsize=262144,swalloc   /dev/mapper/vgdata01-lv02  /data01  
```
#### 8.3 磁盘的调度方式优化
```
临时设置(比如sda盘)
echo deadline > /sys/block/sda/queue/scheduler如果不是SSD的话，还是使用CFQ，否则建议使用DEADLINE。
永久设置
编辑grub文件修改块设备调度策略
vi /boot/grub.conf
elevator=deadline
注意，如果既有机械盘，又有SSD，那么可以使用/etc/rc.local，对指定磁盘修改为对应的调度策略。
```
#### 8.4 关闭透明大页、numa
```

yum install numactl -y
grep -i numa /var/log/dmesg
#如果输出结果为： No NUMA configuration found 或者NUMA turned off 说明numa为disable，如果不是上面内容说明numa为enable,会有节点信息显示；


#RHEL 4, RHEL 5, RHEL 6 (/boot/grub/grub.conf)
    title Red Hat Enterprise Linux AS (2.6.9-55.EL)
            root (hd0,0)
            kernel /vmlinuz-2.6.9-55.EL ro root=/dev/VolGroup00/LogVol00 rhgb quiet  transparent_hugepage=never numa=off 
            initrd /initrd-2.6.9-55.EL.img


#RHEL 7 (/etc/default/grub)
    GRUB_CMDLINE_LINUX="rd.lvm.lv=rhel_vm-210/root rd.lvm.lv=rhel_vm-210/swap vconsole.font=latarcyrheb-sun16 crashkernel=auto  vconsole.keymap=us rhgb quiet  transparent_hugepage=never  numa=off"
#and then rebuild the `/boot/grub2/grub.cfg` file to reflect the above changes:
On BIOS-based machines: ~]# grub2-mkconfig -o /boot/grub2/grub.cfg
On UEFI-based machines: ~]# grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
#重启生效
```

## 二、安装
### 1.安装包
```
[root@sgpg soft]# chown postgres:dba /opt/soft/postgresql-13.2.tar.bz2
[root@sgpg soft]# tar xjvf postgresql-13.0.tar.bz2
```
### 2.编译参数说明
```
-- 编译安装三板斧
1 ./configure
2 make
3 make install
-- make world 包括第三方插件全部编译
-- make install-world 包括第三方插件全部安装
```

### 3.编译安装
```
[root@sgpg soft]# cd /opt/soft/postgresql-13.2/
[root@sgpg postgresql-13.2]# ./configure --prefix=/opt/pg/pg13/pgsql --with-pgport=35432 --with-perl --with-python --with-tcl --with-openssl --with-pam --with-libxml --with-libxslt --without-ldap --with-wal-blocksize=16 --with-blocksize=8
#config.status: linking src/makefiles/Makefile.linux to src/Makefile.port     成功
[root@sgpg postgresql-13.2]# make world
#执行make 部分环境可能需要执行gmake
#PostgreSQL, contrib, and documentation successfully made. Ready to install. 成功
[root@sgpg postgresql-13.2]#  make install
#PostgreSQL installation complete.  成功
[root@sgpg postgresql-13.2]# /opt/pg/pg13/pgsql/bin/postgres --version
postgres (PostgreSQL) 13.2
```
### 4.设置软连接
```
#ln -s /opt/pg/pg13/pgsql /opt/pgsql  #方便升级、按需设置即可。

```

### 5.初始化

```
注意：加-W参数会提示输入数据库的超级用户的密码，默认情况下密码为空。
-D：data目录的路径
-U：数据库超级用户名
-E：配置区域语言、字符集
-X：WAL日志路径
[root@sgpg postgresql-13.2]# su - postgres
/opt/pg/pg13/pgsql/bin/initdb -D /opt/pg/pg13/pgdata -X /opt/pg/pg13/pgwal -E UTF8 -U postgres -W
[postgres@sgpg ~]$ /opt/pg/pg13/pgsql/bin/initdb -D /opt/pg/pg13/pgdata -X /opt/pg/pg13/pgwal -E UTF8 -U postgres -W
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

Enter new superuser password:
Enter it again:

fixing permissions on existing directory /opt/pg/pg13/pgdata ... ok
fixing permissions on existing directory /opt/pg/pg13/pgwal ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Asia/Shanghai
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

initdb: warning: enabling "trust" authentication for local connections
You can change this by editing pg_hba.conf or using the option -A, or
--auth-local and --auth-host, the next time you run initdb.

Success. You can now start the database server using:

    /opt/pg/pg13/pgsql/bin/pg_ctl -D /opt/pg/pg13/pgdata -l logfile start
```
### 6.修改数据库参数
* 修改`/opt/pg/pg13/pgdata/postgresql.conf `的参数建议如下：



### 7.配置环境变量
```
[root@sgpg ~]# su - postgres
[postgres@sgpg ~]$ vim .bash_profile
export PGPORT=5432
export PGUSER=postgres
export PGHOME=/opt/pg/pg13/pgsql
export PGDATA=/opt/pg/pg13/pgdata
export LD_LIBRARY_PATH=$PGHOME/lib
export PATH=$PGHOME/bin:$PATH
export LANG=en_US.utf8

```
### 8.安装contrib目录下的工具
```
[root@sgpg ~]# cd /opt/soft/postgresql-13.2/contrib/
[root@sgpg contrib]# make
[root@sgpg contrib]# make install
```
### 8.启停数据库
```
查看数据库运行状态

pg_ctl status
pg_ctl: no server running

启动数据库

pg_ctl start &

查看数据库运行状态
pg_ctl status
pg_ctl: server is running (PID: 27522)
/opt/pg/pg13/pgsql/bin/postgres

停止数据库
pg_ctl stop
waiting for server to shut down.... done
server stopped
因已经配置了环境变量中的PGDATA，所以无需使用pg_ctl -D $PGDATA start 等命令。直接使用pg_ctl start即可。
可以通过“-m” 参数定义关闭方式
smart：缺省模式，等待所有客户端断开 
fast：    强制模式，对未断开的客户端进行回滚 类似Oracle immediate
immediate：强制模式，但不回滚，重启时需要自动恢复 类似Oracle abort
```
### 9.设置开机自动启动
```
[root@sgpg ~]# cp /opt/soft/postgresql-13.2/contrib/start-scripts/linux /etc/init.d/postgresql
[root@sgpg ~]# chmod +x /etc/init.d/postgresql
[root@sgpg ~]# vim /etc/init.d/postgresql
#修改 prefix、PGDATA
prefix=/opt/pg/pg13/pgsql
PGDATA="/opt/pg/pg13/pgdata"
[root@sgpg ~]# service postgresql start
Starting PostgreSQL: ok
[root@sgpg ~]# chkconfig --add postgresql
```

## 三、修改相关数据库参数
### 1.修改日志模式

```
#开启日志收集
logging_collector = on 
#日志目录
log_directory = '/opt/pg/pg13/pglog'
```
* 以下方案可以任选一个，PG10以上默认是方案3不需要修改
#### 1.1 每天生成一个新的日志文件
```
log_filename = 'postgresql_log-%Y-%m-%d_%H%M%S.log'
log_truncate_on_rotation = off
log_rotation_age = 1d
log_rotation_size = 0
```
#### 1.2 每当日志写满一定大小如（100M），则切换日志
```
log_filename = 'postgresql_log-%Y-%m-%d_%H%M%S.log'
log_truncate_on_rotation = off
log_rotation_age = 0
log_rotation_size = 100M
```
#### 1.3 只保留七天，循环覆盖
```
log_filename = 'postgresql_log-%a.log'
log_truncate_on_rotation = on
log_rotation_age = 1d
log_rotation_size = 0
```
### 2.查看错误日志
* pgerror.log为初始启动时参数生成的日志文件
```
[postgres@pg13 pglog]$ ls
pgerror.log  postgresql_log.Mon  postgresql_log.Mon.csv

```


### 3.修改日志模式

```
log_filename='postgresql-%Y-%m-%d_%H%M%S.log'

```
### 4.配置数据库监听IP和端口
```
listen_addresses = '*'
port = 5555
max_connections = 1000
```

### 5. pg_hba.conf配置
```
#在pg_hba.conf文件中加入
host            all                      all          0/0          md5
```

