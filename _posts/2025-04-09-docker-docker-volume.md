---
title: "[Docker] Docker Volume"
date: 2025-04-09 23:45:03 +0900
categories: [Back-End, DevOps]
tags: []
---

### **Docker Volume**

도커 컨테이너에서 데이터를 **영속적**으로 저장하기 위한 방법

-> 호스트 컴퓨터의 저장 공간을 공유해서 사용한다.

```
docker run -v [호스트 디렉토리 절대경로]:[컨테이너 디렉토리 절대경로] [이미지명]
```

1. 호스트 디렉토리 절대경로에 이미 디렉토리가 존재할 경우

![](/assets/img/posts/2025-04-09-docker-docker-volume-01.png)

2. 호스트 디렉토리 절대경로에 디렉토리가 존재하지 않을 경우

![](/assets/img/posts/2025-04-09-docker-docker-volume-02.png)

### **DBMS 실행**

MySQL, PostgreSQL, MongoDB 등의 DBMS를 실행시킬 때는 환경 변수가 필요하다.

따라서 도커 허브에서 컨테이너 디렉토리 절대경로와 필요한 환경 변수를 확인해야 한다.

<https://hub.docker.com/>

[Docker Hub Container Image Library | App Containerization

Increase your reach and adoption on Docker Hub With a Docker Verified Publisher subscription, you'll increase trust, boost discoverability, get exclusive data insights, and much more.

hub.docker.com](https://hub.docker.com/)

MySQL 예시

```
docker run -e MYSQL_ROOT_PASSWORD=password123 -p 3306:3306 -v [호스트의 디렉토리 절대경로]:/var/lib/mysql -d mysql
```

- -e 옵션은 환경 변수를 설정할 수 있다.
- 볼륨에 환경 변수 값이 저장되기에, 컨테이너를 삭제했다가 다시 설치하며 비밀번호 등의 환경 변수를 수정할 경우 접속이 되지 않는다.
