---
title: ceph-V13.2.6-Mimic版本ceph-deploy安装笔记
categories: ceph
---

## ceph-V13.2.6-Mimic版本ceph-deploy安装笔记

**<center>目录</center>**

[toc]


## 第一步:基本环境

* 准备机器

    192.168.20.113    ceph-admin（ceph-deploy）   mds1、mon1（也可以将monit节点另放一台机器）
    192.168.20.110    ceph-node1                  osd1  
    192.168.20.111    ceph-node2                  osd2 
    192.168.20.112   ceph-node3                  osd3  

-------------------------------------------------

* 每个节点修改主机名

`# hostnamectl set-hostname ceph-admin`

`# hostnamectl set-hostname ceph-node1`

`# hostnamectl set-hostname ceph-node2`

`# hostnamectl set-hostname ceph-node3`

-------------------------------------------------

* 每个节点绑定主机名映射

```
# cat >>  /etc/hosts <<EOF
192.168.20.113    ceph-admin
192.168.20.110    ceph-node1
192.168.20.111   ceph-node2
192.168.20.112    ceph-node3
EOF
```
-------------------------------------------------

* 每个节点确认连通性
```
# ping -c 3 ceph-admin

# ping -c 3 ceph-node1

# ping -c 3 ceph-node2

# ping -c 3 ceph-node3

```
-------------------------------------------------

* 每个节点关闭防火墙和selinux
```
# systemctl stop firewalld

# systemctl disable firewalld

# sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# setenforce 0
```
-------------------------------------------------

* 每个节点安装和配置NTP（官方推荐的是集群的所有节点全部安装并配置 NTP，需要保证各节点的系统时间一致。没有自己部署ntp服务器，就在线同步NTP）
```
# yum install ntp ntpdate ntp-doc -y
# ntpdate 0.cn.pool.ntp.org
# systemctl restart ntpd
# systemctl status ntpd
```
-------------------------------------------------

* 每个节点准备yum源

> 删除默认的源，国外的比较慢
```
# yum clean all

# mkdir /mnt/bak

# mv /etc/yum.repos.d/* /mnt/bak/
```

* 下载阿里云的base源和epel源（可选）

```
# wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

# wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo

# yum clean all && mkdir /mnt/bak && mv /etc/yum.repos.d/* /mnt/bak/ && wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo && wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
```
* 添加ceph源
```
# cat > /etc/yum.repos.d/ceph.repo << EOF
[ceph]
name=ceph
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/x86_64/
gpgcheck=0
priority =1

[ceph-noarch]
name=cephnoarch
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/noarch/
gpgcheck=0
priority =1

[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/SRPMS
gpgcheck=0
priority=1
EOF
```
------------------------------------------------------------


* 每个节点创建cephuser用户，设置sudo权限
```
# useradd -d /home/cephuser -m cephuser

# echo "cephuser"|passwd --stdin cephuser

# echo "cephuser ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/cephuser

# chmod 0440 /etc/sudoers.d/cephuser

# sed -i s'/Defaults requiretty/#Defaults requiretty'/g /etc/sudoers

 ```

* 测试cephuser的sudo权限

```
# su - cephuser

$ sudo su -

```

------------------------------------------------------------

* 配置相互间的ssh信任关系

    现在ceph-admin节点上产生公私钥文件，然后将ceph-admin节点的.ssh目录拷贝给其他节点
```
[root@ceph-admin ~]# su - cephuser

[cephuser@ceph-admin ~]$ ssh-keygen -t rsa    #一路回车

[cephuser@ceph-admin ~]$ cd .ssh/

[cephuser@ceph-admin .ssh]$ ls

id_rsa  id_rsa.pub

[cephuser@ceph-admin .ssh]$ cp id_rsa.pub authorized_keys

 
[cephuser@ceph-admin .ssh]$ scp -r /home/cephuser/.ssh ceph-node1:/home/cephuser/

[cephuser@ceph-admin .ssh]$ scp -r /home/cephuser/.ssh ceph-node2:/home/cephuser/

[cephuser@ceph-admin .ssh]$ scp -r /home/cephuser/.ssh ceph-node3:/home/cephuser/
```
 

* 然后在各节点直接验证cephuser用户下的ssh相互信任关系

