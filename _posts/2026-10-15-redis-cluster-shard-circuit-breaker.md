---
title: "샤드별 서킷 브레이커 (Redis Cluster 부분 장애)"
description: Redis 샤드 1개 장애에 정상 샤드 요청까지 DB로 돌리던 전역 서킷을 샤드 단위로 나눠, 정상 샤드 DB 폴백을 94~97%에서 0%로 줄인 실험 기록
date: 2026-10-15 21:00:00 +0900
categories: [Backend, Redis]
tags: [ETF Orca, Redis, Cluster, circuit-breaker, Resilience4j, Lettuce, fault-isolation]
---

### **들어가며**

ETF Orca 앱의 ETF 전망 투표(BUY·WAIT·SELL)는 집계를 Redis에, 원본을 PostgreSQL에 둔다.

- 앞선 Redis 센티널 구성 실험에서, Redis 장애가 투표와 무관한 API까지 번지는 문제를 즉시 실패와 서킷 브레이커로 막았다.
- 그 서킷은 Redis 전체를 하나의 단위로 보고 실패율을 셌다.
- 데이터를 샤드 3개에 나눠 담는 Redis Cluster에서 샤드 1개만 고장 나면, 이 서킷이 여전히 맞게 판단하는지 검증했다.

결론부터 말하면 맞지 않았다. 인기 종목이 어느 샤드에 있느냐에 따라, 정상 샤드 요청까지 DB로 돌리거나 장애 샤드를 끝내 차단하지 못했다.

**서킷의 단위를 샤드에 맞추자 정상 샤드 조회의 DB 폴백이 94~97%에서 0%로 줄었다.**

<br>

### **배경 : 투표 집계와 1차 대응**

투표 원본은 DB가 강제하고, Redis는 원본에서 언제든 다시 만들 수 있는 파생 집계

![](/assets/img/posts/2026-10-15-redis-cluster-shard-circuit-breaker-01.png)

- 저장 : DB에 투표를 upsert하고 커밋한 뒤, Lua 스크립트로 Redis 집계를 갱신한다. Redis 갱신이 실패해도 응답은 성공으로 두고 로그만 남긴다.
- 조회 : Redis에서 집계를 읽고, 실패하면 DB에서 직접 집계해 응답한다.
- 재조정 : 시작 직후, 5분 주기, Lettuce 재연결 시점에 DB 기준으로 Redis 집계를 통째로 덮어쓴다.

센티널 실험에서 넣은 장치는 두 가지였다.

- 즉시 실패 : Lettuce의 disconnectedBehavior를 REJECT_COMMANDS로 바꾸고 명령 타임아웃을 500ms로 줄였다. 연결이 끊긴 동안 명령을 큐에 쌓아 두며 요청 스레드를 붙잡지 않게 하기 위해서다.
- 서킷 브레이커 : Resilience4j 서킷 하나(`redis`)가 저장과 조회의 Redis 호출을 감쌌다.

```yaml
resilience4j:
  circuitbreaker:
    instances:
      redis:
        sliding-window-size: 20
        minimum-number-of-calls: 10
        wait-duration-in-open-state: 3s
        failure-rate-threshold: ${VOTE_CIRCUIT_FAILURE_RATE:50}
```

- 최근 20건 중 실패가 50% 이상이면 서킷이 열리고, 3초 동안 Redis를 부르지 않고 바로 폴백한다.

**센티널 구성에서는 Redis가 한 덩어리로 죽기 때문에, 서킷 하나로 충분했다.**

<br>

### **실험 설계 : 샤드 하나만 고장 내기**

같은 부하에서 서킷 설정만 바꿔 가며, 샤드 0에만 장애를 주입

![](/assets/img/posts/2026-10-15-redis-cluster-shard-circuit-breaker-02.png)

- 환경 : Docker Desktop 단일 호스트의 Docker Compose. Redis Cluster 6노드(마스터 3·복제본 3), PostgreSQL 16, HikariCP 풀 10
- 부하 : k6로 투표, 집계 조회, Redis와 무관한 정적 페이지 요청을 각 50rps로 3분 동안 보냈다.
- 종목 배치 : Redis와 같은 CRC16으로 슬롯을 계산해 샤드마다 종목 10개를 두었다. 인기 종목 하나가 트래픽의 50%를 받고, 나머지는 다른 종목에 고르게 나뉜다.
- 장애 주입 : 부하 시작 60초 후 샤드 0의 마스터와 복제본에 장애를 넣고, 주입 후 20초 구간을 측정했다.
  - kill : 컨테이너 강제 종료(SIGKILL)
  - pause : 컨테이너 일시 정지. 프로세스는 살아 있지만 응답하지 않는 hang 상태
  - partition : 컨테이너의 네트워크 분리
- `cluster-require-full-coverage`는 no로 두었다. default yes에서는 슬롯 하나만 비어도 모든 노드가 CLUSTERDOWN을 응답해, 부분 장애 자체를 관찰할 수 없다.

