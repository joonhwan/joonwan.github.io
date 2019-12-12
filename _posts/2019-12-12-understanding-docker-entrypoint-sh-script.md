---
layout: post
title:  "Docker의 entrypoint 에서 사용되는 sh 스크립트 이해"
date:   2019-12-12 18:42:02 -0600
tags: docker 
---

(minio 도커 이미지)[https://hub.docker.com/r/minio/minio/] 를 다음과 같이 실행하면 

```
# docker run minio/minio                                                                                                              
NAME:                                                                                                                                 
  minio - Cloud Storage Server.                                                                                                       
                                                                                                                                      
DESCRIPTION:                                                                                                                          
  MinIO is an Amazon S3 compatible object storage server. Use it to store photos, videos, VMs, containers, log files, or any blob of d
ata as objects.                                                                                                                       
                                                                                                                                      
USAGE:                                                                                                                                
  minio [FLAGS] COMMAND [ARGS...]                                                                                                     
                                                                                                                                      
COMMANDS:                                                                                                                             
  server   start object storage server                                                                                                
  gateway  start object storage gateway                                                                                               
  version  print version                                                                                                              
                                                                                                                                      
FLAGS:                                                                                                                                
  --config-dir value, -C value  [DEPRECATED] path to legacy configuration directory (default: "/root/.minio")                         
  --certs-dir value, -S value   path to certs directory (default: "/root/.minio/certs")                                               
  --quiet                       disable startup information                                                                           
  --anonymous                   hide sensitive information from logging                                                               
  --json                        output server logs and startup information in json format                                             
  --compat                      trade off performance for S3 compatibility                                                            
  --help, -h                    show help                                                                                             
                                                                                                                                      
VERSION:                                                                                                                              
  2019-10-12T01:39:57Z                                                                                                                
                                                                                                                                      
joonhwan.lee@DESKTOP-LU5A3MQ C:\WorkSpace\prj\oss\mine\joonhwan.github.io                                                             
#                                                                                                                                     
```

보다시피, `minio` 명령어를 아무 인자도 없이 실행하면 나오는 화면이 출력된다. 그럼 `minio`의 `Dockerfile`은 단순히 `CMD` 만 다음처럼 정의되어 있을 거라고 생각했는데, [실제 파일](https://github.com/minio/minio/blob/master/Dockerfile)을 열어보니  다음 처럼 되어 있었다. 

```yml
.
.
. 
ENTRYPOINT ["/usr/bin/docker-entrypoint.sh"]
.
.
.
CMD ["minio"]
.
.

```
위는 결국, `docker run minio/minio` 했을때, `/usr/bin/docker-entrypoint.sh minio` 를 실행하는 것과 같단다([`ENTRYPOINT`와 `CMD`의 상호작용 이하해기 - Docker공식문서](https://docs.docker.com/engine/reference/builder/#understand-how-cmd-and-entrypoint-interact) 참고)

이제, 이 (`docker-entrypoint.sh`스크립트)[https://github.com/minio/minio/blob/master/dockerscripts/docker-entrypoint.sh] 를 이해하고 싶어졌다. 

중요한 부분이 다음과 같다. 

```sh
# 중략 

# If command starts with an option, prepend minio.
if [ "${1}" != "minio" ]; then
    if [ -n "${1}" ]; then
        set -- minio "$@"
    fi
fi

# 중략

# su-exec to requested user, if service cannot run exec will fail.
docker_switch_user() {
    if [ ! -z "${MINIO_USERNAME}" ] && [ ! -z "${MINIO_GROUPNAME}" ]; then
        addgroup -S "$MINIO_GROUPNAME" >/dev/null 2>&1 && \
            adduser -S -G "$MINIO_GROUPNAME" "$MINIO_USERNAME" >/dev/null 2>&1

        exec su-exec "${MINIO_USERNAME}:${MINIO_GROUPNAME}" "$@"
    else
        # fallback
        exec "$@"
    fi
}

# 중략 


## Switch to user if applicable.
docker_switch_user "$@"
```

위에서 `set -- minio "$@"` 부분이 좀 모호한데, [What does “set --” do in this Dockerfile entrypoint? - SO](https://unix.stackexchange.com/a/308263)의 답변글을 보니, 이해가 좀 갔다. 결국 이 쉘 스크립트에서 인자로 넘어온 것들중 맨 처음것이 `minio` 가 아니면, `minio` 를 맨 앞에 붙인다음, `exec "$@"` 으로 실행(결국, `minio 어쩌구 저쩌구` 식)하는 것이다. 이 때문에, `docker run minio/minio server --help` 와 같이 실행하면, `minio server --help` 를 실행한 것 과 같은 효과가 난다. 

이런식으로 하면, `ENTRYPOINT`에 지정된 스크립트가 사용자가 요청한 명령을 수행하기 전에 무언가를 수행할 수 있도록 할 수 있어 이런 구성이 자주 쓰인단다(예를 들어 minio 를 위한 환경변수값들의 설정, 사용자 권한 설정 등...)

```
joonhwan.lee@DESKTOP-LU5A3MQ C:\WorkSpace\prj\oss\mine\joonhwan.github.io
# docker run minio/minio server --help
NAME:
  minio server - start object storage server

USAGE:
  minio server [FLAGS] DIR1 [DIR2..]
  minio server [FLAGS] DIR{1...64}

DIR:
  DIR points to a directory on a filesystem. When you want to combine
  multiple drives into a single large system, pass one directory per
  filesystem separated by space. You may also use a '...' convention
  to abbreviate the directory arguments. Remote directories in a
  distributed setup are encoded as HTTP(s) URIs.

```


