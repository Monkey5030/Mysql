# Greenplum安装
# 1.修改主机名称
![1609491-20190805161844129-1259150148.png](/tdl/tfl/pictures/202012/tapd_47536328_1606960094_4.png)
#2. 安装准备
## 2.1修改各节点hosts(所有节点) 
```
[root@wuxiang-test-2 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.40.218 wuxiang-test-1
192.168.40.238 wuxiang-test-2
192.168.40.239 wuxiang-test-3
192.168.40.240 wuxiang-test-4
192.168.40.241 wuxiang-test-5
```
注：标注了所有节点的配置项可以在安装greenplum并配置免密后用gpssh统一操作3.3。
## 2.2 修改network文件（所有节点，名称会有差异）
```
[root@wuxiang-test-2 ~]# cat /etc/sysconfig/network
NISDOMAIN=QI
HOSTNAME=wuxiang-test-2
```
## 2.3修改内核文件（所有节点）
```
[root@wuxiang-test-2 ~]# cat /etc/sysctl.conf
vm.swappiness = 10
kernel.shmmax = 500000000
kernel.shmmni = 4096
kernel.shmall = 4000000000
kernel.sem = 250 512000 100 2048
kernel.sysrq = 1
kernel.core_uses_pid = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.msgmni = 2048
net.ipv4.tcp_syncookies = 1
net.ipv4.ip_forward = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.conf.all.arp_filter = 1
net.ipv4.ip_local_port_range = 1025 65535
net.core.netdev_max_backlog = 10000
```
最后是配置生效：
```
[root@wuxiang-test-2 ~]# sysctl -p
```
重新启动虚拟机:
```
reboot
```
## 2.4 修改进程数文件（所有节点）
```
[root@wuxiang-test-2 ~]# cat /etc/security/limits.d/20-nproc.conf 
# Default limit for number of user's processes to prevent
# accidental fork bombs.
# See rhbz #432903 for reasoning.

*          soft    nproc     4096
root       soft    nproc     unlimited
```
## 2.5 关闭防火墙（所有节点）
查看防火墙状态：firewall-cmd --state
关闭防火墙：systemctl stop firewalld.service
禁止防火墙开机启动：systemctl disable firewalld.service
修改配置（所有节点）：
```
[root@wuxiang-test-2 ~]# cat /etc/selinux/config
SELINUX=disabled
SELINUXTYPE=targeted 
```
## 2.6 创建用户（各节点共享）
```
groupadd -g 530 gpadmin
useradd -g 530 -u 530 -m -d /home/gpadmin -s /bin/bash gpadmin
chown -R gpadmin:gpadmin /home/gpadmin
echo "gpadmin" | passwd --stdin gpadmin
```
# 3 安装Greenplum DB
## 3.1 在Master节点上安装Greeplum
安装包下载地址：https://network.pivotal.io/products/pivotal-gpdb/#/releases/413133/file_groups/1866 
安装包是rpm格式的执行rpm安装命令
```
[root@wuxiang-test-1 ~]#  rpm -ivh greenplum-db-6.12.1-rhel7-x86_64.rpm
```
安装前需要下载一些依赖包:
```
yum install apr apr-util bzip2 krb5-devel libyaml lrsync rsync zip net-tools libevent
```
默认的安装路径是/usr/local。
将/usr/local/greenplum-db-5.21.0文件拷贝至所有节点（可以压缩再解压，也可以使用gpssh方式）
然后需要修改该路径gpadmin操作权限（所有节点）：

