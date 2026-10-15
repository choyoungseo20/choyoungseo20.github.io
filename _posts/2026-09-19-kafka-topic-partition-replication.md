---
title: "Kafka 토픽과 파티션 (Replication, ISR)"
date: 2026-09-19 00:10:00 +0900
categories: [Backend, Kafka]
tags: [TransCourse, Kafka, KRaft, partition, replication, ISR]
---

### **토픽과 파티션 구성**

하나의 토픽이 파티션으로 나뉘고, 각 파티션이 RF만큼의 replica로 복제되어 Broker에 배치되는 구성

- 토픽 하나는 파티션 여러 개로 나뉘고, 각 파티션은 Broker의 디스크에 추가 전용 로그로 저장된다. 오프셋은 파티션 안에서만 유일하며, 파티션 사이에는 순서 관계가 없다.
- 파티션 로그는 일정 크기마다 세그먼트 파일로 나뉜다. 보존 기간이나 용량을 넘긴 세그먼트는 파일 단위로 삭제된다.
- 파티션마다 RF(Replication Factor)만큼 replica가 서로 다른 Broker에 배치된다. 그중 하나가 leader이고 나머지는 follower다.
- 파티션은 늘릴 수는 있지만 줄일 수는 없다. 파티션을 늘리면 키 기반 파티션 배정이 바뀌어 같은 키의 메시지가 다른 파티션으로 가기 시작한다.

**토픽은 이름이고, 실제로 저장되고 복제되고 할당되는 단위는 파티션이다.**

<br>

### **토픽 생성 과정**

토픽 생성 요청 → Controller의 replica 배치 → 메타데이터 전파 → Broker의 로그 디렉터리 생성 순으로 진행

1. 클라이언트나 관리 도구가 파티션 수와 RF를 지정해 토픽 생성을 요청한다. 요청은 어느 Broker로 가든 Active Controller에 전달된다.
2. Controller는 파티션마다 RF개의 Broker를 골라 replica를 배치한다. 같은 파티션의 replica는 서로 다른 Broker에 두고, leader가 Broker에 고르게 퍼지도록 배치한다. Rack 정보가 있으면 서로 다른 Rack에 나눈다.
3. 배치 결과와 각 파티션의 첫 leader가 메타데이터 로그에 커밋되어 모든 Broker에 전파된다.
4. replica를 맡은 Broker는 log.dirs 아래에 토픽-파티션 이름의 디렉터리를 만들고, follower는 leader에서 fetch를 시작한다.
5. 자동 생성(auto.create.topics.enable)이 켜져 있으면 없는 토픽에 메시지를 보낼 때 num.partitions와 default.replication.factor로 토픽이 만들어진다. 운영에서는 끄고 명시적으로 만든다.

**파티션 수와 RF는 생성 시점에 정해지고, 배치는 Controller가 결정한다.**

<br>

### **저장과 조회**

저장은 leader 로그의 끝에 추가, 조회는 High Watermark까지만 허용

- 프로듀서가 보낸 메시지는 leader의 로그 끝에 추가되어 오프셋을 받는다. 다음 메시지가 받을 오프셋(마지막 오프셋 + 1)을 LEO(Log End Offset)라 한다.
- follower는 leader에 fetch 요청을 보내 새 메시지를 받아 자기 로그에 추가한다. 복제는 leader가 밀어 주는 것이 아니라 follower가 끌어가는 방식이다.
- HW(High Watermark)는 ISR의 모든 replica가 기록을 마친 오프셋이다. leader는 follower의 fetch 요청에 실린 오프셋으로 각 follower의 진행 상황을 알고 HW를 올린다.
- 프로듀서의 acks 설정이 응답 시점을 정한다. acks=1은 leader가 기록한 직후, acks=all은 ISR 전체가 기록한 뒤 응답한다. 그래서 acks=all의 응답을 받은 메시지는 그 시점에 ISR 전체에 있다.
- 컨슈머는 HW까지만 조회할 수 있다. 공식 문서는 이를 커밋된 메시지라 부르며, ISR 전체에 복제되고 ISR 크기가 min.insync.replicas 이상일 때만 컨슈머에게 보인다. HW를 넘긴 메시지는 leader 장애 시 사라질 수 있기 때문이다.
- 저장과 조회 모두 leader에서만 처리된다. follower는 복제만 하며, Broker에 replica.selector.class를 지정하면 컨슈머가 같은 Rack의 follower에서 조회하는 것은 예외다. default는 leader에서만 조회한다.

**HW는 ISR 전체가 가진 지점이고, 컨슈머에게 보이는 메시지는 그 지점까지다.**

<br>

### **ISR**

leader를 따라잡고 있는 replica의 집합

- ISR(In-Sync Replica)에는 leader 자신과, Controller와 세션을 유지하면서 leader의 LEO를 replica.lag.time.max.ms(default 30초) 안에 따라잡고 있는 follower가 들어간다.
- follower가 이 시간 동안 leader의 끝을 따라잡지 못하면 leader가 ISR에서 제외한다. 다시 따라잡으면 ISR에 복귀한다.
- ISR 변경은 leader가 Controller에 알리고, Controller가 메타데이터에 반영한다.
- ISR의 크기가 저장의 안전성을 정한다. ISR이 leader 하나뿐이면 acks=all로 보내도 leader 한 대에만 기록된 채 응답이 나간다.
- min.insync.replicas는 acks=all 저장이 성공하기 위한 ISR의 최소 크기다. ISR이 이 값보다 작으면 leader는 저장을 거부하고 NotEnoughReplicas 오류를 돌려준다.

