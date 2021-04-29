---
title: 记一次 Mysql 集群搭建经历
description: 整整三天啊，太菜了。
date: 2021-04-29
slug: pxc-cluster
image: img/miku.jpg
categories:
  - mysql
---

## 常用的两种方案

- MySQL Cluster：ndb 存储引擎，比较复杂，业界并没有大规模使用（不支持 InnoDB）
- PXC(Percona XtraDB Cluster)

由于当前部门没有专业的运维人员，故采取了 PXC 作为本次系统的部署方案。

## PXC 的特点

- 速度慢，但能保证强一致性，适用于保存价值较高的数据，比如订单、客户、支付等。
- 数据同步是双向的，在任一节点写入数据，都会同步到其他所有节点，在任何节点上都能同时读写。
- 采用同步复制，向任一节点写入数据，只有所有节点都同步成功后，才会向客户端返回成功。事务在所有节点要么同时提交，要么不提交。

PXC 集群的架构图如下：

![架构图](img/pxc-struct.jpg)

## 开始部署

本次部署均基于 **Docker**

### 单台机器

**注意**：虽然整个集群的所有节点都在一个 `docker-compose.yml` 文件中，但是我们还是要一个一个节点启动，不然很可能一个节点都起不来；另外出现问题需要重启的时候，最好删除 container 以及 volume，不然也会直接挂掉。

单台机器的情况下比较简单，整个集群处在一个 Docker Network 环境下，仅向外界暴露`3306`即可。下面给出具体的`docker-compose.yml`

```yml
# 3.5版本开始可以指定volume的名称
version: "3.5"

services:
  # 这里，我们将mysql1当成集群中的bootstrap节点
  mysql1:
    image: percona/percona-xtradb-cluster:5.7
    container_name: mysql1
    privileged: true
    volumes:
      # persist数据
      - v1:/var/lib/mysql
      # 我这边的出来的结论是，bootstrap节点使用自定义cnf启动，其他节点使用默认配置
      - ./mysql/cnf/my1.cnf:/etc/my.cnf
    ports:
      - "14306:3306"
    environment:
      - TZ=Asia/Shanghai
      - MYSQL_ROOT_PASSWORD=root # root密码
      - CLUSTER_NAME=mysql-cluster # 集群名称，各个节点保持一致
      - XTRABACKUP_PASSWORD=pass # 节点间同步数据的用户xtrabackup的密码，各个节点必须保持一致
      # CLUSTER_JOIN配置为空时，会自动以bootstrap模式启动
      # - CLUSTER_JOIN=172.31.103.116
    networks:
      my-network:
        # 指定每个节点的ip
        ipv4_address: 172.31.0.11
  mysql2:
    image: percona/percona-xtradb-cluster:5.7
    container_name: mysql2
    privileged: true
    volumes:
      - v2:/var/lib/mysql
    # - ./mysql/cnf/my2.cnf:/etc/my.cnf
    ports:
      - "14307:3306"
    environment:
      - TZ=Asia/Shanghai
      - MYSQL_ROOT_PASSWORD=root
      - CLUSTER_NAME=mysql-cluster
      - XTRABACKUP_PASSWORD=pass
      # 集群中bootstrap节点的ip（由于在一个docker网络中，这里采用内部DNS访问）
      - CLUSTER_JOIN=mysql1
    depends_on:
      - mysql1
    networks:
      my-network:
        ipv4_address: 172.31.0.12
  mysql3:
    image: percona/percona-xtradb-cluster:5.7
    container_name: mysql3
    # restart: always
    privileged: true
    volumes:
      - v3:/var/lib/mysql
      # - ./mysql/cnf/my3.cnf:/etc/my.cnf
    ports:
      - "14308:3306"
    environment:
      - TZ=Asia/Shanghai
      - MYSQL_ROOT_PASSWORD=root
      - CLUSTER_NAME=mysql-cluster
      - XTRABACKUP_PASSWORD=pass
      - CLUSTER_JOIN=mysql1
    depends_on:
      - mysql1
    networks:
      my-network:
        ipv4_address: 172.31.0.13
networks:
  # 定义整个pxc集群的网络环境
  my-network:
    ipam:
      driver: default
      config:
        # 子网掩码需要兼容当前物理机IP，此处仅作参考
        - subnet: "172.31.0.0/24"
# 定义节点挂载用的volume
volumes:
  v1:
    name: v1
  v2:
    name: v2
  v3:
    name: v3
```

