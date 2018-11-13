---
layout: post
title:  "Windows 에서 Docker VirtualBox VM Disk File위치 변경방법"
date:   2018-11-13 01:41:47 -0600
tags: docker virtualbox dev
---

Windows 7 에서 Docker Host를 VirtualBox 로 만들때, 종종 VM Disk File 을 별도의 하드디스크에 저장하고 싶을때가 있다. 
(C:를 SSD로 사용시 용량부족 등). 

두가지 방법 이 가능 

## .docker 디렉토리 symbolic link

```mklink "%USERPROFILE%\.docker" D:\Docker``` 이런식으로 하는 방법([소스](https://stackoverflow.com/a/45492583/884268))


## Docker가 만든 VM Disk 파일 자체를 옮기는 방법 

 이  [블로그 포스트](https://forgetfulprogrammer.wordpress.com/2016/10/28/how-to-move-your-large-virtualbox-vm-disk-created-by-docker/) 를 참고.

