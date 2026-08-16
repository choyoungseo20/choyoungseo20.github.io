---
title: "[Docker] AWS ECR"
date: 2025-04-10 01:21:08 +0900
categories: [Back-End, DevOps]
tags: []
---

### **ECR (Elastic Container Registry)**

Dockerhub와 같은 역할을 하는 AWS의 서비스

### **ECR에 사용법 (이미지 push & pull)**

**1. EC2 ubuntu에 Docker & Docker Compose 설치**

```
sudo apt-get update && \
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common && \
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add - && \
sudo apt-key fingerprint 0EBFCD88 && \
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" && \
sudo apt-get update && \
sudo apt-get install -y docker-ce && \
sudo usermod -aG docker ubuntu && \
newgrp docker && \
sudo curl -L "https://github.com/docker/compose/releases/download/2.27.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose && \
sudo chmod +x /usr/local/bin/docker-compose && \
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

버전 확인

```
docker -v 
docker compose version
```

**2. AWS  CLI 설치**

[윈도우 (Local)]

<https://awscli.amazonaws.com/AWSCLIV2.msi>

[우분투 (EC2)]

```
sudo apt install unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

**3. IAM 사용자 생성**

1. IAM 사용자 추가
2. 사용자 이름 설정
3. 직접 정책 연결 선택
4. 'AmazonEC2ContainerRegistryFullAccess' 권한 부여
5. 생성
6. 액세스 키 만들기
7. AWS 외부에서 실행되는 애플리케이션 선택
8. 액세스 키 발급

**4. 액세스 키 등록 (윈도우 & 우분투)**

```
aws configure
AWS Access Key ID [None]: <발급 받은 Access Key id>
AWS Secret Access Key [None]: <발급 받은 Secret Access Key>
Default region name [None]: ap-northeast-2
Default output format [None]:
```

**5. ECR 생성**

![](/assets/img/posts/2025-04-10-docker-aws-ecr-01.png)

**6. ECR push 명령어 (윈도우 - Local)**

![](/assets/img/posts/2025-04-10-docker-aws-ecr-02.png)

```
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin 664418967804.dkr.ecr.ap-northeast-2.amazonaws.com
docker build -t bodycheck-server .
docker tag bodycheck-server:latest 664418967804.dkr.ecr.ap-northeast-2.amazonaws.com/bodycheck-server:latest
docker push 664418967804.dkr.ecr.ap-northeast-2.amazonaws.com/bodycheck-server:latest
```

- 이미지를 푸시하는 명령어로 Dockerfile이 존재하는 디렉토리에서 수행해야 한다.

**7. ECR pull 명령어 (우분투 - EC2)**

![](/assets/img/posts/2025-04-10-docker-aws-ecr-03.png)

```
docker pull 664418967804.dkr.ecr.ap-northeast-2.amazonaws.com/bodycheck-server
```

- URI로 pull 명령어를 수행한다.

**8. pull 받은 이미지를 이용하여 컨테이너 실행**

1. docker run으로 이미지 실행
2. compose.yml 파일을 생성하여 다른 이미지와 함께 실행
