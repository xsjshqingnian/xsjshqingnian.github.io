---
layout: post
title: ceph集群安装
categories: ceph
description: 如何载集群中安装ceph
keywords: ceph, install, cluster
---

## 1.Centos 7
    + 关闭防火墙
    + 关闭selinux
    + 配置网络
    + 修改hosts文件

## 2.在deploy节点
    + 执行以下命令(可以不执行)
    ```
    sudo yum install -y yum-utils && sudo yum-config-manager --add-repo https://dl.fedoraproject.org/pub/epel/7/x86_64/ && sudo yum install --nogpgcheck -y epel-release && sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7 && sudo rm /etc/yum.repos.d/dl.fedoraproject.org*
    ```

## 3.所有节点安装：

```
yum install ntp ntpdate ntp-doc python-setuptools yum-plugin-priorities epel-release
```

## 4.所有节点创建用户wuyq

```
1. 创建用户
sudo useradd -d /home/{username} -m {username}
sudo passwd {username}

2. 确保各 Ceph 节点上新创建的用户都有 sudo 权限。
echo "{username} ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/{username}
sudo chmod 0440 /etc/sudoers.d/{username}
```

## 5.在deploy节点上

```
1.切换为wuyq用户
2.执行ssh-keygen
3.免密登录
    ssh-copy-id {username}@node1
    ssh-copy-id {username}@node2
    ssh-copy-id {username}@node3
4. deploy 管理节点上的 ~/.ssh/config 文件
    Host node1
    Hostname node1
    User {username}
    Host node2
    Hostname node2
    User {username}
    Host node3
    Hostname node3
    User {username}

5.修改~/.ssh/config文件权限
    sudo chmod 600 ~/.ssh/config

```

## 6.在deploy节点上创建本地repo

+ 配置ceph源

```
[Ceph]
name=Ceph packages for $basearch
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/$basearch
enabled=1
gpgcheck=0
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
priority=1
[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/noarch
enabled=1
gpgcheck=0
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
priority=1
[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/SRPMS
enabled=1
gpgcheck=0
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
priority=1
```

+ 安装epel-release

```
sudo yum install epel-release
```

+ 下载ceph源repo

```
[cephfsd@deploy ~]$ sudo mkdir -p /yum/ceph/mimic
[cephfsd@deploy ~]$ sudo yum -y install --downloadonly --downloaddir=/yum/ceph/mimic/ ceph ceph-radosgw htop sysstat iftop iotop ntp ntpdate
[cephfsd@deploy ~]$ sudo yum install -y createrepo httpd
[cephfsd@deploy ~]$ sudo yum install -y ceph-deploy python-pip ntp ntpdate
```

+ 配置内部源

```
sudo createrepo -pdo /yum/ceph/mimic/ /yum/ceph/mimic/
如果源的包发生变化，例如加入了新的包，或者删掉了一些旧包，需要更新xml：
sudo createrepo --update /yum/ceph/luminous/
```

+ 修改/etc/httpd/conf/httpd.conf文件，然后启动httpd

```
DocumentRoot "/yum/ceph/"

<Directory "/yum/ceph/">
    AllowOverride None
    # Allow open access:
    Require all granted
</Directory>



[cephfsd@deploy ~]$ sudo systemctl enable httpd.service
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.
[cephfsd@deploy ~]$ sudo systemctl start httpd.service
```

+ 6.替换每个node上的源

```
[cephfsd@ceph1 ~]$ cd /etc/yum.repos.d/
[cephfsd@ceph1 yum.repos.d]$ sudo mkdir backup
[cephfsd@ceph1 yum.repos.d]$ sudo mv ./*.repo backup/
[cephfsd@ceph1 yum.repos.d]$ cat inter.repo
[http-repo]
name=internal-ceph-repo
baseurl=http://deploy/mimic
enabled=1
gpgcheck=0
```

## 7.在每个节点上安装ceph

```
sudo yum -y install  ceph ceph-radosgw htop sysstat iftop iotop ntp ntpdate
```

## 8.在deploy节点上使用wuyq用户

+ 创建目录：my-clu，并进入
+ 执行 `ceph-deploy new node1`（node1为初始monitor节点）

## 9.修改my-clu目录下的ceph.conf配置文件为：

```
[global]
fsid = ec291e8c-6dc2-4e1b-83c5-c40cf36f2978
mon_initial_members = node1
mon_host = 192.168.10.101
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

osd_pool_default_size = 2
public_network = 192.168.10.0/24
cluster_network = 192.168.20.0/24
```

## 10.在deploy节点上使用wuyq用户

```
执行：
1. ceph-deploy mon create-initial
2. ceph-deploy admin node1 node2 node3
3. ceph-deploy mgr create node1

创建osd：
4. ceph-deploy osd create --data {device} {ceph-node}
    如：ceph-deploy osd create --data /dev/sdb node1

5. ceph-deploy mds create {ceph-node}
    如：ceph-deploy mds create node1


6. ceph-deploy rgw create {gateway-node}
    如：ceph-deploy rgw create node1

7. ceph-deploy mon add {ceph-node}
    如：ceph-deploy mon add node2



```

## 11.在集群节点上

```
创建文件系统
ceph osd pool create cephfs_data <pg_num>
ceph osd pool create cephfs_metadata <pg_num>
ceph fs new <fs_name> cephfs_metadata cephfs_data
```

## 12.将node1设置为ntp服务器，node2与node3同步node1的时间

```
1. node1:
[cephfsd@deploy ceph-deploy]$ sudo cat /etc/ntp.conf |grep ^server
server time1.aliyun.com minpoll 3 maxpoll 4 iburst
server time2.aliyun.com minpoll 3 maxpoll 4 iburst
server time3.aliyun.com minpoll 3 maxpoll 4 iburst
[cephfsd@deploy ceph-deploy]$ sudo systemctl enable ntpd.service
Created symlink from /etc/systemd/system/multi-user.target.wants/ntpd.service to /usr/lib/systemd/system/ntpd.service.
[cephfsd@deploy ceph-deploy]$ sudo systemctl start ntpd.service

2. node2或node3：
[cephfsd@ceph1 ~]$ cat /etc/ntp.conf |grep ^server
server node1 minpoll 3 maxpoll 4 iburst
[cephfsd@ceph1 ~]$ sudo systemctl enable ntpd.service
Created symlink from /etc/systemd/system/multi-user.target.wants/ntpd.service to /usr/lib/systemd/system/ntpd.service.
[cephfsd@ceph1 ~]$ sudo systemctl start ntpd.service

3.查看ntp服务状态
ntpq -pn
```

