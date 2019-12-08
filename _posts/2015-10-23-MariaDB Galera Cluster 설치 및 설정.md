---
published: true
layout: single
toc: true
title: MariaDB Galera Cluster 설치 및 설정
category: infra
tags: mariadb
comments: true
---

Active / Active 구성이 가능한 MySQL Cluster 제품 Galera Cluster 의 설치 / 초기 설정 (MariaDB ver 10.1 , centos 7.x 기준)


## Repository 등록
```
# MariaDB 10.1 CentOS repository list - created 2015-10-22 18:18 UTC
# http://mariadb.org/mariadb/repositories/
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.1/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
```

## 패키지 설치
```
sudo yum install MariaDB-server MariaDB-client galera
```
MariaDB 가 10.1 버전으로 업데이트 되면서 기존 10.0 버전까지는 일반 MariaDB-server 패키지와 MariaDB-Galera-server 패키지로 나뉘어 배포되던 것이 통합된 하나의 패키지로 합쳐져 배포되기 시작했음.

## 방화벽 포트 설정
Galera Cluster 를 위해서는 총 4개의 포트가 방화벽에서 열려 있어야 한다.

3306 - MySQL

4567 - Galera Cluster

4568 - IST

4444 - SST



위 포트들의 방화벽을 open 해 주자.

SST 포트인 4444 포트는 아래 my.cnf 옵션 중 wsrep_sst_receive_address 의 값으로 수동 지정할 수도 있다.


## 일단 MySQL 실행 후 해야할 것
Galera Cluster 는 노드간 데이터 전송을 위한 MySQL 계정 등록이 되어 있어야 한다.

해서 우선 root 계정으로 접근하여 Clustering 할 노드 (서버) 별 데이터 전송 계정 생성 및 생성된 계정에 모든 테이블의 권한을 줘야 한다.

```
create user 'replication'@'node_ip' identified by 'password' ;
grant all privileges on *.* to replication@node_ip;
flush all privileges;
```

클러스터링할 서버가 3개면 3개, 4개면 4개.. 다 귀찮고 보안 신경쓰기 싫으면 '%' 로 하나만 등록해도 무관하다.

주의할 것은 모든 노드에 생성할 이중화 계정이 계정명과 암호가 일치해야 한다는 것이다.



서버간 이중화 계정 등록을 마쳤으면 기동 중인 MySQL 을 중지하고, 설정 파일을 손보자.


## 시작 전 my.cnf 설정

```
# * Galera-related settings
[galera]
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
#wsrep_cluster_address=gcomm://
wsrep_cluster_address=gcomm://
wsrep_cluster_name=[cluster_name]
wsrep_node_address=[local_node_ip]
wsrep_node_name=[local_node_name]
wsrep_sst_method=rsync 
wsrep_sst_auth=[replication:password]
#wsrep_sst_receive_address=[sst_port]
innodb_flush_log_at_trx_commit=2
binlog_format=ROW
bind-address=0.0.0.0
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
innodb_locks_unsafe_for_binlog=1
```

모든 노드에 기본적으로 설정해야 되는 내용은 위와 같다.

[cluster_name] - 클러스터링으로 묶일 이중화의 명칭

[local_node_ip] - 클러스터링으로 묶이는 자신의 IP

[local_node_name] - 클러스터링으로 묶이는 자신의 노드 이름

[replication:password] - 4번 과정에서 생성한 이중화 계정 정보 account:password 의 형식으로 입력

[sst_port] - 3번 과정에서 설명한 SST 포트를 커스텀하게 지정하려 한다면 지정하려는 커스텀 포트

노드별로 옵션을 지정하며 특정 노드별로 주의하며 설정할 옵션이 있다.

wsrep_cluster_address 옵션인데, 노드의 성격별로 초기 설정값이 다르다.

클러스터링의 기본이 되어야 할 1번 노드는 해당 값을 비워두고,
1번 노드가 시작된 뒤 2번 노드를 실행할 때는 1번 노드의 ip 만 지정, 
2번 노드를 시작된 뒤 3번 노드를 실행할 때는 1번 노드, 2번 노드 지정 ... 식으로 되풀이.

해당 옵션은 자신이 장애가 발생했을 때 (혹은 재구동할 때) 이중화 복구 데이터를 전송받을 노드의 주소를 의미한다.

1..N 노드까지 모두 MySQL instance 가 클러스터링으로 정상 구동되었다면, N-1 ~ 1 노드까지 다시 my.cnf 를 편집하여 클러스터 내 자신을 제외한 모든 노드의 IP 를 다시 입력해주고 instance 를 재구동 하도록 하자.



## 클러스터 시작

Galera Cluster 는 클러스터 내 1번 노드에서 클러스터 모드를 시작하는 옵션으로 instance 를 구동함으로 시작된다.

```
sudo service mysql start --wsrep-new-cluster
```

위와 같은 옵션으로 1번 노드의 instance 구동이 완료되면 2번 노드 부터는 일반적으로 mysql 을 구동하는 방식 그대로 구동하면 Galera Cluster 에 들러 붙는다.


```
sudo service mysql start
```

노드를 추가할 때마다 아래와 같은 MySQL 시스템 쿼리를 날리면 클러스터링 된 노드의 수를 확인할 수 있다.


```
MariaDB [(none)]> SHOW STATUS LIKE 'wsrep_cluster_size';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 3     |
+--------------------+-------+
1 row in set (0.00 sec)

MariaDB [(none)]> 
```

5번 과정에서 미리 언급했 듯이, N 까지의 클러스터링 붙이는 과정이 모두 완료된 후에는 1~N-1 노드 까지 설정 파일 (my.cnf) 에서 클러스터링 내 노드들의 IP 들을 모두 추가해준 뒤 재구동 하도록 하자.



## Trouble Shooting

```
2015-10-23 15:39:09 139766104323840 [Note] WSREP: Running: 'wsrep_sst_rsync --role 'joiner' --address '' --auth 'replication:password' --datadir ''   --parent '25620'  '' '
which: no lsof in (/usr/sbin:/sbin:/usr//bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin)
2015-10-23 15:39:09 139766104323840 [ERROR] WSREP: Failed to read 'ready <addr>' from: wsrep_sst_rsync --role 'joiner' --address  --auth 'replication:password' --datadir ''   --parent '25620'  '' 
	Read: ''lsof' not found in PATH'
2015-10-23 15:39:09 139766104323840 [ERROR] WSREP: Process completed with error: wsrep_sst_rsync --role 'joiner' --address  --auth 'replication:password' --datadir ''   --parent '25620'  '' : 2 (No such file or directory)
2015-10-23 15:39:09 139766424127232 [ERROR] WSREP: Failed to prepare for 'rsync' SST. Unrecoverable.
2015-10-23 15:39:09 139766424127232 [ERROR] Aborting
```

위와 같은 오류가 나며 추가 노드 instance 가 구동되지 않는다면 linux 에 lsof 패키지가 존재하지 않아 발생하는 오류이니, lsof 패키지를 설치해주면 된다.