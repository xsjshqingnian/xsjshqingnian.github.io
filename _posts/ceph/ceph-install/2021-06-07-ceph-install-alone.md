---
layout: post
title: ceph install alone
categories: [ceph]
description: ceph install alone
keywords: ceph, install, alone

---

## 1.Centos 7

+ 关闭防火墙
+ 关闭selinux
+ 配置网络
+ 修改hosts文件

## 2.在节点上创建ceph repo

+ 创建/etc/yum.repos.d/ceph.repo，内容如下：

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

## 3.安装ceph-deploy python-setuptools

```
yum install ceph-deploy python-setuptools -y
```

## 4.创建文件夹

```
mkdir /home/my-clu
```

## 5.创建mon

```
# cephalone为主机名
cd /home/my-clu
ceph-deploy new cephalone
```

## 6.修改配置文件

```
cd /home/my-clu
vim ceph.conf

# 加入以下内容
osd_pool_default_min_size = 1              #pool的备份数
osd_pool_default_size = 1                  #后续创建的osd硬盘的个数
```

## 7.在节点上安装ceph

```
ceph-deploy install cephalone
```

## 8.在节点上执行

```
1. ceph-deploy mon create-initial
2. ceph-deploy admin cephalone
3. ceph-deploy mgr create cephalone

创建osd：
4. ceph-deploy osd create --data {device} {ceph-node}
    如：ceph-deploy osd create --data /dev/sdb cephalone

5. ceph-deploy mds create {ceph-node}
    如：ceph-deploy mds create cephalone

```

## 9.在节点上

```
创建文件系统
ceph osd pool create cephfs_data <pg_num>
ceph osd pool create cephfs_metadata <pg_num>
ceph fs new <fs_name> cephfs_metadata cephfs_data
```

## 10.挂载cephfs

```
# {admin-secret}：为/etc/ceph/ceph.client.admin.keyring文件key值
mount.ceph 192.168.10.100:6789:/ /mnt/ -o name=admin,secret={admin-secret}

192.168.2.125:6789:/     /mnt/ceph    ceph    name=admin,secretfile=/etc/ceph/secret.key,noatime,_netdev    0       2


```

