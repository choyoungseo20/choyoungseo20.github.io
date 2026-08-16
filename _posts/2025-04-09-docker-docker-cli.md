---
title: "[Docker] Docker CLI"
date: 2025-04-09 22:19:57 +0900
categories: [Back-End, DevOps]
tags: []
---

#### **Docker에서 Docker Hub의 이미지를 컨테이너에 띄우기 위한 세 단계**

1. 이미지 가져오기

```
docker pull nginx
```

- docker hub의 nginx 이미지를 가져오는 명령어
- '이미지:tag'의 형태로 되어 있으며 tag를 명시하지 않을 경우 latest 버전을 가져온다.

2. 컨테이너 생성하기

```
docker create nginx
```

- 해당 이미지로 컨테이너를 생성만 하는 명령어

3. 컨테이너 실행하기

```
docker start [컨테이너 ID or 컨테이너명]
```

- 올려놓은 컨테이너를 실행하는 명령어

\* 위의 세 단계는 아래의 명령어로 한 번에 수행할 수 있다.

```
docker run nginx
```

#### **docker run 명령어의 option**

컨테이너의 경우 Host 컴퓨터 내의 미니 컴퓨터이다. 따라서 네트워크가 독립적으로 존재하는데, 컨테이너에서 실행시킨 프로세스에 접근하려면 해당 port를 Host 컴퓨터의 port와 연결해주어야 한다.

![](/assets/img/posts/2025-04-09-docker-docker-cli-01.png)

이는 아래의 명령어로 설정 가능하다.

```
docker run -p 80:80 nginx
```

- -p 옵션 호스트 컴퓨터 내부에서 실행되는 컨테이너의 포트를 호스트 컴퓨터와 연결하는 작업이다.
- -p [호스트 포트]:[컨테이너 포트]

백그라운드로 실행은 아래의 명령어로 가능하다.

```
docker run -d nginx
```

- -d 옵션으로 백그라운드 실행을 할 수 있다.

컨테이너에 이름도 부여할 수 있다.

```
docker run --name [컨테이너명] nginx
```

#### **Docker 이미지 조회, 삭제**

```
조회
docker image ls

삭제
docker image rm [이미지 ID or 이미지명] 
docker image rm -f [이미지 ID or 이미지명]
```

- 리눅스와 비슷하게 ls가 조회이다.
- rm의 경우 해당 이미지를 사용하는 컨테이너가 없을 경우 사용하여 삭제할 수 있다.
- -f 옵션을 사용하면 해당 이미지를 사용하는 컨테이너가 중지된 컨테이너일 경우에도 삭제할 수 있다.

\* 이미지를 사용하는 컨테이너에는 세 가지의 상태(컨테이너 X, 중지된 컨테이너, 실행중인 컨테이너) 존재

\*\* 실행 중인 컨테이너에서 사용하고 있는 이미지는 삭제 불가

\*\*\* 이미지 삭제 시 ID 중 일부 (앞의 3~4글자)만 입력해도 삭제 가능

\*\*\*\* 여러 이미지를 한 번에 삭제 가능

```
docker image rm $(docker images -q)
docker image rm -f $(docker images -q)
```

- docker images -q 명령어는 시스템에 있는 모든 이미지의 ID를 반환하는 기능

#### **Docker 컨테이너 조회, 중지, 삭제**

```
조회 
docker ps
docker ps -a 

중지
docker stop 
docker kill 

삭제
docker rm [컨테이너 ID or 컨테이너명]
docker rm - f [컨테이너 ID or 컨테이너명] 
docker rm $(docker ps -qa)
docker rm -f $(docker ps -qa)
```

- ps로 실행중인 컨테이너를 확인할 수 있고, -a 옵션으로 중지된 컨테이너도 포함하여 확인할 수 있다.
- stop으로 정상 중지를 할 수 있고, stop으로 중지가 되지 않을 경우 kill로 강제 중지 할 수 있다.
- rm의 경우 중지된 컨테이너일 경우에만 사용 가능하고, -f 옵션으로 실행중인 컨테이너도 삭제할 수 있다.
- docker ps -qa 명령어로 시스템에 있는 모든 컨테이너 ID를 반환하여 여러 컨테이너를 한 번에 삭제할 수 있다.

\* 컨테이너 삭제 시 ID 중 일부 (앞의 3~4글자)만 입력해도 삭제 가능

#### **Docker 컨테이너 로그**

컨테이너를 포그라운드로 실행하면 로그를 볼 수 있지만, 다른 작업을 할 수 없다.

따라서 백그라운드로 실행하고, 필요할 때 로그를 확인한다.

```
docker logs [컨테이너 ID or 컨테이너명]
docker logs --tail 10 [컨테이너 ID 또는 컨테이너명] 
docker logs -f [컨테이너 ID or 컨테이너명]
docker logs --tail 0 -f [컨테이너 ID or 컨테이너명]
```

- logs로 특정 컨테이너의 전체 로그를 확인할 수 있다.
- --tail n 옵션으로 마지막 n개 줄의 로그를 확인할 수 있다.
- -f 옵션으로 전체 로그 + 실시간 로그를 조회할 수 있다.
- 위 두 옵션을 이용하면 실시간 로그만 조회할 수 있다.

#### **Docker 컨테이너 접속**

```
docker exec -it [컨테이너 ID or 컨테이너명] bash
```

- -it 옵션은 명령어를 입력하고 결과를 확인하기 위함이다.
