---
title: "Redis 싱글 구성과 영속성 (RDB, AOF)"
date: 2026-09-12 21:00:00 +0900
categories: [Backend, Redis]
tags: [Redis, persistence, RDB, AOF]
---

### **싱글 구성**

Redis 프로세스 하나로 운영하는 가장 단순한 구성

![](/assets/img/posts/2026-09-12-redis-single-persistence-01.png)

<br>

### **영속성이 필요한 이유**

Redis는 모든 저장과 조회를 메모리에서 처리

- Redis는 데이터를 메모리에만 두기 때문에, 프로세스가 죽으면 데이터가 사라진다.
- 디스크에 데이터를 남겨두면 재시작할 때 다시 로드할 수 있다. 그래서 영속성(RDB, AOF)이 필요하다.

<br>

### **RDB (Redis Database)**

특정 시점의 메모리 전체를 스냅샷으로 디스크에 기록하는 방식

```
save [초] [변경 횟수]
```

- 예를 들어 `save 900 1` 은 마지막 스냅샷 이후 900초가 지났고, 그동안 변경이 1번 이상 있었으면 스냅샷을 기록한다.
- 자식 프로세스를 fork 해서 기록하므로, 기록 중에도 요청을 계속 처리할 수 있다.
- 파일 하나로 관리되고 로드가 빨라서, 백업과 복구에 유리하다.
- 마지막 스냅샷 이후의 변경은 유실된다.
- 데이터가 크면 fork 자체가 지연을 만들고, 기록 중의 변경(Copy-on-Write)만큼 메모리가 추가로 든다.

<br>

### **AOF (Append Only File)**

수신한 write 명령을 파일에 순서대로 기록하는 방식 (default 비활성)

- 재시작 시 이 명령들을 처음부터 다시 실행해서 복구한다.
- write 명령은 먼저 OS 버퍼에 기록되고, fsync가 호출되어야 디스크에 반영된다. appendfsync는 이 fsync 시점을 정한다.

```
appendonly yes
appendfsync [always | everysec | no]
```

- always : 명령을 처리할 때마다 fsync 한다. 유실이 거의 없지만 느리다.
- everysec : 1초마다 fsync 한다. 최대 1초 유실될 수 있다. (default)
- no : fsync를 OS에 맡긴다. 빠르지만 유실 범위가 OS에 달려 있다. (리눅스 기준 약 30초)
- 파일이 계속 커지므로, rewrite로 현재 데이터를 기준으로 파일을 새로 작성해 크기를 줄인다.

<br>

### **RDB와 AOF 선택**

|  | RDB | AOF |
|---|---|---|
| 기록 단위 | 시점 스냅샷 | write 명령 로그 |
| 유실 범위 | 마지막 스냅샷 이후 (분 단위) | 최대 1초 (everysec) |
| 재시작 로드 | 빠름 | 느림 |

- 둘 다 켜서 함께 쓸 수 있고, 이 경우 재시작할 때 AOF를 우선 로드한다.
- rewrite된 AOF는 앞부분이 RDB 포맷으로 기록되어(aof-use-rdb-preamble, default yes), 로드 속도의 차이가 줄어든다.

<br>

### **장애 발생과 대응**

![](/assets/img/posts/2026-09-12-redis-single-persistence-02.png)

Redis 프로세스가 죽으면 복구 필요

1. 모니터링/알람으로 장애를 인지한다.
2. 원인을 확인하고 프로세스를 재기동한다. (재기동 자체는 systemd, Docker 등으로 자동화할 수 있다)
3. 재시작 과정에서 RDB/AOF가 로드되어 데이터가 복원된다.

RDB/AOF 파일은 Redis와 같은 호스트의 디스크에 있다.

**복구될 때까지 서비스는 중단되고, 디스크까지 잃으면 복구할 데이터 자체가 사라진다.**

<br>

> RDB와 AOF를 이용하면 프로세스가 죽어도 데이터를 복구할 수 있다.  
> 하지만 그 파일이 담긴 디스크까지 잃는다면 어떻게 해야 할까?  
> 다음 편에서는 다른 호스트에 실시간으로 데이터를 이중화해 이 한계를 해결하는 [레플리카](/posts/redis-replication/)를 다룬다.
{: .prompt-tip }

<br>

> **Redis 아키텍처 시리즈**
>
> 1. **싱글 구성과 영속성 (RDB, AOF)**
> 2. [레플리카 (Replication)](/posts/redis-replication/)
> 3. [센티널 (Sentinel)](/posts/redis-sentinel/)
> 4. [클러스터 (Cluster)](/posts/redis-cluster/)
{: .prompt-info }