```
$ ssh -p22 cephuser@ceph-admin

$ ssh -p22 cephuser@ceph-node1

$ ssh -p22 cephuser@ceph-node2

$ ssh -p22 cephuser@ceph-node3
```

## 第二步:准备磁盘（ceph-node1、ceph-node2、ceph-node3三个节点）

    测试时使用的磁盘不要太小，否则后面添加磁盘时会报错，建议磁盘大小为20G及以上。

    如下分别在三个节点挂载了一块20G的裸盘

 

* 检查磁盘
```
# fdisk -l /dev/vda

Disk /dev/vdb: 21.5 GB, 21474836480 bytes, 41943040 sectors

Units = sectors of 1 * 512 = 512 bytes

Sector size (logical/physical): 512 bytes / 512 bytes

I/O size (minimum/optimal): 512 bytes / 512 bytes

```

* 格式化磁盘

```
# mkfs.xfs /dev/vda -f

 先umount  
# umount /dev/vda1

查看磁盘格式（xfs格式）

# blkid -o value -s TYPE /dev/vda
```

## 第三步:部署阶段（ceph-admin节点上使用ceph-deploy快速部署）
```
[root@ceph-admin ~]# su - cephuser
```   

* 安装ceph-deploy
```
[cephuser@ceph-admin ~]$ sudo yum update -y && sudo yum install ceph-deploy -y
```   

* 创建cluster目录

```
[cephuser@ceph-admin ~]$ mkdir cluster
[cephuser@ceph-admin ~]$ cd cluster/
```
      

* 创建集群（后面填写monit节点的主机名，这里monit节点和管理节点是同一台机器，即ceph-admin）

```
[cephuser@ceph-admin cluster]$ ceph-deploy new ceph-admin

.........

[ceph-admin][DEBUG ] IP addresses found: [u'192.168.10.220']

[ceph_deploy.new][DEBUG ] Resolving host ceph-admin

[ceph_deploy.new][DEBUG ] Monitor ceph-admin at 192.168.10.220

[ceph_deploy.new][DEBUG ] Monitor initial members are ['ceph-admin']

[ceph_deploy.new][DEBUG ] Monitor addrs are ['192.168.10.220']

[ceph_deploy.new][DEBUG ] Creating a random mon key...

[ceph_deploy.new][DEBUG ] Writing monitor keyring to ceph.mon.keyring...

[ceph_deploy.new][DEBUG ] Writing initial config to ceph.conf...

```

* 修改ceph.conf文件（注意：mon_host必须和public network 网络是同网段内！）

```
[cephuser@ceph-admin cluster]$ vim ceph.conf     #添加下面两行配置内容

......

public network = 192.168.10.220/24

osd pool default size = 3

```

* 安装ceph（过程有点长，需要等待一段时间....）

```
[cephuser@ceph-admin cluster]$ ceph-deploy install ceph-admin ceph-node1 ceph-node2 ceph-node3 --adjust-repos
```
      

* 初始化monit监控节点，并收集所有密钥
```
[cephuser@ceph-admin cluster]$ ceph-deploy mon create-initial

[cephuser@ceph-admin cluster]$ ceph-deploy gatherkeys ceph-admin
```
      

* 添加OSD到集群

    检查OSD节点上所有可用的磁盘

`[cephuser@ceph-admin cluster]$ ceph-deploy disk list ceph-node1 ceph-node2 ceph-node3`

      

* 使用zap选项删除所有osd节点上的分区
```
[cephuser@ceph-admin cluster]$ ceph-deploy disk zap ceph-node1 /dev/vda
[cephuser@ceph-admin cluster]$ ceph-deploy disk zap ceph-node2 /dev/vda
[cephuser@ceph-admin cluster]$ ceph-deploy disk zap ceph-node3 /dev/vda
```

* 添加mgr 管理节点

`[cephuser@ceph-admin cluster]$ ceph-deploy mgr create ceph-admin`

* 准备OSD（使用prepare命令）

```
[cephuser@ceph-admin cluster]$ ceph-deploy osd create --data /dev/vda ceph-node1
[cephuser@ceph-admin cluster]$ ceph-deploy osd create --data /dev/vda ceph-node2
[cephuser@ceph-admin cluster]$ ceph-deploy osd create --data /dev/vda ceph-node3
``` 

