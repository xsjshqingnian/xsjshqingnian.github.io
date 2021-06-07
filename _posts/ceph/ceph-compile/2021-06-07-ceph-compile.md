---
layout: post
title: 编译ceph 13.2.10
categories: [ceph]
description: 编译ceph
keywords: ceph, compile

---

## 1.修改pip源

+ mkdir ~/.pip

+ touch ~/.pip/pip.conf




添加如下内容：

```
[global]
index-url = https://pypi.mirrors.ustc.edu.cn/simple/
[install]
use-mirrors =true
mirrors = https://pypi.mirrors.ustc.edu.cn/simple/
trusted-host = pypi.mirrors.ustc.edu.cn
```

## 2.安装httpd

```
yum install httpd -y
mkdir /home/file
cd /var/www/html
ln -s /home/file file

#将boost_1_67_0.tar.bz2放入/home/file
#启动httpd，访问http://ip/file

vim cmake/modules/BuildBoost.cmake：149
#将内容换为以下：
set(boost_url 
      http://127.0.0.1/file/boost_${boost_version_underscore}.tar.bz2)

```

## 3.其他加速技巧

```
yum install http://mirrors.aliyun.com/epel/7/x86_64/Packages/e/epel-release-7-13.noarch.rpm
rm -rf /etc/yum.repos.d/epel*
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
sed -i '/aliyuncs/d' /etc/yum.repos.d/epel.repo


#进入ceph源码目录：
sed -i 's/https:\/\/dl.fedoraproject.org\/pub\//https:\/\/mirrors.aliyun.com\//g' install-deps.sh

do_cmake.sh注释掉原来的命令，如下：
#{CMAKE} -DBOOST_J=$(nproc) $ARGS "$@" ..   
添加新的cmake命令参数:
${CMAKE} -DCMAKE_C_FLAGS="-O0 -g3 -gdwarf-4" -DCMAKE_CXX_FLAGS="-O0 -g3 -gdwarf-4" -DBOOST_J=$(nproc) $ARGS "$@" ..

#进入build目录
{CMAKE} .. -DWITH_LTTNG=OFF -DWITH_RDMA=OFF -DWITH_FUSE=OFF -DWITH_DPDK=OFF -DCMAKE_INSTALL_PREFIX=/usr 
```

## 4. 忠告

如果extract执行很长时间，等它，然后打断，删除对应的node_modules目录，重新编译即可。