``` 
chown -R gpadmin:gpadmin /usr/local
chown -R gpadmin:gpadmin /opt
```
## 3.2 创建hostlist，seg_hosts文件
切换gpadmin用户，创建conf文件夹（需要创建conf）
```
[gpadmin@wuxiang-test-1 ~]# cd conf/ 
[gpadmin@wuxiang-test-1 conf]# cat hostlist 
wuxiang-test-1 
wuxiang-test-2 
wuxiang-test-3 
wuxiang-test-4 
wuxiang-test-5 
[gpadmin@wuxiang-test-1 conf]# cat seg_hosts 
wuxiang-test-2 
wuxiang-test-3 
wuxiang-test-4 
wuxiang-test-5
```
## 3.3 配置免密连接
```
[root@ wuxiang-test-1 ~]# su gpadmin
[gpadmin@ wuxiang-test-1 ~]# source /usr/local/greenplum-db/greenplum_path.sh  
[gpadmin@ wuxiang-test-1 ~]# gpssh-exkeys -f /home/gpadmin/conf/hostlist
 
[STEP 1 of 5] create local ID and authorize on local host
  ... /home/gpadmin/.ssh/id_rsa file exists ... key generation skipped
 
[STEP 2 of 5] keyscan all hosts and update known_hosts file
 
[STEP 3 of 5] authorize current user on remote hosts
  ... send to wuxiang-test-1
  ... send to wuxiang-test-2
  ... send to wuxiang-test-3
  ... send to wuxiang-test-4
  ... send to wuxiang-test-5
#提示：这里提示输入各个子节点gpadmin用户密码
[STEP 4 of 5] determine common authentication file content
 
[STEP 5 of 5] copy authentication files to all remote hosts
  ... finished key exchange with wuxiang-test-1
  ... finished key exchange with wuxiang-test-2
  ... finished key exchange with wuxiang-test-3
  ... finished key exchange with wuxiang-test-4
  ... finished key exchange with wuxiang-test-5
[INFO] completed successfully
```
测试免密是否成功：
```
[gpadmin@wuxiang-test-1 ~]# ssh wuxiang-test-4
```
或者用gpssh：
```
[gpadmin@wuxiang-test-1 ~]$ gpssh -f /home/gpadmin/conf/hostlist
=> pwd
[wuxiang-test-1] /home/gpadmin
[wuxiang-test-4] /home/gpadmin
[wuxiang-test-5] /home/gpadmin
[wuxiang-test-3] /home/gpadmin
[wuxiang-test-2] /home/gpadmin
=> exit
```
# 4 初始化数据库
## 4.1 创建资源目录
```
source /usr/local/ greenplum-db/greenplum_path.sh
gpssh -f /home/gpadmin/conf/hostlist #统一处理所有节点
 
#创建资源目录 /opt/greenplum/data下一系列目录（生产目录个数可根据需求生成）
=> mkdir -p /opt/greenplum/data/master
=> mkdir -p /opt/greenplum/data/primary
=> mkdir -p /opt/greenplum/data/mirror
=> mkdir -p /opt/greenplum/data2/primary
=> mkdir -p /opt/greenplum/data2/mirror
```
## 4.2 环境变量配置（所有节点）
```
[gpadmin@wuxiang-test-1 ~]$ cat /home/gpadmin/.bash_profile
# .bash_profile

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi

# User specific environment and startup programs

PATH=$PATH:$HOME/.local/bin:$HOME/bin


export GPHOME=/usr/local/greenplum-db-6.12.1
export PATH
source /usr/local/greenplum-db/greenplum_path.sh
export MASTER_DATA_DIRECTORY=/opt/greenplum/data/master/gpseg-1
export GPPORT=5432
export PGDATABASE=gp_sydb
```
注：不能用gpssh编辑文件
让环境变量生效：
```
source /home/gpadmin/.bash_profile
```
## 4.3 NTP配置 （单节点不需要配置）
启用master节点上的ntp，并在Segment节点上配置和启动NTP：
```
#master 节点
[root@wuxiang-test-1 ~]#&nbsp;echo "server 127.127.1.0" >>/etc/ntp.conf 
#Segment节点
[root@wuxiang-test-2 ~]#&nbsp;echo "server wuxiang-test-1 perfer" >>/etc/ntp.conf
#master节点
[root@wuxiang-test-1 ~]#&nbsp;systemctl start  ntpd
[root@wuxiang-test-1 ~]#&nbsp;systemctl enable  ntpd
```
## 4.4 检查各节点的连通性
`[gpadmin@wuxiang-test-1 bin]$ cd /usr/local/greenplum-db/bin` 
```
[gpadmin@wuxiang-test-1 bin]$ gpcheckperf -f /home/gpadmin/conf/hostlist -r N -d /tmp
/usr/local/greenplum-db/./bin/gpcheckperf -f /home/gpadmin/conf/hostlist -r N -d /tmp

-------------------
--  NETPERF TEST
-------------------
[Warning] retrying with port 23012
[Warning] retrying with port 23024
[Warning] retrying with port 23036
[Warning] retrying with port 23048
[Warning] retrying with port 23060

====================
==  RESULT
====================
Netperf bisection bandwidth test
wuxiang-test-1 -> wuxiang-test-2 = 110.490000
wuxiang-test-3 -> wuxiang-test-4 = 112.120000
wuxiang-test-5 -> wuxiang-test-1 = 108.990000
wuxiang-test-2 -> wuxiang-test-1 = 102.830000
wuxiang-test-4 -> wuxiang-test-3 = 112.010000
wuxiang-test-1 -> wuxiang-test-5 = 108.930000

Summary:
sum = 655.37 MB/sec
min = 102.83 MB/sec
max = 112.12 MB/sec
avg = 109.23 MB/sec
median = 110.49 MB/sec
```
## 4.5 执行初始化
```
[gpadmin@wuxiang-test-1 bin]$ cd /usr/local/greenplum-db/docs/cli_help/gpconfigs
[gpadmin@wuxiang-test-1 gpconfigs]$ cp gpinitsystem_config initgp_config
[gpadmin@wuxiang-test-1 gpconfigs]$ vim initgp_config
```
修改内容：
```
# FILE NAME: gpinitsystem_config

# Configuration file needed by the gpinitsystem

################################################
#### REQUIRED PARAMETERS
################################################

#### Name of this Greenplum system enclosed in quotes.
ARRAY_NAME="gp_sydb"        #初始化数据库名称

#### Naming convention for utility-generated data directories.
SEG_PREFIX=gpseg

#### Base number by which primary segment port numbers
#### are calculated.
PORT_BASE=6000

#### File system location(s) where primary segment data directories
#### will be created. The number of locations in the list dictate
#### the number of primary segments that will get created per
#### physical host (if multiple addresses for a host are listed in
#### the hostfile, the number of segments will be spread evenly across
#### the specified interface addresses).
declare -a DATA_DIRECTORY=(/opt/greenplum/data/primary /opt/greenplum/data2/primary)

#### OS-configured hostname or IP address of the master host.
MASTER_HOSTNAME=gp01                    #主节点名称

#### File system location where the master data directory
#### will be created.
MASTER_DIRECTORY=/opt/greenplum/data/master/gpseg-1      # 这个是目录否则初始化失败
MASTER_DATA_DIRECTORY=/opt/greenplum/data/master/gpseg-1


DATABASE_NAME=gp_sydb                                                
MACHINE_LIST_FILE=/home/gpadmin/conf/seg_hosts
#### Port number for the master instance.
MASTER_PORT=5432

#### Shell utility used to connect to remote hosts.
TRUSTED_SHELL=ssh

#### Maximum log file segments between automatic WAL checkpoints.
CHECK_POINT_SEGMENTS=8

#### Default server-side character set encoding.
ENCODING=UNICODE

################################################
#### OPTIONAL MIRROR PARAMETERS
################################################

#### Base number by which mirror segment port numbers
#### are calculated.
#MIRROR_PORT_BASE=7000

#### File system location(s) where mirror segment data directories
#### will be created. The number of mirror locations must equal the
#### number of primary locations as specified in the
#### DATA_DIRECTORY parameter.
#declare -a MIRROR_DATA_DIRECTORY=(/data1/mirror /data1/mirror /data1/mirror /data2/mirror /data2/mirror /data2/mirror)


################################################
#### OTHER OPTIONAL PARAMETERS
################################################

#### Create a database of this name after initialization.
#DATABASE_NAME=name_of_database

#### Specify the location of the host address file here instead of
#### with the -h option of gpinitsystem.
#MACHINE_LIST_FILE=/home/gpadmin/gpconfigs/hostfile_gpinitsystem
```
执行初始化:
```
[gpadmin@wuxiang-test-1 bin]$ gpinitsystem -h /home/gpadmin/conf/seg_hosts -c initgp_config
```
在初始化前要执行：
```
cd /opt/greenplum/data/master/
mkdir gpseg-1
```
# 5 数据库操作
## 5.1 停止和启动集群
```
gpstop -M fast
gpstart -a
```
## 5.2 登陆数据库（可以初始化后直接登录数据库）
```
psql -d postgres
```
## 5.3 集群状态
```
gpstate -e #查看mirror的状态
gpstate -f #查看standby master的状态
gpstate -s #查看整个GP群集的状态
gpstate -i #查看GP的版本
gpstate --help #帮助文档，可以查看gpstate更多用法
```
目前为止数据库已经操作完毕。默认只有本地可以连数据库，如果需要别的I可以连，需要修改gp_hba.conf文件
# GPText安装
确保nc（netcat）已安装在所有Greenplum群集主机
```
yum install nc
```
lsof建议在所有群集主机上安装
```
sudo yum install lsof
```
## 1JDK安装
### 1.1 解压
```
sudo tar zxvf jdk-8u66-linux-x64.tar.gz
```
### 1.2 设置JDK的环境变量
```
vim /etc/profile
```
```
#JAVA
export JAVA_HOME=/usr/local/jdk1.8.0_271
export JRE_HOME=$JAVA_HOME/jre
export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
```
使环境变量生效:
```
source /etc/profile
```
### 1.3 检验安装是否成功
```
[root@gp01 local]# java -version

java version "1.8.0_271"
Java(TM) SE Runtime Environment (build 1.8.0_271-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.271-b09, mixed mode)
```
## 2 安装ZK