* 激活OSD（注意由于ceph对磁盘进行了分区，/dev/vdb磁盘分区为/dev/vdb1）
```
[cephuser@ceph-admin cluster]$ ceph-deploy osd activate ceph-node1:/dev/vdb1 ceph-node2:/dev/vdb1 ceph-node3:/dev/vdb1
```

---------------------------------------------------------------------------------------------

> 可能出现下面的报错：

```
[ceph-node1][WARNIN] ceph_disk.main.Error: Error: /dev/vdb1 is not a directory or block device

[ceph-node1][ERROR ] RuntimeError: command returned non-zero exit status: 1

[ceph_deploy][ERROR ] RuntimeError: Failed to execute command: /usr/sbin/ceph-disk -v activate --mark-init systemd --mount /dev/vdb1
```
      


* 查看OSD初始化情况
```
[cephuser@ceph-admin cluster]$ ceph-deploy disk list ceph-node1 ceph-node2 ceph-node3

........

[ceph-node1][DEBUG ]  /dev/vdb2 ceph journal, for /dev/vdb1      #如下显示这两个分区，则表示成功了

[ceph-node1][DEBUG ]  /dev/vdb1 ceph data, active, cluster ceph, osd.0, journal /dev/vdb2

........

[ceph-node3][DEBUG ]  /dev/vdb2 ceph journal, for /dev/vdb1       

[ceph-node3][DEBUG ]  /dev/vdb1 ceph data, active, cluster ceph, osd.1, journal /dev/vdb2

.......

[ceph-node3][DEBUG ]  /dev/vdb2 ceph journal, for /dev/vdb1

[ceph-node3][DEBUG ]  /dev/vdb1 ceph data, active, cluster ceph, osd.2, journal /dev/vdb2
```
     

     

- *用ceph-deploy把配置文件和admin密钥拷贝到管理节点和Ceph节点，这样你每次执行Ceph命令行时就无需指定monit节点地址和ceph.client.admin.keyring了*

    `[cephuser@ceph-admin cluster]$ ceph-deploy admin ceph-admin ceph-node1 ceph-node2 ceph-node3`

     

* 修改密钥权限

    `[cephuser@ceph-admin cluster]$ sudo chmod 644 /etc/ceph/ceph.client.admin.keyring`

     

* 检查ceph状态
```
[cephuser@ceph-admin cluster]$ sudo ceph health

HEALTH_OK

[cephuser@ceph-admin cluster]$ sudo ceph -s

    cluster 33bfa421-8a3b-40fa-9f14-791efca9eb96

     health HEALTH_OK

     monmap e1: 1 mons at {ceph-admin=192.168.10.220:6789/0}

            election epoch 3, quorum 0 ceph-admin

     osdmap e14: 3 osds: 3 up, 3 in

            flags sortbitwise,require_jewel_osds

      pgmap v29: 64 pgs, 1 pools, 0 bytes data, 0 objects

            100 MB used, 45946 MB / 46046 MB avail

                  64 active+clean

```

* 查看ceph osd运行状态


```
[cephuser@ceph-admin ~]$ ceph osd stat
osdmap e19: 3 osds: 3 up, 3 in flags sortbitwise,require_jewel_osds
```

   

* 查看osd的目录树

```
[cephuser@ceph-admin ~]$ ceph osd tree

ID WEIGHT  TYPE NAME           UP/DOWN REWEIGHT PRIMARY-AFFINITY

-1 0.04376 root default                                       

-2 0.01459     host ceph-node1                                

 0 0.01459         osd.0            up  1.00000          1.00000

-3 0.01459     host ceph-node2                                

 1 0.01459         osd.1            up  1.00000          1.00000

-4 0.01459     host ceph-node3                                

 2 0.01459         osd.2            up  1.00000          1.00000
```


- 查看monit监控节点的服务情况

```
[cephuser@ceph-admin cluster]$ sudo systemctl status ceph-mon@ceph-admin

[cephuser@ceph-admin cluster]$ ps -ef|grep ceph|grep 'cluster'

ceph     28190     1  0 11:44 ?        00:00:01 /usr/bin/ceph-mon -f --cluster ceph --id ceph-admin --setuser ceph --setgroup ceph
```

    

- 分别查看下ceph-node1、ceph-node2、ceph-node3三个节点的osd服务情况，发现已经在启动中。

