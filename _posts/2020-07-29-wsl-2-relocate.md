---
layout: post
title:  "WSL2 의 저장공간을 다른 디스크로 옮기는 법"
date:  2020-02-06 10:57:35 +0900
tags: wsl2, docker
---

Windows 10 - 버젼 2004 에서 새로이 추가된 WSL 2 를 쓰면 docker 등 작업을 하는데 있어서 훨씬 일관되고 좋다는 얘기를 듣고(https://www.docker.com/blog/docker-desktop-wsl-2-best-practices), 써보기로 마음먹고, https://www.youtube.com/watch?v=hwbbFY4Yww0&t=1018s 에 있는 동영상 강좌를 보고 설치를 마쳤다. 

그런데, 문제가 되는것은 SSD 인 C:\ 드라이브 용량이 부족 -,.- 

```
Run powershell.exe as Administrator
PS C:\WINDOWS\system32> wsl -l
Windows Subsystem for Linux Distributions:
Ubuntu (Default)

# mkdir S:\ISOs\

PS C:\WINDOWS\system32> wsl --export Ubuntu S:\ISOs\ubuntu-wsl.tar

# mkdir w:\VMs

PS C:\WINDOWS\system32> cd w:\VMs
PS W:\VMs> mkdir ubuntu-wsl
PS W:\VMs> wsl --unregister Ubuntu
Unregistering...
PS W:\VMs> wsl --import Ubuntu W:\VMs\ubuntu-wsl S:\ISOs\ubuntu-wsl.tar
PS W:\VMs> wsl -l
Windows Subsystem for Linux Distributions:
Ubuntu (Default)
```

위의 내용은 [WSL Github 이슈에 올라온 글](https://github.com/MicrosoftDocs/WSL/issues/412#issuecomment-575923176) 의 내용이다. 물론, 위 내용에서 `Ubuntu` 라는 부분은 반드시 `wsl -l -v` 명령으로 현재 설치된 Ubuntu 가 어떤 명칭으로 설치되었는지에 따라 다르다. 예를들어 Ubuntu 20.04 LTS 가 설치되었다면, `Ubuntu-20.04` 라고 명명된 Distro가 설치된다. 따라서 이름을 잘 보고 해야 한다. 

그 다음 중요한것은, 위와 같이 한다음 해당 Distro 에 접속을 하면, `root` 계정으로 들어가진다. 특정 사용자 계정을 쓰는것이 바람직할 텐데, 이를 위해서는 `ubuntu config --default-user <some-user>` 명령을 cmd 창에서 입력하라는 검색결과가 나왔지만, 이것은 항상 통하는게 아니다. 이를테면 저 `ubuntu` 라는 명령은 특정 distro에 대한 설치 패키지인듯 하다. 

해결책은 https://github.com/microsoft/WSL/issues/3974#issuecomment-576782860 에서 얻었다. 

바로 `/etc/wsl.conf` 파일을 아래의 내용으로 하나 만들어 둔다음 `wsl --shutdown <Distro명>` 하여 가상머신을 내린 후 다시 올리는 방법이다.

```
[user]
default=username
```

