---
layout: post
title:  "Docker Volume 개념 및 MySql을 Docker상에서 운용하는 방법"
date:   2018-11-13 15:38:01 -0600
tags: docker dev
---

MySQL image를  docker로 실행할 때, 이전 실행시 가지고 있던 정보를 유지하려면, docker volume을 사용해야 한다.

# Docker Volume 특징 및 암시적 생성/제거

docker volume은, docker에서 디스크에 무언가를 기록할 대상을 container로 부터 분리해낸 개념으로, 마치 
사용자가 usb 메모리를 자신이 쓰던 컴퓨터에 꼽는것과 유사하다. 

Docker Volume에 대한 설명을 친절히 알려준 [유투브영상](https://www.youtube.com/watch?v=YK-V8aq32WA)이 있어,
이걸 보고, 개념을 파악함. 

[MySQL 8.0 docker image](https://hub.docker.com/_/mysql/) 8.0버젼의 
[Dockerfile](https://github.com/docker-library/mysql/blob/223f0be1213bbd8647b841243a3114e8b34022f4/8.0/Dockerfile) 을 보면 다음과 같다. 


```
FROM debian:stretch-slim
.
.
.
VOLUME /var/lib/mysql
.
.
.
```

이는 해당 Docker image가 실행될때, DB의 실제 데이터가 저장되는 `/var/lib/mysql' 디렉토리를 외부 저장공간(=Volume)을 사용한다는 의미.
하지만, 이렇게 생성된 volume은 이름이 없다. 

mysql:8.0 을 실행해 본다. 

```
docker run --name testdb -e MYSQL_ROOT_PASSWORD=mirero -d mysql
```

이렇게 하면, docker는 `/var/lib/mysql` 이라는 디렉토리를 외부 volume을 사용해 mount한다. 
실행된 docker container에 mount된 volume을 확인하려면 다음과 같이 한다. 

```
docker inspect -f "{{.Mounts}}" testdb
```

이 명령에 의한 출력예는 다음과 같다.

```
[{volume c4c1fedc28f594882ee6bdbe86bcd754a82dca181d5954d5dc0b03940992ef44 /mnt/sda1/var/lib/docker/volumes/c4c1fedc28f594882ee6bdbe86bcd754a82dca181d5954d5dc0b03940992ef44/_data /var/lib/mysql local  true }]
```

출력된 결과를 보면, 결국 docker host내 /mnt/sda1/var/lib/docker/volumes/c4c1fedc28f594882ee6bdbe86bcd754a82dca181d5954d5dc0b03940992ef44 디렉토리가 있고, 이 경로의 내용이 /var/lib/mysql 로 mount되어 있음을 알 수 있다. volume의 목록을 보기 위해...

```
docker volume ls
```

명령을 수행하면, 다음과 같이 현재 docker host에 있는 volume들이 나열된다. 

```
DRIVER              VOLUME NAME
local               c4c1fedc28f594882ee6bdbe86bcd754a82dca181d5954d5dc0b03940992ef44  <--- 이것만 신경쓰자.
local               d42b71c63042a2e771ed6bc9c59ba896d729950432202a3020c9982954e9eb04
local               d5afd7f8670201495f6246551623e4ce8c5deba00b0b634b878b3def11283cf1
local               portainer_data
```

위 volume 목록을 보면, 길고복잡하긴 하지만 해시값으로 volume의 이름이 매겨져 있고, `c4c1fed...` 로 시작하는 volume이 ```local``` 드라이버를 사용하여 만들어져 있음을 알 수 있다. 
mysql 컨테이너가 접근중이 volume에 무언가 적기 위해 아래 처럼 mysql database를 하나 만들고 나가자.

```
C:\Users\jhlee>docker exec -it testdb mysql --password=mirero
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 8.0.13 MySQL Community Server - GPL

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

mysql> create database mirero;
Query OK, 1 row affected (0.08 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mirero             |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

mysql> exit
Bye
```

이제 testdb  컨테이너에는 mirero 라는 이름의 database가 만들어져 있는 상태이다. 여기서 testdb 컨테이너를 종료하고 다시 시작해 보자.

```
C:\Users\jhlee>docker container stop testdb
testdb

C:\Users\jhlee>docker container start testdb
testdb

C:\Users\jhlee>docker exec -it testdb mysql --password=mirero
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.13 MySQL Community Server - GPL

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mirero             |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.01 sec)

mysql> exit
Bye

C:\Users\jhlee>
```

보다시피. 재기동 해도 여전히 mirero 로는 database가 남아있다. 
그러면 아예 컨테이너를 지우고 다시 만드는 경우에는 어떻게 될까?

```
C:\Users\jhlee>docker container stop testdb
testdb

C:\Users\jhlee>docker container rm testdb
testdb

C:\Users\jhlee>docker run --name testdb -e MYSQL_ROOT_PASSWORD=mirero -d mysql
8546698b8189acee4eb65d795d5feb72fe239b72332d40559ff84c9d66ba423f

C:\Users\jhlee>docker inspect -f "{{.Mounts}}" testdb
[{volume 49a2a82be479740c7f9eeff0d4fadc0a7e7715c0dc40e5ee50039603c3571778 /mnt/sda1/var/lib/docker/volumes/49a2a82be479740c7f9eeff0d4fadc0a7e7715c0dc40e5ee50039603c3571778/_data /var/lib/mysql local  true }]

C:\Users\jhlee>docker volume ls
DRIVER              VOLUME NAME
local               49a2a82be479740c7f9eeff0d4fadc0a7e7715c0dc40e5ee50039603c3571778
local               c4c1fedc28f594882ee6bdbe86bcd754a82dca181d5954d5dc0b03940992ef44
local               d42b71c63042a2e771ed6bc9c59ba896d729950432202a3020c9982954e9eb04
local               d5afd7f8670201495f6246551623e4ce8c5deba00b0b634b878b3def11283cf1
local               portainer_data
```

맨 처음 만들었던 컨테이너에서 volume 값이 `c4c1fed...`  였는데,
 지금은 어떤가? `49a2a82b...` 로 바뀌어 있다. 앞서 만들어졌던 `c4c1fed...` volume 도 여전히 있고, 
 새로운  `49a2a82b...` volume이 생성되어 있는 상태이다.

이 상태로는 mysql 을 실행해서 보면 아까 만들었던 mirero 라는 이름의 db가 없다.

```
C:\Users\jhlee>docker exec -it testdb mysql --password=mirero
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.13 MySQL Community Server - GPL

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.01 sec)

mysql>
```

이렇게 docker가 암시적으로 만든 volume은 container가 `docker stop` 된 다음 `docker rm`, 즉 제거될때  함께 제거된다는 특징도 있다. 

```
docker stop testdb 
```

한 다음, 

```
docker volume ls 
```

해 보면 앞서 생성되었던 volume이 제거 되었음을 알 수 있다. 

# Docker Volume의 명시적 생성

container 의 생성/소멸과 별도로 항상 영구적인 데이터를 저장하기 위해서는 container 생성시 암시적으로 생성되도록 하지 않고, 우리가 명시적으로 volume 을 만들어 사용한다. 
이렇게 생성된 volume은 container 가 제거된 다음에도 계속 남아 있다. 


이러한 hash값 이름은 해당 volume이 docker에 의해서 자동생성되거나 이름을 주지 않고, 명시적으로 `docker volume create` 명령어로 volume을 만들때 사용된다. 
만일 사용자가 명시적으로 volume에 이름을 붙이고 싶다면, 맨 마지막에 이름을 전달하면 된다. 

```

C:\Users\jhlee>docker volume create testdb-volume
testdb-volume

C:\Users\jhlee>docker volume ls
DRIVER              VOLUME NAME
local               testdb-volume

```

그리고, 이렇게 생성된 *이름있는* volume은 다음과 같이 실행하여, docker image에 정의된 volume의 mount directory에 연결할 수 있다. 



```
c:\Users\jhlee
λ docker run --rm -d --name testdb -e MYSQL_ROOT_PASSWORD=mirero -v testdb-volume:/var/lib/mysql mysql
3fd2a5beb9a2de76fcfbae72b70bc57818f5c8bdec99635ae9680eca23565429
```

여기 잘 보면, `--rm` 옵션은 container 가 stop 될 때, 제거하겠다는 의미이며, 위와 같이 `-v` 옵션을 주지 않은 경우에 생성되는 암시적 volume은 함께 제거된다. 
하지만, 우리는 여기서 `-v` 옵션을 통해, 명시적으로 만든 volume을 `/var/lib/mysql` (이 경로는 `Dockerfile` 의 `VOLUME` 부분에 정의된 경로이다.)의 경로에 mapping하게 하였으므로, container가 제거되더라도 이 volume은 그냥 docker host내에 유지된다. 

```
c:\Users\jhlee
λ docker exec -it testdb mysql --password=mirero
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.13 MySQL Community Server - GPL

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

mysql> create database mirero;
Query OK, 1 row affected (0.02 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mirero             |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

mysql> exit
Bye
```

이제 이 `testdb` container를 중지하면, 완전히 container가 제거되지만, volume은 여전히 남아있음을 다음에서 알 수 있다.

```
c:\Users\jhlee
λ docker stop testdb
testdb

c:\Users\jhlee
λ docker container ls --all
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES

c:\Users\jhlee
λ docker volume ls
DRIVER              VOLUME NAME
local               testdb-volume
```

이제 container 를 다시 만들어 보자. 

```
c:\Users\jhlee
λ docker run --rm -d --name testdb -e MYSQL_ROOT_PASSWORD=mirero -v testdb-volume:/var/lib/mysql mysql
d2eb6126ad97a17392fd1eeafd412f0386e6314bf039e054aaf44e3a248cf633

c:\Users\jhlee
λ docker exec -it testdb mysql --password=mirero
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.13 MySQL Community Server - GPL

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mirero             |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.01 sec)

mysql> exit
Bye

c:\Users\jhlee
```

위에서 알 수 있듯이, testdb-volume에 저장되어 있던 mirero라는 database는 여전히 살아 있음을 알 수 있다. 

-------------------

# Volume 에 대한 추가 설명 

## Volume 에 있는 파일 내용을 Browse하는 방법 

  docker의 volume에 있는 파일 내용을 보고 싶다면, 해당 volume을 임의 컨테이너에 mount해서 보면 된다. [여기](https://github.com/moby/moby/issues/31417) 참조 

  요약하면, (busybox)[https://hub.docker.com/_/busybox/] 같은 간단한 docker image를 사용해서 해당 volume을 특정 경로에 mount하고, `sh` 등 shell을 실행해서, 
  `ls` 해보면 알 수 있다는 얘기임.

  ```
  
c:\Users\jhlee
λ docker run --rm -it -v testdb-volume:/volume busybox sh
/ # cd /volume/
/volume # ls -al
total 166924
drwxr-x---    2 999      999           4096 Nov 20 05:35 #innodb_temp
drwxr-xr-x    7 999      999           4096 Nov 20 05:35 .
drwxr-xr-x    1 root     root          4096 Nov 20 05:36 ..
-rw-r-----    1 999      999             56 Nov 20 05:05 auto.cnf
-rw-r-----    1 999      999        3071654 Nov 20 05:05 binlog.000001
-rw-r-----    1 999      999            365 Nov 20 05:07 binlog.000002
-rw-r-----    1 999      999            178 Nov 20 05:35 binlog.000003
-rw-r-----    1 999      999             48 Nov 20 05:10 binlog.index
-rw-------    1 999      999           1676 Nov 20 05:05 ca-key.pem
-rw-r--r--    1 999      999           1112 Nov 20 05:05 ca.pem
-rw-r--r--    1 999      999           1112 Nov 20 05:05 client-cert.pem
-rw-------    1 999      999           1680 Nov 20 05:05 client-key.pem
-rw-r-----    1 999      999           3397 Nov 20 05:35 ib_buffer_pool
-rw-r-----    1 999      999       50331648 Nov 20 05:35 ib_logfile0
-rw-r-----    1 999      999       50331648 Nov 20 05:05 ib_logfile1
-rw-r-----    1 999      999       12582912 Nov 20 05:35 ibdata1
drwxr-x---    2 999      999           4096 Nov 20 05:06 mirero
drwxr-x---    2 999      999           4096 Nov 20 05:05 mysql
-rw-r-----    1 999      999       31457280 Nov 20 05:10 mysql.ibd
drwxr-x---    2 999      999           4096 Nov 20 05:05 performance_schema
-rw-------    1 999      999           1680 Nov 20 05:05 private_key.pem
-rw-r--r--    1 999      999            452 Nov 20 05:05 public_key.pem
-rw-r--r--    1 999      999           1112 Nov 20 05:05 server-cert.pem
-rw-------    1 999      999           1676 Nov 20 05:05 server-key.pem
drwxr-x---    2 999      999           4096 Nov 20 05:05 sys
-rw-r-----    1 999      999       12582912 Nov 20 05:35 undo_001
-rw-r-----    1 999      999       10485760 Nov 20 05:35 undo_002
/volume # exit
  ```

이러한 특징을 잘 활용하면, 특정 docker volume의 파일내역을 압축해서 다른 사람과 공유하는것도 가능하다. 

## 특정 Container에 Mount되어 있는 Volume을 확인

   `docker inspect {container id 혹은 이름}` 명령어를 치면 "Mounts" 항목에 해당 내용이 나와 있음.
   아래의 예를 보면, `testdb-volume` 이라는 이름의 docker volume이 `/var/lib/mysql` 이라는 경로로 mount되어 있음을 알 수 있다. 

   ```
   .
   .
   .
        },                                                                                          
        "Mounts": [                                                                                 
            {                                                                                       
                "Type": "volume",                                                                   
                "Name": "testdb-volume",                                                            
                "Source": "/mnt/sda1/var/lib/docker/volumes/testdb-volume/_data",                   
                "Destination": "/var/lib/mysql",                                                    
                "Driver": "local",                                                                  
                "Mode": "z",                                                                        
                "RW": true,                                                                         
                "Propagation": ""                                                                   
            }                                                                                       
        ],                                                                                          
    .
    .
    .
   ```

## 혹시 MYSQL에서 File Access 관련 오류가 날때 대처법

어떻게 하면 이런문제가 생겼는지 알 수 없지만, 이 문서를 작성하는 과정중에 어찌하다보니, 아래와 같은 오류가 난경우가 있었다. 

```
[Note] Basedir set to /usr/
[Warning] The syntax '--symbolic-links/-s' is deprecated and will be removed in a future release
[Warning] 'NO_ZERO_DATE', 'NO_ZERO_IN_DATE' and 'ERROR_FOR_DIVISION_BY_ZERO' sql modes should be used with strict mode. They will be merged with strict mode in a future release.
[Warning] Setting lower_case_table_names=2 because file system for /var/lib/mysql/ is case insensitive
[Warning] You need to use --log-bin to make --log-slave-updates work.
libnuma: Warning: /sys not mounted or invalid. Assuming one node: No such file or directory mbind: Operation not permitted
[ERROR] InnoDB: Operating system error number 22 in a file operation.
[ERROR] InnoDB: Error number 22 means 'Invalid argument'
[ERROR] InnoDB: File ./ib_logfile101: 'aio write' returned OS error 122. Cannot continue operation
[ERROR] InnoDB: Cannot continue operation.
```

이런 경우에는 mysql의 docker image가 기본적으로 수행하는 mysqld 데몬 서비스에 `--innodb_use_native_aio=0` 옵션을 주면 해결이 되었다.

```
docker run --name mirero-mysql -v /c/Users/docker/testmysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=mirero -it mysql --innodb_use_native_aio=0
```

이것과 관련된 [SO의 Q&A](https://stackoverflow.com/a/48631919/884268) 를 참고