```
[cephuser@ceph-node1 ~]$ sudo systemctl status ceph-osd@0.service         #启动是start、重启是restart

[cephuser@ceph-node1 ~]$ sudo ps -ef|grep ceph|grep "cluster"

ceph     28749     1  0 11:44 ?        00:00:00 /usr/bin/ceph-osd -f --cluster ceph --id 0 --setuser ceph --setgroup ceph

cephuser 29197 29051  0 11:54 pts/2    00:00:00 grep --color=auto cluster

  

[cephuser@ceph-node2 ~]$ sudo systemctl status ceph-osd@1.service

[cephuser@ceph-node2 ~]$ sudo ps -ef|grep ceph|grep "cluster"

ceph     28749     1  0 11:44 ?        00:00:00 /usr/bin/ceph-osd -f --cluster ceph --id 0 --setuser ceph --setgroup ceph

cephuser 29197 29051  0 11:54 pts/2    00:00:00 grep --color=auto cluster

  

[cephuser@ceph-node3 ~]$ sudo systemctl status ceph-osd@2.service

[cephuser@ceph-node3 ~]$ sudo ps -ef|grep ceph|grep "cluster"

ceph     28749     1  0 11:44 ?        00:00:00 /usr/bin/ceph-osd -f --cluster ceph --id 0 --setuser ceph --setgroup ceph

cephuser 29197 29051  0 11:54 pts/2    00:00:00 grep --color=auto cluster
```


## 第四步:创建文件系统

> 先查看管理节点状态，默认是没有管理节点的。

```
[cephuser@ceph-admin ~]$ ceph mds stat
未创建：
e1:
已创建：
cephfs-1/1/1 up  {0=ceph-node1=up:active}, 1 up:standby
```


* 创建管理节点（ceph-admin作为管理节点）。

> 需要注意：如果不创建mds管理节点，client客户端将不能正常挂载到ceph集群！！


```
[cephuser@ceph-admin ~]$ pwd

/home/cephuser

[cephuser@ceph-admin ~]$ cd cluster/

[cephuser@ceph-admin cluster]$ ceph-deploy mds create ceph-admin
```

  

* 再次查看管理节点状态，发现已经在启动中


```
[cephuser@ceph-admin cluster]$ ceph mds stat

e2:, 1 up:standby

[cephuser@ceph-admin cluster]$ sudo systemctl status ceph-mds@ceph-admin

[cephuser@ceph-admin cluster]$ ps -ef|grep cluster|grep ceph-mds

ceph     29093     1  0 12:46 ?        00:00:00 /usr/bin/ceph-mds -f --cluster ceph --id ceph-admin --setuser ceph --setgroup ceph

```
  

* 创建pool，pool是ceph存储数据时的逻辑分区,它起到namespace的作用

```
[cephuser@ceph-admin cluster]$ ceph osd lspools           #先查看pool

0 rbd,
```


- 新创建的ceph集群只有rdb一个pool。这时需要创建一个新的pool

```
[cephuser@ceph-admin cluster]$ ceph osd pool create cephfs_data 10       #后面的数字是PG_NUM的数量

pool 'cephfs_data' created

  

[cephuser@ceph-admin cluster]$ ceph osd pool create cephfs_metadata 10     #创建pool的元数据

pool 'cephfs_metadata' created

  

[cephuser@ceph-admin cluster]$ ceph fs new myceph cephfs_metadata cephfs_data

new fs with metadata pool 2 and data pool 1
```


- 再次查看pool状态


```
[cephuser@ceph-admin cluster]$ ceph osd lspools

0 rbd,1 cephfs_data,2 cephfs_metadata,
```


- 检查mds管理节点状态
```
[cephuser@ceph-admin cluster]$ ceph mds stat

e5: 1/1/1 up {0=ceph-admin=up:active}
```

  

- 查看ceph集群状态

```
[cephuser@ceph-admin cluster]$ sudo ceph -s

    cluster 33bfa421-8a3b-40fa-9f14-791efca9eb96

     health HEALTH_OK

     monmap e1: 1 mons at {ceph-admin=192.168.10.220:6789/0}

            election epoch 3, quorum 0 ceph-admin

      fsmap e5: 1/1/1 up {0=ceph-admin=up:active}           #多了此行状态

     osdmap e19: 3 osds: 3 up, 3 in

            flags sortbitwise,require_jewel_osds

      pgmap v48: 84 pgs, 3 pools, 2068 bytes data, 20 objects

            101 MB used, 45945 MB / 46046 MB avail

                  84 active+clean
```