bootstrap 节点采用的配置如下:

```properties
[client]
# 节点内部登录mysql需要的文件
socket=/var/lib/mysql/mysql.sock

[mysqld]
# 当前节点id，集群中节点id不能重复
server-id=1
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
default_storage_engine=InnoDB
binlog_format=ROW

innodb_flush_log_at_trx_commit  = 0
innodb_flush_method             = O_DIRECT
innodb_file_per_table           = 1
innodb_autoinc_lock_mode=2

character_set_server = utf8
bind_address = 0.0.0.0

wsrep_slave_threads=2
# 集群中所有节点的ip集合，留空会以bootstrap模式启动
wsrep_cluster_address=gcomm://
wsrep_provider=/usr/lib64/galera3/libgalera_smm.so

wsrep_cluster_name=mysql-cluster
wsrep_sst_method=xtrabackup-v2
# 集群中同步数据的用户和密码
wsrep_sst_auth="xtrabackup:pass"
```

#### 遇到的一些问题

> 如果 bootstrap 以外的节点也以自定义 cnf 启动的话？

- 会直接挂掉

> 如果 bootstrap 以外的节点以默认配置启动的话？

- 集群成功建立连接了，但是只能在 bootstrap 节点登录 mysql。

#### 检查集群是否成功建立连接的方式

登录 mysql 之后，`show status like 'wsrep%';`，结果如下：

![集群状态](img/pxc-status.png)

### 多台机器

由于涉及到多台服务器，如果有可能的话，可以采用 k8s 或者 swarm，或者采用 host 模式，不过我们的部署环境以上条件都不满足就是了。

#### 1. 首先部署 bootstrap 节点

```yml
# 通过环境变量填充下列参数(.env文件)
version: "3.5"

services:
  pxc:
    image: percona/percona-xtradb-cluster:${PXC_VERSION}
    container_name: pxc
    privileged: true
    volumes:
      - data:/var/lib/mysql # persist数据
      - ./conf/my.cnf:/etc/my.cnf # 配置文件
      - ./init:/docker-entrypoint-initdb.d/ # 初始化脚本文件
    ports:
      # 不采用host模式，选择映射所有端口
      - "3307:3306"
      - "4444:4444"
      - "4567:4567"
      - "4568:4568"
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - CLUSTER_NAME=${CLUSTER_NAME}
      - XTRABACKUP_PASSWORD=${XTRABACKUP_PASSWORD}
      # - CLUSTER_JOIN=${CLUSTER_JOIN} # bootstrap节点以外填充bootstrap节点的IP
volumes:
  data:
    name: data
```

```properties
[client]
socket=/var/lib/mysql/mysql.sock

[mysqld]
# 当前节点Id，集群中不能重复
server-id=1
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
default_storage_engine=InnoDB
binlog_format=ROW

innodb_flush_log_at_trx_commit  = 0
innodb_flush_method             = O_DIRECT
innodb_file_per_table           = 1
innodb_autoinc_lock_mode=2

character_set_server = utf8
bind_address = 0.0.0.0

wsrep_slave_threads=2

# bootstrap节点留空，以bootstrap模式启动
wsrep_cluster_address=gcomm://
# 其他节点填充整个集群中的所有节点IP(以逗号分隔)
wsrep_cluster_address=gcomm://IP1, IP2, IP3
wsrep_provider=/usr/lib64/galera3/libgalera_smm.so
# 集群名称
wsrep_cluster_name=mysql-cluster

# 当前节点名称
wsrep_node_name=pxc1
# 当前节点IP
wsrep_node_address=IP1

wsrep_sst_method=xtrabackup-v2
# 集群间备份数据的用户和密码
wsrep_sst_auth="xtrabackup:changeme"
```

后续还会继续学习Haproxy+Keepalived，现在机器可能没那么多，没办法太HA。

另外通过ETCD通信的方案也会研究一下。

## 参考链接

- [PXC 官方文档](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/index.html)
- [如何搭建 Percona XtraDB Cluster（PXC）集群](https://blog.csdn.net/qq_26402707/article/details/107786504)
- [Docker 部署 Mysql 集群的实现](https://www.jb51.net/article/195587.htm)
- [Neokylin-Server 离线环境、跨主机、使用 Docker 部署 PXC 集群](https://blog.csdn.net/weixin_43538901/article/details/113807116)
