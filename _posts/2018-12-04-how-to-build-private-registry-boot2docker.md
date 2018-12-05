---
layout: post
title:  "Docker Private Registry를 docker toolbox상에서 운용"
date:   2018-12-04 16:43:01 -0600
tags: docker dev registry virtualbox 
---

# Docker Registry를 개인PC에서 운용하는 방법

docker registry는 이미 dockerize되어서 여기저기에서 쓰이는 것 같다. 통상은 web api만 제공하는데, 다음 github 프로젝트가 web ui와 함께 실행할 수 있도록 
잘 꾸며져 있어서 이걸로 구축하기로 하였다.

  - [Example how to configure Docker Registry with 3rd party UI](https://github.com/slydeveloper/docker-registry-joxit-ui-compose)

위 레퍼지터리를 clone 한다음, 해당 디렉토리로 이동하여 

```
docker-compose up
```

명령어만 실행하면 된다. ........지만,  나는 windows7에서 docker toolbox를 쓰고 있어서 volume 설정에 약간 문제가 있다(윽.)

## Volume 설정 손보기 

이것은 모든 파일시스템이 docker host상에 노출되는 docker-for-windows나 docker-for-mac 과는 달리, docker toolbox로 설치한 docker-machine환경은 virtualbox내 docker host가 운용되기 때문이다. 
따라서, 영속적인 reigstry설정내역(즉, 사용자가 docker push한 내역)이 docker container의 실행/중지 뒤에도 언제든 다시 실행하면 그대로 남아있게 하기 위해, 
volume mounting이 반드시 되어야 한다. 통상 docker실행하는 파일시스템을 그냥 쓰는게 편하므로 일단 이걸 기준으로 설명하겠다.

레퍼지터리내 `docker-compose.yml` 을 수정해야 한다. 이 파일을 보면, 크게 web api인 `registry-srv`와 web ui인 `registry-ui` 의 2개 컨테이너가 실행되도록 구성되어 있는데, 
실제 docker registry의 저장을 담당하는 `registry-srv` 의 volume설정을 보자.

```
     volumes:
      - ./registry/storage:/var/lib/registry
      - ./registry/config.yml:/etc/docker/registry/config.yml:ro
```

위에 보면 volumes 항목이 

  - `/var/lib/registry` --> 현재디렉토리(그니깐 `docker-compose.yml`이 있는 디렉토리)밑의 `registry/storage` 라는 폴더로 마운트.
  - `/etc/docker/registry/config.yml` --> 현재디렉토리 밑의 `registry/config.yml` 파일로 마운트.

win7에서 docker toolbox로 설치된 docker-machine 환경은 실제로 아무 디렉토리나 volume mount용으로 쓸 수 없고, 반드시 자신의 개인 폴더(통상 `c:\Users\{사용자명}`임)이 docker host의 `/c/Users/{사용자명}` 으로 mount되어 있다. 따라서 docker 입장에서는 무슨 폴더든 `c:\Users\{사용자명}` 하위 디렉토리만 볼 수 있다. 
따라서 

  - `c:\Users\{사용자명}\docker-registry\storage` 디렉토리를 추가
  - 현재 레퍼리지터리의 `registry/config.yml` 파일을 `c:\Users\{사용자명}\docker-registry`에 복사

한 다음, `docker-compose.yml` 파일을 다음과 같이 수정한다.

```
    volumes:
      - /c/Users/${USERNAME}/docker-registry/storage:/var/lib/registry
      - /c/Users/${USERNAME}/docker-registry/config.yml:/etc/docker/registry/config.yml:ro
```

## Network 손보기 

운용할 docker registry를 다른 컴퓨터에서 접근할 수 있게 하려면, docker host가 운용되는 virtual box의 NAT 설정에서 포트포워딩을 해야 한다.

![VirtualBox내 docker host인 'default'의 네트워크 설정변경](/img/2018-12-04-virtualbox-설정화면.png){:class="img-responsive"}

중요한것은 호스트IP와 게스트포트이다. 호스트IP는 virtualbox을 실행하는 컴퓨터의 ip 주소를 적어주고, 8080 과 5000포트 2개를 반드시 맵핑시켜놓는다.
8080포트는 web ui용이고, 5000포트는 web api(web ui에서 접근한다)용이다. 

## Docker Registry의 HTTPS 비활성화

Docker registry의 web api는 보안때문에 Default로 https로 접근할 수 밖에 없게 되어 있는것 같다. 이것 때문에, 그냥 실행하면 docker toolbox의 경우 여러 곤란한 일이 생긴다(-,.-)

이문제를 해결하려면, (여기)[http://developmentalmadness.com/2016/03/09/docker-configure-insecure-registry-in-docker toolbox/]에서와 같이 아예 제대로 https를 쓰도록 
하던지, 아니면, (여기)[https://github.com/docker/machine/issues/3433]에 기술된 데로 다음과 같은 내용의 파일을 docker host내 `/etc/docker/daemon.json`에
저장한다.

```json
{ "insecure-registries" : ["docker-machine:5000", "192.168.100.115:5000"] }
```

위에서 명시적인 ip는 virtualbox나 container의 ip가 아니라, docker-machine이 운용되는(즉, virtualbox를 실행하는..) 서버자체의 ip주소값이다. 5000번 포트는 docker registry api가 동작하는 포트이다. 
맨 마지막의 이 명시적인 ip가 없으면 서버외부에서 docker registry에 접근하지 못한다.

위의 설정을 추가한다음에는 반드시 docker-machine restart 해준다.

## 새로만든 registry에 image를 push해보기. .


다음의 명령을 수행해 본다. 여기서 나온 ip 주소는 docker-machine virtualbox을 실행하는 서버의 주소다.

```sh
>docker pull busybox
...
>docker image tag busybox 192.168.100.115:5000/hello-world
...
>docker push 192.168.100.115:5000/hello-world
The push refers to repository [192.168.100.115:5000/hello-world]
428c97da766c: Pushed
latest: digest: sha256:1a6fd470b9ce10849be79e99529a88371dff60c60aab424c077007f6979b4812 size: 524
```

## web ui 를 접속해 본다. 

`192.168.100.115:8080` 으로 접속하면 web ui를 볼 수 있다. 

![web ui 화면](/img/firefox_2018-12-05_11-28-19.png){:class="img-responsive"}


## 참고
 
 - [Docker Toolbox - Github](https://github.com/docker/toolbox)
 - [Docker Registry+UI 구축예제 프로젝트 - Github]:(https://github.com/slydeveloper/docker-registry-joxit-ui-compose)
 - [virtualbox 네트웍 설정](https://www.jhipster.tech/tips/020_tip_using_docker_containers_as_localhost_on_mac_and_windows.html)
 - [Docker Insecure Registry - Docker문서](https://docs.docker.com/registry/insecure)