- 查看ceph集群端口

```
[cephuser@ceph-admin cluster]$ sudo lsof -i:6789

COMMAND    PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME

ceph-mon 28190 ceph   10u  IPv4  70217      0t0  TCP ceph-admin:smc-https (LISTEN)

ceph-mon 28190 ceph   19u  IPv4  70537      0t0  TCP ceph-admin:smc-https->ceph-node1:41308 (ESTABLISHED)

ceph-mon 28190 ceph   20u  IPv4  70560      0t0  TCP ceph-admin:smc-https->ceph-node2:48516 (ESTABLISHED)

ceph-mon 28190 ceph   21u  IPv4  70583      0t0  TCP ceph-admin:smc-https->ceph-node3:44948 (ESTABLISHED)

ceph-mon 28190 ceph   22u  IPv4  72643      0t0  TCP ceph-admin:smc-https->ceph-admin:51474 (ESTABLISHED)

ceph-mds 29093 ceph    8u  IPv4  72642      0t0  TCP ceph-admin:51474->ceph-admin:smc-https (ESTABLISHED)

```

## 第五步:client端挂载ceph存储（采用fuse方式）

    安装ceph-fuse（这里的客户机是centos6系统）

```
[root@centos6-02 ~]# rpm -Uvh https://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm

[root@centos6-02 ~]# yum install -y ceph-fuse
```

 

创建挂载目录

[root@centos6-02 ~]# mkdir /cephfs

 

复制配置文件

将ceph配置文件ceph.conf从管理节点copy到client节点（192.168.10.220为管理节点）

[root@centos6-02 ~]# rsync -e "ssh -p22" -avpgolr root@192.168.10.220:/etc/ceph/ceph.conf /etc/ceph/

或者

[root@centos6-02 ~]# rsync -e "ssh -p22" -avpgolr root@192.168.10.220:/home/cephuser/cluster/ceph.conf /etc/ceph/    #两个路径下的文件内容一样

 

复制密钥

将ceph的ceph.client.admin.keyring从管理节点copy到client节点

[root@centos6-02 ~]# rsync -e "ssh -p22" -avpgolr root@192.168.10.220:/etc/ceph/ceph.client.admin.keyring /etc/ceph/

或者

[root@centos6-02 ~]# rsync -e "ssh -p22" -avpgolr root@192.168.10.220:/home/cephuser/cluster/ceph.client.admin.keyring /etc/ceph/

 

查看ceph授权

[cephuser@ceph-admin ~]$ ceph auth list
installed auth entries:

mds.ceph-admin
	key: AQB2WCBdZtP3BxAA+m9BCr4rVPjyiG41FkVijg==
	caps: [mds] allow
	caps: [mon] allow profile mds
	caps: [osd] allow rwx
osd.0
	key: AQAHVCBdNP3+ORAAITW7m+//GQM91B7YxOUTkg==
	caps: [mgr] allow profile osd
	caps: [mon] allow profile osd
	caps: [osd] allow *
osd.1
	key: AQAcVCBdrRfsMxAAxPDKO781HGItz7YuFsTifQ==
	caps: [mgr] allow profile osd
	caps: [mon] allow profile osd
	caps: [osd] allow *
osd.2
	key: AQCzVCBdl6B9EhAAryb5aguveSpi59tFMkPeUg==
	caps: [mgr] allow profile osd
	caps: [mon] allow profile osd
	caps: [osd] allow *
client.admin
	key: AQC3TiBd7v5pDBAAYiNE+fdeF+UxAVpX4c0CuQ==
	caps: [mds] allow *
	caps: [mgr] allow *
	caps: [mon] allow *
	caps: [osd] allow *
client.bootstrap-mds
	key: AQC3TiBdyVJqDBAAocpbDtoHAWwN+F98JkSvVw==
	caps: [mon] allow profile bootstrap-mds
client.bootstrap-mgr
	key: AQC3TiBd02xqDBAA8njK+RDFLOqf08LbVfmbSg==
	caps: [mon] allow profile bootstrap-mgr
client.bootstrap-osd
	key: AQC3TiBdrIZqDBAA/aDwi1IbqjoM/TPuHNyNFw==
	caps: [mon] allow profile bootstrap-osd
