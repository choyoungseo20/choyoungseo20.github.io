---
title: "[Docker] Docker 개념"
date: 2025-04-09 21:41:33 +0900
categories: [Back-End, DevOps]
tags: []
---

### **Docker**

리눅스 **컨테이너** 기반의 오픈소스 **가상화** 플랫폼

#### **가상화 종류**

- 머신 가상화 (하이퍼바이저)

![](/assets/img/posts/2025-04-09-docker-docker-개념-01.png)

- OS 수준 가상화 (리눅스 컨테이너)

![](/assets/img/posts/2025-04-09-docker-docker-개념-02.png)

- 개발 환경 가상화 (파이썬 가상 환경)

![](/assets/img/posts/2025-04-09-docker-docker-개념-03.jpg)

#### **Docker를 사용하는 이유**

**격리성** : 각 프로그램을 OS 커널을 공유하며 독립적인 환경에서 실행할 수 있는 특성

**이식성** : 특정 프로그램을 다른 곳으로 쉽게 옮겨서 설치 및 실행할 수 있는 특성

#### **Container**

독립적인 실행 환경 (미니 컴퓨터)

![](/assets/img/posts/2025-04-09-docker-docker-개념-04.png)

#### **Container의 독립성**

- 파일 시스템

- 라이브러리 및 의존성

- 네트워크

- 환경변수

- 프로세스

#### **Image**

프로그램을 실행하는데 필요한 내용 (닌텐도 칩)