| 시나리오 | 명령 타임아웃 | 연결 끊김 시 동작 | 서킷 | 서킷 범위 |
|---|---|---|---|---|
| C1 default | 60s | 큐에 쌓기 | 무력화 | 전역 |
| C2 즉시 실패 | 500ms | 즉시 거부 | 무력화 | 전역 |
| C3 전역 서킷 | 500ms | 즉시 거부 | 실패율 50% | 전역 |
| C4 샤드 서킷 | 500ms | 즉시 거부 | 실패율 50% | 샤드별 |

- 서킷 무력화는 실패율 임계치를 100%로 둔 설정이다. 창 20건이 전부 실패해야 열린다.
- 조합마다 3번씩 실행했고, 수치는 실행 간 범위로 적었다.

**센티널 실험의 대응(C3)이 부분 장애에서도 통하는지가 질문이었다.**

<br>

### **발견 : 같은 장애, 다른 실패율**

전역 서킷이 세는 실패율은 장애 크기가 아니라 인기 종목의 위치가 결정

![](/assets/img/posts/2026-10-15-redis-cluster-shard-circuit-breaker-03.png)

샤드 0이 고장 나면 실패하는 요청의 비율은 인기 종목이 어디에 있느냐에 따라 갈린다.

- 인기 종목이 장애 샤드에 있을 때 : 인기 종목 50%에 나머지 50% 중 약 1/3이 더해져, 전체 요청의 약 67%가 실패한다.
- 인기 종목이 정상 샤드에 있을 때 : 나머지 50% 중 약 1/3만 실패해, 약 17%가 실패한다.

전역 서킷(C3, kill)은 두 경우 모두 틀린 판단을 내렸다.

| 인기 종목 위치 | 전역 실패율 | 서킷 | 결과 |
|---|---|---|---|
| 장애 샤드 | 약 67% | 열림 | 정상 샤드 조회의 94.4~96.5%가 DB로 폴백 |
| 정상 샤드 | 약 17% | 3번 모두 열리지 않음 | 장애 샤드 요청이 계속 Redis로 가서 실패 |

- 67%인 경우 : 서킷이 열리는 순간 Redis 호출 전체가 차단된다. 멀쩡한 샤드 2개의 조회까지 DB로 몰려, 조회 폴백이 초당 48.9~49.3건에 달했다.
- 17%인 경우 : 정상 샤드는 지켜졌지만, 장애 샤드로 가는 요청은 서킷의 보호를 받지 못했다.

**샤드 1개의 장애가 전역 서킷 안에서는 Redis 전체의 장애처럼 보이거나, 아예 보이지 않았다.**

<br>

### **임계치를 바꾸면 되지 않을까**

임계치 조정은 두 경우 중 하나를 고치면 다른 하나를 틀리게 하는 선택

- 같은 장애인데 전역 실패율은 17%에서 67%까지 오간다. 이 범위의 어디에 임계치를 두어도 한 경우는 틀린다.
  - 임계치를 67% 위로 올리면, 인기 종목이 장애 샤드에 있어도 서킷이 열리지 않는다.
  - 임계치를 17% 아래로 내리면 두 경우 모두 열리지만, 전역 서킷이 열리는 순간 정상 샤드 요청도 함께 차단된다.
- 서킷이 보는 범위를 바꾸는 다른 방법도 검토했다.
  - 노드 ID 단위 서킷 : 장애 조치로 복제본이 승격되면 노드 ID가 바뀌어, 그동안 쌓은 서킷 상태를 잃는다.
  - 기동 시 한 번 만든 슬롯 표 : 슬롯이 다른 샤드로 옮겨지면 실패를 엉뚱한 샤드에 합산한다.

**문제는 임계치가 아니라 실패를 세는 단위였다. 장애가 샤드 단위로 나므로 서킷도 샤드 단위여야 했다.**

<br>

### **서킷 단위를 샤드에 맞추기**

요청 키가 속한 샤드를 찾아, 샤드마다 다른 서킷을 사용

![](/assets/img/posts/2026-10-15-redis-cluster-shard-circuit-breaker-04.png)

```java
public CircuitBreaker of(String etfCode) {
    if (client == null) {
        return registry.circuitBreaker("redis");
    }
    int slot = SlotHash.getSlot("vote:{" + etfCode + "}:count");
    var partitions = client.getPartitions();
    RedisClusterNode master = partitions.isEmpty() ? null : partitions.getMasterBySlot(slot);
    if (master == null || master.getSlots().isEmpty()) {
        return registry.circuitBreaker("redis");
    }
    return registry.circuitBreaker("redis-shard-" + Collections.min(master.getSlots()),
            registry.circuitBreaker("redis").getCircuitBreakerConfig());
}
```

