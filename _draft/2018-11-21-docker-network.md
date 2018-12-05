---
layout: post
title:  "Docker Network 기본"
date: 2018-11-21  09:58:27 -0600
tags: docker dev
---

# Docker Container의 Network Mode

일단, 통상 Docker를 공부하는 사람의 입장에서 보면.. [Networking with standalone containers](https://docs.docker.com/network/network-tutorial-standalone/) 문서를 
읽어보고 한번 따라 해보는게 좋을 것 같다. 요약하면, 

     - Bridge모드(기본값) : Docker Host에 설치된 docker0 인터페이스에 물려서 Docker Container <-> Docker Host 간 통신 및 다른 Container와의 통신이 중계된다
     - Host 모드 : `-h` 옵션을 넣어 실행된 Container는 Docker Host의 eth0 등 인터페이스를 그냥 사용한다("network isolation"을 해제하였다..고 한단다). Container에서 Docker Host 바깥세상에 접속시 편한거 같다.

그리고 위 문서에 기록된 내용중 다음과 같은 표현이 있다.

   > Remember, the default bridge network is not recommended for production. To learn about user-defined bridge networks, continue to the next tutorial.

Bridge모드는 필드에서 사용하는것은 권장되지 않는다고... 

Host 모드를 사용하면, 해당 docker container가 마치, docker host내에서 직접 실행된 프로그램처럼 docker host의 네트워크에 물린다.

