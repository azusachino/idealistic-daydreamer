---
title: 练习 Minikube
description: 最简单的 k8s 集群方案
date: 2021-08-31
slug: practice-minikube-with-mysql
image: img/2021/08/VeniceBeach.jpg
categories: [Practice]
tags: [Practice, Kubernetes, K8S, MySQL]
---

## Define credentials -> mysql-secret.yml

use `echo -n 'your_password' | base64` to generate a opaque password.

```yml
apiVersion: v1
kind: Secret
metadata:
  namespace: iris
  name: iris-mysql-password
type: Opaque
data:
  password: Y2hpbm8xMjA0
```

## mysql-configmap.yml

a copy of config file.

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: iris
  name: iris-mysql-config
data:
  mysqld.cnf: |-
    [client]
    default-character-set=utf8
    [mysqld]
    pid-file        = /var/run/mysqld/mysqld.pid
    socket          = /var/run/mysqld/mysqld.sock
    datadir         = /var/lib/mysql
    skip_external_locking
    skip-symbolic-links
    memlock=true
    max_connect_errors = 20000
    max_connections = 3000
    skip-name-resolve
    default-time-zone = system
    default-storage-engine = InnoDB
    explicit_defaults_for_timestamp = 1
    lower_case_table_names = 1
    key_buffer_size = 4096M
    table_open_cache = 1024
    sort_buffer_size = 4M
    read_buffer_size = 4M
    thread_cache_size = 128
    query_cache_size = 512M
    character-set-server = utf8
    collation-server = utf8_general_ci
    sql_mode='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'
    # innoDB
    innodb_page_size = 16K
    innodb_read_io_threads  = 4
    innodb_write_io_threads = 4
    innodb_io_capacity = 200
    innodb_io_capacity_max = 2000
```

## mysql-pvc.yml

```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: iris
  name: iris-mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

## mysql-deployment.yml

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: iris
  name: iris-mysql-deployment
  labels:
    name: iris-mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      name: iris-mysql
  template:
    metadata:
      labels:
        name: iris-mysql
    spec:
      containers:
        - name: mysql
          image: mysql:5.7
          imagePullPolicy: IfNotPresent
          args:
            - --default_authentication_plugin=mysql_native_password
          ports:
            - containerPort: 3306
              name: mysql-port
          volumeMounts:
            - mountPath: /var/lib/mysql
              name: mysql-data
              subPath: mysql
            - mountPath: /etc/mysql/mysql.conf.d/
              name: mysql-conf
          env:
            - name: MYSQL_ROOT_PASSWORD # env used in docker image
              valueFrom:
                secretKeyRef:
                  key: password
                  name: iris-mysql-password
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: iris-mysql-pvc
        - name: mysql-conf
          configMap:
            name: iris-mysql-config
```

## mysql-service.yml

Recommend to use `ClusterIP` rather than `NodePort`, here is just for dev practice.

```yml
apiVersion: v1
kind: Service
metadata:
  namespace: iris
  name: iris-mysql-service
  labels:
    name: iris-mysql-service
spec:
  type: NodePort
  selector:
    name: iris-mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
      nodePort: 30036
```

## Last Step

After you have applied all files above, you can access mysql in minikube or inside the docker container of minikube. If you want to access mysql at `localhost`, there are two ways:

1. use minikube service => `minikube service iris-mysql-service -n iris`
2. use kubectl port-forward => `kubectl -n iris port-forward service/iris-mysql-service 33306:3306`