- 슬롯 계산 : 투표 키는 `vote:{종목코드}:count` 형태다. 중괄호 안의 종목 코드만 해싱되므로, 한 종목의 키는 모두 같은 슬롯에 있다.
- 샤드 찾기 : Lettuce 클러스터 클라이언트가 이미 들고 있는 슬롯 배치표(Partitions)에서 그 슬롯을 가진 마스터를 찾는다. 별도 저장소나 조회 없이 클라이언트의 라우팅 정보를 그대로 쓴다.
- 서킷 이름 : 그 마스터가 가진 슬롯 중 가장 작은 번호를 쓴다. 장애 조치로 노드가 바뀌어도 슬롯 범위는 그대로라서 서킷 상태가 이어진다.
- 설정 : 샤드 서킷은 기존 `redis` 서킷의 설정을 그대로 복사한다. 판단 기준은 그대로 두고 범위만 바꿨다.
- 배치표가 비어 있거나 슬롯의 주인을 찾지 못하면, 기존 전역 서킷으로 돌아간다.

**바뀐 것은 실패를 세는 범위 하나였다. 임계치와 창 크기는 C3와 같다.**

<br>

### **결과**

정상 샤드는 Redis에 그대로 남고, 장애 샤드의 요청만 빠르게 DB로 폴백

![](/assets/img/posts/2026-10-15-redis-cluster-shard-circuit-breaker-05.png)

| 지표 | 비교 | 전 | 후 |
|---|---|---|---|
| 정상 샤드 조회 DB 폴백 (kill) | C3 → C4 | 94.4~96.5% | 0% |
| 조회 폴백 초당 건수 (kill) | C3 → C4 | 48.9~49.3 | 32.7~32.9 |
| 장애 샤드 조회 중 타임아웃까지 기다린 비율 (pause·partition) | C2 → C4 | 99.2~99.8% | 7.0% |
| 장애 샤드 조회 대기 중앙값 (pause·partition) | C2 → C4 | 504ms | 3ms |

- 정상 샤드 보호 : C4는 kill, pause, partition 9번의 실행 모두에서 정상 샤드 조회 폴백이 0%였다. 남은 폴백 초당 33건은 전부 장애 샤드 몫이다.
- 장애 샤드 차단 : pause와 partition에서 `redis-shard-0` 서킷만 주입 0.8초 후에 열렸다.
  - 서킷이 없으면(C2) hang 상태의 샤드로 간 요청은 명령 타임아웃 500ms를 다 채우고 실패한다. 그동안 요청 스레드를 쥐고 있다.
  - 샤드 서킷이 열린 뒤에는 기다리지 않고 바로 DB로 폴백했다. 남은 7%는 서킷이 회복을 확인하려 보내는 시험 호출이다.
- 복제본 승격 : 마스터만 종료해 복제본을 승격시킨 별도 실행(이전 MySQL 환경과 이전 코드, 10번)에서, 승격까지 걸린 7~9초 동안 정상 샤드 조회 폴백은 0%였다.

**kill에서는 정상 샤드를 지켰고, hang과 분단에서는 장애 샤드의 대기 시간까지 줄었다.**

<br>

### **한계와 남은 질문**

검증한 범위와 검증하지 않은 범위

- 모든 수치는 단일 호스트 Docker Compose 실험이다. 50rps는 3분 안에 장애 전파를 관찰하기 위한 값이고, 운영 환경의 처리량이나 가용성을 보장하지 않는다.
- 리샤딩 : 서킷은 요청 시점의 슬롯 배치표로 샤드를 찾도록 설계했다. 하지만 리샤딩 실험과 테스트는 하지 않았고, 옮겨지기 전 이름의 서킷을 정리하는 처리도 없다.
- 복제본 승격 구간은 이전 MySQL 환경에서만 측정했다. PostgreSQL 환경에서는 측정하지 않았다.
- 재조정은 여전히 전체 종목을 대상으로 돈다. 서킷이 열렸던 샤드의 종목만 재조정하는 방식은 구현하지 않았다.
- 샤드가 늘어나면 서킷도 그만큼 늘어난다. 샤드 수가 커질 때의 관리 방식은 아직 다루지 않았다.

<br>

### **마치며**

처음 질문은 센티널 구성에서 만든 서킷이 Cluster에서도 통하는가였다.

- C3에서도 서킷은 열렸다. 정상 샤드 조회가 어디로 갔는지 나눠 세어 보니 틀린 판단이었다.
- 장애는 샤드 단위로 나는데 서킷은 Redis 전체를 단위로 실패를 세고 있었다.

> 부분 장애를 격리하려면, 실패를 판단하는 범위를 실제 장애 범위에 맞춰야 한다.
{: .prompt-tip }

- 코드와 실험 스크립트 : [alphaeveryday/edge app-api](https://github.com/alphaeveryday/edge/tree/app-api-2026-10-04/src/apps/cloud/app-api)
- 클러스터 동작 정리 : [Redis 클러스터 (Cluster)](/posts/redis-cluster/)
- 센티널 동작 정리 : [Redis 센티널 (Sentinel)](/posts/redis-sentinel/)