| 설정 조합 | 한 대 장애 시 저장 | 유실 가능성 |
|---|---|---|
| RF 3, min.insync.replicas 1 | 계속된다 | ISR이 1로 줄면 leader 한 대에만 남는다 |
| RF 3, min.insync.replicas 2 | 계속된다 | 항상 2대 이상에 기록된 뒤 응답한다 |
| RF 3, min.insync.replicas 3 | 거부된다 | 유실은 없지만 한 대만 장애가 나도 저장이 멈춘다 |

**RF는 복제본의 수이고, 실제 안전성은 ISR 크기와 min.insync.replicas가 정한다.**

<br>

### **장애 판정**

follower 장애는 leader의 lag 판정으로, leader 장애는 Controller의 Broker 판정으로 결정

- follower가 replica.lag.time.max.ms 동안 leader의 끝을 따라잡지 못하면 leader가 ISR에서 뺀다. Broker가 살아 있어도 디스크나 네트워크가 느리면 같은 판정을 받는다.
- leader가 있는 Broker의 heartbeat가 끊겨 Controller가 fenced로 판정하면, 그 Broker의 모든 leader 파티션이 failover 대상이 된다.
- ISR이 min.insync.replicas 아래로 줄면 저장이 거부된다. 이것은 장애 판정이 아니라 유실을 막기 위한 저장 차단이고, ISR이 회복되면 자동으로 풀린다.

**follower는 leader가, leader는 Controller가 판정한다.**

<br>

### **failover**

Controller가 ISR 중에서 새 leader를 선정하고, follower는 leader epoch 기준으로 로그를 맞춤

1. Controller는 장애가 난 leader의 파티션마다 ISR에 남은 replica 중 하나를 새 leader로 정한다. ISR 안의 replica는 모두 HW까지 기록을 마쳤으므로, 컨슈머에게 보였던 메시지는 유실되지 않는다.
2. HW 이후의 메시지는 새 leader가 가진 만큼만 남는다. acks=1로 보낸 메시지는 leader에만 기록됐을 수 있어 이 구간에서 사라질 수 있다.
3. 나머지 follower는 새 leader의 로그와 자기 로그를 비교해 새 leader에 없는 부분을 잘라내고(truncate) 다시 fetch를 시작한다.
4. ISR에 남은 replica가 없으면, Kafka 4.1부터 새 클러스터에 기본으로 켜지는 ELR(Eligible Leader Replicas) 중에서 leader를 고른다. ELR도 없으면 파티션은 leader 없이 멈춘다. unclean.leader.election.enable을 켜면 ISR 밖의 replica가 leader가 되지만, 그 replica에 없던 메시지는 사라진다.
5. 장애가 났던 Broker가 복구되면 follower로 합류해 따라잡고, ISR에 복귀한다. auto.leader.rebalance.enable(default true)이 켜져 있으면 Controller가 주기적으로 원래 leader(preferred leader)에게 leader 자리를 되돌린다.

**ISR 안에서 leader를 뽑으면 HW까지는 보존되고, HW 이후의 유실 여부는 acks 설정이 정한다.**

<br>

### **관련 설정**

토픽 단위 설정과 Broker default. RF는 설정 키가 아니라 토픽 생성 시 `--replication-factor`로 지정한다.

```
min.insync.replicas=2
retention.ms=604800000
segment.bytes=1073741824
cleanup.policy=delete
```

- RF(--replication-factor) : 파티션당 replica 수다. 생성 시 지정하며, 지정하지 않으면 Broker의 default.replication.factor(default 1)를 따른다.
- num.partitions : 파티션 수를 지정하지 않은 토픽에 적용되는 Broker default다. (default 1)
- auto.create.topics.enable : 없는 토픽에 메시지를 보낼 때 자동으로 만들지 정한다. (default true) 운영에서는 끄고 명시적으로 만든다.
- min.insync.replicas : acks=all 저장이 성공하기 위한 ISR 최소 크기다. (default 1)
- replica.lag.time.max.ms : follower가 이 시간 동안 leader를 따라잡지 못하면 ISR에서 제외된다. (default 30초)
- retention.ms : 메시지 보존 기간이다. (default 7일) retention.bytes로 파티션당 용량 상한도 둘 수 있다. (default -1, 무제한)
- segment.bytes : 세그먼트 파일 하나의 크기다. (default 1GB) 보존 정책은 세그먼트 단위로 적용된다.
- cleanup.policy : delete는 기간·용량 초과분을 삭제하고, compact는 키별 최신 메시지만 남긴다. (default delete)
- unclean.leader.election.enable : ISR 밖의 replica도 leader가 될 수 있게 할지 정한다. (default false)

<br>

> 파티션의 저장과 복제, ISR과 HW가 유실을 막는 범위를 정리했다.  
> 그렇다면 프로듀서는 메시지를 어느 파티션으로 보내고, 응답을 언제 받으며, 재시도 중 중복은 어떻게 막을까?  
> 다음 편에서는 전송 경로의 반대편인 [프로듀서](/posts/kafka-producer/)를 다룬다.
{: .prompt-tip }

<br>

> **Kafka 아키텍처 시리즈**
>
> 1. [클러스터 구성 (KRaft, Controller, Broker)](/posts/kafka-cluster-kraft/)
> 2. **토픽과 파티션 (Replication, ISR)**
> 3. [프로듀서 (acks, 파티셔너, 멱등성)](/posts/kafka-producer/)
> 4. [컨슈머 그룹과 오프셋 (Group Coordinator, Rebalancing)](/posts/kafka-consumer-group/)
> 5. [적용 : 컨슈머 증설과 리밸런싱 비용 (TransCourse)](/posts/kafka-rebalance-duplicate-encoding/)
{: .prompt-info }
