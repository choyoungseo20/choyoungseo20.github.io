---
title: "[Docker] Docker Compose"
date: 2025-04-10 00:54:40 +0900
categories: [Back-End, DevOps]
tags: []
---

### **Docker Compose**

**여러 개의 Docker 컨테이너**들을 하나의 서비스로 정의하고 구성하여 관리할 수 있게 도와주는 툴

-> 여러 개의 컨테이너로 이루어진 복잡한 애플리케이션을 관리하는데 용이하다.

-> 복잡한 명령어로 실행시키던 것을 간소화 시킬 수 있다.

#### **Docker Compose 명령어**

```
compose.yml에서 정의한 컨테이너 실행
docker compose up -d

compose.yml에서 정의한 컨테이너 조회
docker compose ps -a

로그 조회
docker compose logs

컨테이너 실행 시 이미지 재빌드
docker compose up -d --build

이미지 다운 및 업데이트
docker compose pull

컨테이너 종료
docker compose down
```

- docker compose up은 이미지가 없을 때만 빌드하고, --build 옵션을 붙이면 이미지의 존재 여부와 상관 없이 빌드를 해서 컨테이너를 실행시킨다.
- docker compose pull의 경우 Dockerhub이나 ECR(Elastic Containerr Registry)에서 새로운 이미지를 받아올 때 사용한다.

#### **compose.yml**

Dockerfile과 같은 디렉토리에 생성하여 위치 시킨다.

![](/assets/img/posts/2025-04-10-docker-docker-compose-01.png)

**하늘색 박스 : 서비스 이름 (원하는 이름으로 설정)**

**파란색 박스 : 컨테이너 이름 (원하는 이름으로 설정)**

**초록색 박스 : Dockerfile의 경로를 작성한 후 실행시켜 이미지 사용**

**연두색 박스 : Dockerhub의 이미지 사용**

**빨간색 박스 : Host 컴퓨터의 포트와 컨테이너의 포트를 매핑 (필수 X)**

**연보라 박스 : Docker Volume 설정**

**보라색 박스 : depend\_on의 경우 특정 서비스들을 의존하기에, 특정 서비스의 condition이 healthy(정상 작동 상태) 해야 실행**

**보라색 박스 : healthcheck의 경우 서비스가 잘 작동하는지 test를 하여 확인 (각 서비스의 healthcheck 명령어 존재, interval은 healthcheck test 간격, retries는 최대 시도 횟수)**