client.bootstrap-rbd
	key: AQC3TiBd4qFqDBAAiIcn/yagCNpp5fY5q4R3pg==
	caps: [mon] allow profile bootstrap-rbd
client.bootstrap-rgw
	key: AQC3TiBdnL1qDBAAwiqHykKEvK8NWAqrDUWn/Q==
	caps: [mon] allow profile bootstrap-rgw
mgr.ceph-admin
	key: AQCHVyBda1/1JBAAEPz9VYv/V9NFJMtFDFjN/A==
	caps: [mds] allow *
	caps: [mon] allow profile mgr
	caps: [osd] allow *

 

mds.ceph-admin

    key: AQAZZxdbH6uAOBAABttpSmPt6BXNtTJwZDpSJg==

    caps: [mds] allow

    caps: [mon] allow profile mds

    caps: [osd] allow rwx

osd.0

    key: AQCuWBdbV3TlBBAA4xsAE4QsFQ6vAp+7pIFEHA==

    caps: [mon] allow profile osd

    caps: [osd] allow *

osd.1

    key: AQC6WBdbakBaMxAAsUllVWdttlLzEI5VNd/41w==

    caps: [mon] allow profile osd

    caps: [osd] allow *

osd.2

    key: AQDJWBdbz6zNNhAATwzL2FqPKNY1IvQDmzyOSg==

    caps: [mon] allow profile osd

    caps: [osd] allow *

client.admin

    key: AQCNWBdbf1QxAhAAkryP+OFy6wGnKR8lfYDkUA==

    caps: [mds] allow *

    caps: [mon] allow *

    caps: [osd] allow *

client.bootstrap-mds

    key: AQCNWBdbnjLILhAAT1hKtLEzkCrhDuTLjdCJig==

    caps: [mon] allow profile bootstrap-mds

client.bootstrap-mgr

    key: AQCOWBdbmxEANBAAiTMJeyEuSverXAyOrwodMQ==

    caps: [mon] allow profile bootstrap-mgr

client.bootstrap-osd

    key: AQCNWBdbiO1bERAARLZaYdY58KLMi4oyKmug4Q==

    caps: [mon] allow profile bootstrap-osd

client.bootstrap-rgw

    key: AQCNWBdboBLXIBAAVTsD2TPJhVSRY2E9G7eLzQ==

    caps: [mon] allow profile bootstrap-rgw

 

 

将ceph集群存储挂载到客户机的/cephfs目录下

[root@centos6-02 ~]# ceph-fuse -m 192.168.10.220:6789 /cephfs

2018-06-06 14:28:54.149796 7f8d5c256760 -1 init, newargv = 0x4273580 newargc=11

ceph-fuse[16107]: starting ceph client

ceph-fuse[16107]: starting fuse

 

[root@centos6-02 ~]# df -h

Filesystem            Size  Used Avail Use% Mounted on

/dev/mapper/vg_centos602-lv_root

                       50G  3.5G   44G   8% /

tmpfs                 1.9G     0  1.9G   0% /dev/shm

/dev/vda1             477M   41M  412M   9% /boot

/dev/mapper/vg_centos602-lv_home

                       45G   52M   43G   1% /home

/dev/vdb1              20G  5.1G   15G  26% /data/osd1

ceph-fuse              45G  100M   45G   1% /cephfs

 

由上面可知，已经成功挂载了ceph存储，三个osd节点，每个节点有15G（在节点上通过"lsblk"命令可以查看ceph data分区大小）,一共45G！

 

取消ceph存储的挂载

[root@centos6-02 ~]# umount /cephfs

 

温馨提示：

当有一半以上的OSD节点挂掉后，远程客户端挂载的Ceph存储就会使用异常了，即暂停使用。比如本案例中有3个OSD节点，当其中一个OSD节点挂掉后（比如宕机），

客户端挂载的Ceph存储使用正常；但当有2个OSD节点挂掉后，客户端挂载的Ceph存储就不能正常使用了（表现为Ceph存储目录下的数据读写操作一直卡着状态），

当OSD节点恢复后，Ceph存储也会恢复正常使用。OSD节点宕机重新启动后，osd程序会自动起来（通过监控节点自动起来）



## 使用Ceph RBD

Ceph RBD
官方demo：https://github.com/kubernetes-incubator/external-storage/tree/master/ceph/rbd