``` 
cd /usr/local
tar -zxvf zookeeper-3.4.13.tar.gz
cd zookeeper-3.4.13
mkdir data
mkdir logs
touch data/myid
```

``` 
vim data、myid      #分别在不同的主机上写入主机名
```
### 2.1 配置文件
``` 
mv conf/zoo_sample.cfg conf/zoo.cfg
vim conf/zoo.cfg
```
```
 dataDir=/usr/local/zookeeper-3.4.13/data
 dataLogDir=/usr/local/zookeeper-3.4.13/logs
 server.1=mdw:2888:3888
 server.2=sdw1:2888:3888
 server.3=sdw2:2888:3888
```
### 2.2 配置环境
```
vim /etc/profile
```
```
#ZOOKEEPER_HOME
export ZOOKEEPER_HOME=/usr/local/zookeeper-3.4.6
export PATH=$ZOOKEEPER_HOME/bin:$PATH
```
```
source /etc/profile
```
```
zkServer.sh start
```

## 3 安装GPText
下载gptext：https://network.pivotal.io/products/pivotal-gpdb/#/releases/253113/file_groups/1331
``` 
cd /home/gpadmin
tar -zxvf greenplum-text-3.1.0-rhel6_x86_64.tar.gz
ls
      >>gptext_install_config 
      >>greenplum-text-3.1.0-rhel6_x86_64.bin
```
链接其他主机
```
source $GPHOME/greenplum_path.sh
```
```
vim hostlist.txt                         //创建hostaname文件，用于链接其他主机
       mdw
       sdw1
       sdw2
```

