---
layout: post
title:  "docker를 Windows 7에서 다시 시작함."
date:   2018-11-09 01:41:47 -0600
tags: docker dev
---

# Windows 7 에서 Docker 설치환경

회사에서 준 컴이 Windows-7 이라서, 최신 docker환경의 설치가 안되었는데,
우연히 [choco](https://chocolatey.org/)를 이용해서 편리하게 Windows 7 에서 환경을 구축하는 방법이
 있는 [블로그 문서](https://stefanscherer.github.io/yes-you-can-docker-on-windows-7/)를 찾게 되어 시도하여 꽤 괜찮은 결과를 얻었음.

  준비물 

   - Windows 7
   - VirtualBox
   - choco

# 설치 로그

admin cmd 환경에서 다음을 이용함.

```
choco install -y docker
choco install -y docker-machine
choco install -y docker-machine-vmwareworkstation
```

[블로그 문서](https://stefanscherer.github.io/yes-you-can-docker-on-windows-7/) 에는 vmware를 사용하려고 했는데, 나는 일단, vmware가 없어서, 
virtualbox를 사용함. 사실, ```docker-machine-vmwareworkstation``` 은 필요가 없는 거 같음.

위에꺼가 다 설치되었으면, 

```
docker-machine --native-ssh create -d virtualbox default
```

로 설치한다. ```-d``` 옵션은 driver 옵션으로 어떤 가상화 드라이버를 사용할 것인지를 결정하고, ```--native-ssh``` 는 내장된 go 기반의 ssh를 그냥 쓰겠다는 의미.

이걸 하고 나서 virtual box 를 실행해 보면, "default"라는 이름의 인스턴스가 추가된것을 알 수 있다. 

이제 전원을 넣자 

```
docker-machine start default 
```

이걸 한 다음 virtual box 화면을 보면 ```default``` 가상머신의 동작중으로 나온다. 이제 docker를 쓸 수 있다. ..... 다만,...

linux/windows 10 처럼 docker host가 운영체제 자체에 면밀히? 연동되는 경우와 달리 windows 7 은 docker host의 역할을 virtual box가 하고 있다. 따라서 docker에게 이 host에 대해 알려주어야 한다.

```docker``` 명령어로 이 default에 접근하도록 설정하려면, 사용자의 shell환경에서 ```docker-machine env default``` 혹은 ```docker-machine env``` 명령어를 쳐서 나오는 설명을 잘 따라하면 된다. 
예를들어 다음과 같다. 

```
C:\Users\jhlee
λ docker-machine env
SET DOCKER_TLS_VERIFY=1
SET DOCKER_HOST=tcp://192.168.99.100:2376
SET DOCKER_CERT_PATH=C:\Users\jhlee\.docker\machine\machines\default
SET DOCKER_MACHINE_NAME=default
SET COMPOSE_CONVERT_WINDOWS_PATHS=true
REM Run this command to configure your shell:
REM     @FOR /f "tokens=*" %i IN ('"C:\ProgramData\chocolatey\lib\docker-machine\bin\docker-machine.exe" env') DO @%i
```

이것은 ```docker-machine``` 으로 하여금 현재 shell환경에서 docker가 특정 docker host를 찾아갈 수 있도록 설정하거다. 
위 ```SET ...``` 으로 시작하는 부분을 배치파일로 만들어서 실행해도 되고, 아니면, ```Run this command ...``` 라고 나오는 부분 다음행의 내용(이 경우 ```REM```은 cmd 쉘 환경에서의 주석문이므로, ```REM``` 바로 다음 부분)을 복사해서 실행해 주면 된다). 즉, 

```
    @FOR /f "tokens=*" %i IN ('"C:\ProgramData\chocolatey\lib\docker-machine\bin\docker-machine.exe" env') DO @%i
```

을 실행해면 된다.  이제 docker를 사용할 수 있는지 확인해 보자.


```
C:\Users\jhlee
λ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

```docker ps```는 docker host에서 실행중인 docker container의 목록을 보여준다. 이게 위와 같이 결과를 뿌리면, docker는 사용준비가 된것. 

여기서 부터는 docker의 실제 사용과 관련된 부분임. 


# docker에 대한 설명을 읽을 때 유념해야 할 상황(windows 7에서 docker 배우기시 빨간주머니/파란주머니...)

## docker container는 127.0.0.1 로 접근할 수 없다

  ```docker-machine ip``` 를 쳐서 나오는 ip 주소가 실제 docker의 ip주소가 된다.  
  따라서 특정 docker image를 실행했을때 0.0.0.0 등과 같이 localhost 역할 을 하는 주소로 
  접속하라는 메시지는 모두 이 ip주소로 바꾸어서 생각해야 한다.

### docker container에 로컬파일시스템의 특정 경로를 마운트시 유의점 

 docker를 배우다 보면  db file 등 영속적인 데이터를 유지 하기 위해 docker container에 로컬파일시스템의 특정 디렉토리를 마운팅 할 일이 있는데,
 windows 7 에서의 "로컬파일시스템"이라 함은, virtual box의 default 가상머신에 존재하는 파일 경로이다. 따라서 로컬 시스템을 마운팅 하기위해서는 
 docker host역할을 하는 virtual box의 default 가상머신에 먼저 로컬파일시스템의 디렉토리를 마운트한 다음, 그걸 다시 docker에 마운트 해야 한다. 