```

1.先创建deploy/rbac
kubectl apply -f deploy/rbac
2.测试POD 
kubectl apply -f examples


  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: fast
  provisioner: kubernetes.io/rbd
  parameters:
    monitors: 10.16.153.105:6789
    adminId: kube
    adminSecretName: ceph-secret
    adminSecretNamespace: kube-system
    pool: kube
    userId: kube
    userSecretName: ceph-secret-user
    fsType: ext4
    imageFormat: "2"

monitors: Ceph monitors, comma delimited. It is required.
adminId: Ceph client ID that is capable of creating images in the pool. Default is "admin".
adminSecretName: Secret Name for adminId. It is required. The provided secret must have type "kubernetes.io/rbd".
adminSecretNamespace: The namespace for adminSecret. Default is "default".
pool: Ceph RBD pool. Default is "rbd".
userId: Ceph client ID that is used to map the RBD image. Default is the same as adminId.
userSecretName: The name of Ceph Secret for userId to map RBD image. It must exist in the same namespace as PVCs. It is required.
fsType: fsType that are supported by kubernetes. Default: "ext4".
imageFormat: Ceph RBD image format, "1" or "2". Default is "2".
imageFeatures: Ceph RBD image format 2 features, comma delimited. This is optional, and only be used if you set imageFormat to "2". Currently supported features are layering only. Default is "", no features is turned on.
NOTE: We cannot turn on exclusive-lock feature for now (and object-map, fast-diff, journaling which require exclusive-lock), because exclusive lock and advisory lock cannot work together. (See #45805) 

 
(combined from similar events): MountVolume.WaitForAttach failed for volume "pvc-06af5836-a007-11e9-96aa-5254c0a81b20" : rbd: map failed exit status 2, rbd output: 2019-07-06 16:30:23.392202 7f6524ca1100 -1 did not load config file, using default settings. 2019-07-06 16:30:23.398414 7f6524ca1100 -1 auth: unable to find a keyring on /etc/ceph/ceph.client.admin.keyring,/etc/ceph/ceph.keyring,/etc/ceph/keyring,/etc/ceph/keyring.bin: (2) No such file or directory libkmod: ERROR ../libkmod/libkmod.c:586 kmod_search_moddep: could not open moddep file '/lib/modules/3.10.0-862.11.6.el7.x86_64/modules.dep.bin' modinfo: ERROR: Module alias rbd not found. modprobe: ERROR: ../libkmod/libkmod.c:586 kmod_search_moddep() could not open moddep file '/lib/modules/3.10.0-862.11.6.el7.x86_64/modules.dep.bin' modprobe: FATAL: Module rbd not found in directory /lib/modules/3.10.0-862.11.6.el7.x86_64 rbd: failed to load rbd kernel module (1) rbd: sysfs write failed In some cases useful info is found in syslog - try "dmesg | tail" or so. rbd: map failed: (2) No such file or directory	

原因：rancher创建的集群由于是docker化的，导致/lib/modules为挂载到容器中。修改默认挂载参数即可！
解决方案：https://github.com/rancher/rancher/issues/13774#issuecomment-421633426
```


引申：mimic版本 ceph-mgr

 
mimic 版 dashboard 安装

1、添加mgr 功能

 　　# ceph-deploy mgr create node1 node2 node3

 2、开启dashboard 功能

 　　# ceph mgr module enable dashboard

3、创建证书

　　# ceph dashboard create-self-signed-cert

4、创建 web 登录用户密码

　　# ceph dashboard set-login-credentials admin admin
如果报错：使用  pip install requests urllib3 pyOpenSSL --force --upgrade  修复

ceph dashboard set-login-credentials admin admin@2019
Error EIO: Module 'dashboard' has experienced an error and cannot handle commands: No module named 'requests.packages.urllib3'

5、查看服务访问方式

　　# ceph mgr services



相关记录:

-------------------------------------------------------------------

清除ceph存储信息，重新安装

清除安装包

[cephuser@ceph-admin ~]$ ceph-deploy purge ceph1 ceph2 ceph3

  

清除配置信息

[cephuser@ceph-admin ~]$ ceph-deploy purgedata ceph1 ceph2 ceph3

[cephuser@ceph-admin ~]$ ceph-deploy forgetkeys

  

每个节点删除残留的配置文件