### 在需要安装的机器上批量安装并创建目录
``` 
mkdir /usr/local/greenplum-text-3.1.0
mkdir /usr/local/greenplum-solr
chown gpadmin:gpadmin /usr/local/greenplum-text-3.1.0
chmod 775 /usr/local/greenplum-text-3.1.0
chown gpadmin:gpadmin /usr/local/greenplum-solr
chmod 775 /usr/local/greenplum-solr
mkdir -p /data/gptext
chown -R gpadmin:gpadmin /data/gptext
chmod 775 /data/gptext
chown gpadmin:gpadmin greenplum-text-3.1.0-rhel6_x86_64.bin
chown gpadmin:gpadmin gptext_install_config
```
### 进入gpadmin
``` 
su – gpadmin
```
### 修改配置文件gptext_install_config
```
 declare -a GPTEXT_HOSTS=(mdw swd1 sdw2)    
 			declare -a GPTEXT_HOSTS=(mdw swd1 sdw2)                             //声明集群的主机名
            declare -a DATA_DIRECTORY=(/data/gptext/primary /data/gptext/primary)  
//设置数据存储路径
			JAVA_OPTS="-Xms1024M -Xmx2048M"              //设置SolrCloud JVM的最大值和最小值
            GPTEXT_PORT_BASE=18983                            //设置端口的范围
             GP_MAX_PORT_LIMIT=28983
       ZOO_CLUSTER="mdw:2181,sdw1:2181,sdw2:2181"     //zookeeper
       ZOO_GPTXTNODE="/usr/local/zookeeper-3.4.6/data"  //填写zookeeper的路径到data路径
       ZOO_PORT_BASE=2188
       ZOO_MAX_PORT_LIMIT=12188
       GPTEXT_JAVA_HOME=/usr/local/jdk1.8.0_191 
```
### 运行安装文件
```
./greenplum-text-3.1.0-rhel6_x86_64.bin -c gptext_install_config
```
### 启动gptext
```
source /usr/local/greenplum-text-3.5.0/greenplum-text_path.sh
source /usr/local/greenplum-db/greenplum_path.sh
```
 在数据库安装gptext实例，gp_sydb是本地数据库
```
gptext-installsql gp_sydb
```
###  启动gptext
``` 
gpconfig -c custom_variable_classes -v 'gptext'
```
### 配置greenplum数据库
http://gptext.docs.pivotal.io/350/topics/installing.html                           ---根据官方文档修改