[cephuser@ceph-admin ~]$ sudo rm -rf /var/lib/ceph/osd/*

[cephuser@ceph-admin ~]$ sudo rm -rf /var/lib/ceph/mon/*

[cephuser@ceph-admin ~]$ sudo rm -rf /var/lib/ceph/mds/*

[cephuser@ceph-admin ~]$ sudo rm -rf /var/lib/ceph/bootstrap-mds/*

[cephuser@ceph-admin ~]$ sudo rm -rf /var/lib/ceph/bootstrap-osd/*

[cephuser@ceph-admin ~]$ sudo rm -rf /var/lib/ceph/bootstrap-mon/*

[cephuser@ceph-admin ~]$ sudo rm -rf /var/lib/ceph/tmp/*

[cephuser@ceph-admin ~]$ sudo rm -rf /etc/ceph/*

[cephuser@ceph-admin ~]$ sudo rm -rf /var/run/ceph/*

 

-----------------------------------------------------------------------

查看ceph命令，关于ceph osd、ceph mds、ceph mon、ceph pg的命令

[cephuser@ceph-admin ~]$ ceph --help             

 

-----------------------------------------------------------------------

如下报错

[cephuser@ceph-admin ~]$ ceph osd tree

2018-06-06 14:56:27.843841 7f8a0b6dd700 -1 auth: unable to find a keyring on /etc/ceph/ceph.client.admin.keyring,/etc/ceph/ceph.keyring,/etc/ceph/keyring,/etc/ceph/keyring.bin: (2) No such file or directory

2018-06-06 14:56:27.843853 7f8a0b6dd700 -1 monclient(hunting): ERROR: missing keyring, cannot use cephx for authentication

2018-06-06 14:56:27.843854 7f8a0b6dd700  0 librados: client.admin initialization error (2) No such file or directory

Error connecting to cluster: ObjectNotFound

[cephuser@ceph-admin ~]$ ceph osd stat

2018-06-06 14:55:58.165882 7f377a1c9700 -1 auth: unable to find a keyring on /etc/ceph/ceph.client.admin.keyring,/etc/ceph/ceph.keyring,/etc/ceph/keyring,/etc/ceph/keyring.bin: (2) No such file or directory

2018-06-06 14:55:58.165894 7f377a1c9700 -1 monclient(hunting): ERROR: missing keyring, cannot use cephx for authentication

2018-06-06 14:55:58.165896 7f377a1c9700  0 librados: client.admin initialization error (2) No such file or directory

 

解决办法：

[cephuser@ceph-admin ~]$ sudo chmod 644 /etc/ceph/ceph.client.admin.keyring

 
查看OSD状态

[cephuser@ceph-admin ~]$ ceph osd stat
3 osds: 3 up, 3 in; epoch: e30

[cephuser@ceph-admin ~]$ ceph osd tree
ID CLASS WEIGHT  TYPE NAME           STATUS REWEIGHT PRI-AFF
-1       0.73228 root default
-3       0.24409     host ceph-node1
 0   hdd 0.24409         osd.0           up  1.00000 1.00000
-5       0.24409     host ceph-node2
 1   hdd 0.24409         osd.1           up  1.00000 1.00000
-7       0.24409     host ceph-node3
 2   hdd 0.24409         osd.2           up  1.00000 1.00000




## Ceph IOPS测试
    
    指的是系统在单位时间内能处理的最大的I/O频度，是衡量磁盘性能的主要指标之一
    (1) 用以上创建的块设备，用fio命令对该设备进行压测读，其中bs=4k，先写入设备，线程深度大点-iodepth 16

```
[root@node-1 ~]# fio -filename=/dev/rbd0 -direct=1 -iodepth 16 -thread -rw=write -ioengine=psync -bs=4k -size=10G -numjobs=10 -name=mytest --eta-newline=1

fio -filename=/dev/rbd0 -direct=1 -iodepth 16 -thread -rw=write -ioengine=psync -bs=16k -size=1G -numjobs=10 -name=mytest --eta-newline=1

随机写测试
fio -filename=/dev/rbd0  -direct=1 -rw=randwrite -bs=8k -size=1G -numjobs=16 -runtime=60 -group_reporting -name=fio_test


fio -filename=/home/cephuser/test -direct=1 -iodepth 1 -thread -rw=randwrite -ioengine=psync -bs=4k -size=10G -numjobs=50 -runtime=60 -group_reporting -name=rand_100write_4k
```
