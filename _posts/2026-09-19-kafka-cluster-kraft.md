---
title: "Kafka 클러스터 구성 (KRaft, Controller, Broker)"
date: 2026-09-19 00:00:00 +0900
categories: [Backend, Kafka]
tags: [TransCourse, Kafka, KRaft, controller, broker, replication]
---

### **Kafka의 기본 모델**

프로듀서가 토픽에 메시지를 추가하고, 컨슈머 그룹이 파티션을 나눠 맡아 오프셋 순서로 가져가는 로그 구조

- 토픽은 메시지의 논리적 이름이고, 실제 저장 단위는 토픽을 나눈 파티션이다. 파티션은 끝에만 추가되는 로그(append-only log)이며, 메시지는 파티션 안에서 0부터 증가하는 오프셋으로 식별된다.
- 메시지는 컨슈머가 읽어 가도 지워지지 않고, 보존 기간(retention)이 지나야 삭제된다. 꺼내면 사라지는 큐와 다른 점이다.
- 컨슈머는 Broker가 밀어 주는 것이 아니라 스스로 가져가고(fetch), 어디까지 읽었는지를 오프셋으로 컨슈머 그룹 단위로 기억한다. 그래서 같은 토픽을 여러 그룹이 각자의 오프셋으로 독립적으로 읽는다.
- 컨슈머 그룹 안에서 한 파티션은 인스턴스 하나에만 할당된다. 파티션 수가 그룹의 최대 병렬 처리 수이므로, 파티션 수는 그룹의 인스턴스 수 이상으로 둔다.
- 토픽과 컨슈머 그룹은 N : M이다. 한 토픽을 여러 그룹이 구독하고, 한 그룹이 여러 토픽을 구독할 수 있다.
- 파티션은 RF(Replication Factor)만큼 복제되어 서로 다른 Broker에 놓인다. 그중 하나가 leader, 나머지가 follower다.

**Kafka는 꺼내면 사라지는 큐가 아니라 보존 기간 동안 남는 로그이고, 어디까지 읽었는지는 컨슈머 그룹이 기억한다.**

<br>

### **클러스터 구성**

컨트롤러 쿼럼과 브로커 여러 대로 이루어진 Kafka 클러스터에 프로듀서가 메시지를 publish 하고 컨슈머 그룹이 subscribe 하는 구성

![](/assets/img/posts/2026-09-19-kafka-cluster-kraft-01.png)

- 노드마다 역할(process.roles)이 있다. Controller는 클러스터 메타데이터를 관리하는 Control Plane이고, Broker는 메시지를 저장하고 전달하는 Data Plane이다. 한 노드가 두 역할을 함께 가질 수도 있다(combined). combined는 개발 환경처럼 작은 규모에 맞고, Controller를 Broker와 따로 롤링하거나 확장할 수 없어 운영 환경에는 권장되지 않는다.
- Controller는 3대 또는 5대로 둔다. 메타데이터 합의는 과반(quorum)이 성립해야 진행되므로, 3대면 1대, 5대면 2대의 장애를 견딘다. 쿼럼 리더가 Active Controller가 되고 나머지는 hot standby로 대기한다.
- Broker는 3대 이상 둔다. 같은 파티션의 replica는 서로 다른 Broker에 배치되므로, RF 3이면 Broker도 최소 3대가 필요하다.
- 파티션의 leader는 Broker에 고르게 분산된다. 저장과 조회는 leader에서만 처리되고, follower는 복제만 한다.

**Controller는 메타데이터를, Broker는 메시지를 맡고, 둘 다 3대 이상이어야 한 대의 장애를 견딘다.**

<br>

### **KRaft**

ZooKeeper 없이 Kafka 자체의 Raft 합의로 메타데이터를 관리하는 모드

- 이전에는 브로커 목록, 토픽 구성, 파티션 leader 정보를 ZooKeeper에 두고 Controller 브로커 한 대가 이를 읽어 클러스터를 관리했다.
- KRaft에서는 Controller 노드들이 Raft 쿼럼을 구성하고, 메타데이터를 내부 토픽(\_\_cluster\_metadata)의 로그로 복제한다. 쿼럼 리더가 Active Controller이며, 나머지 Controller는 같은 로그를 따라가며 대기한다.
- Broker는 Active Controller에서 메타데이터 로그를 가져와 자기 캐시에 반영한다. 그래서 어느 Broker든 최신 파티션 배치와 leader 정보를 알고 있다.
- 운영 대상이 Kafka 하나로 줄고, Controller 전환이 로그 복제 기반이라 메타데이터를 다시 읽어 올 필요가 없어 전환이 빠르다.
- Kafka 4.0부터 ZooKeeper 모드는 제거되어 KRaft만 남았다.

**KRaft는 메타데이터 저장소를 ZooKeeper에서 Kafka 내부의 Raft 복제 로그로 옮긴 것이다.**

<br>

### **클러스터 시작 과정**

저장 디렉터리 초기화 → Controller 쿼럼 형성 → Broker 등록 → 메타데이터 동기화 순으로 시작

1. 클러스터 ID를 만들고 각 노드의 저장 디렉터리를 그 ID로 초기화한다. (kafka-storage format)
2. Controller 노드들이 서로 연결해 Raft 선거로 리더를 뽑는다. 과반이 모여야 리더가 결정되고, 이 리더가 Active Controller가 된다.
3. Broker가 기동하면 Active Controller에 자신을 등록하고 heartbeat를 보내기 시작한다. 등록 직후는 fenced 상태라 아직 요청을 받지 않는다.
4. Broker가 메타데이터 로그를 따라잡으면 Controller가 fenced를 해제하고, 그때부터 파티션 leader 역할과 클라이언트 요청을 맡는다.
5. 클라이언트는 bootstrap.servers에 적힌 Broker 아무 곳에나 접속해 메타데이터를 받고, 그 뒤로는 각 파티션의 leader Broker에 직접 연결한다.

**Controller 쿼럼이 먼저 구성되고, Broker는 메타데이터를 따라잡은 뒤에야 요청을 받는다.**

<br>

### **저장과 조회**

저장과 조회 모두 파티션 leader가 처리하고, follower는 leader에서 복제

- 프로듀서는 메시지가 갈 파티션을 정하고, 그 파티션의 leader Broker에 직접 전송한다. Controller는 이 경로에 관여하지 않는다.
- leader는 메시지를 자기 로그 끝에 추가하고, follower들은 leader에서 로그를 가져와(fetch) 자기 로그에 기록한다. leader를 따라잡고 있는 follower의 집합이 ISR(In-Sync Replica)이고, ISR 중 몇 대가 기록한 뒤 응답할지는 프로듀서의 acks 설정이 정한다. 이 구조는 다음 편에서 다룬다.
- 컨슈머는 파티션 leader에서 오프셋 순서대로 메시지를 가져온다. 한 파티션 안에서는 순서가 보장되지만 파티션 사이에는 보장되지 않는다.
- 어느 오프셋까지 처리했는지는 컨슈머 그룹이 내부 토픽(\_\_consumer\_offsets)에 기록한다. 이 구조는 컨슈머 그룹 편에서 다룬다.

**메시지 경로는 프로듀서 → leader → follower → 컨슈머이고, Controller는 그 경로 밖에서 배치만 결정한다.**

<br>

### **Controller의 역할**

Active Controller가 수행하는 클러스터 관리 작업

| 역할 | 내용 |
|---|---|
| Broker 멤버십 | Broker 등록을 받고 heartbeat로 생존을 추적한다 |
| 파티션 leader 선출 | Broker에 장애가 나거나 Broker가 추가될 때 영향받는 파티션의 새 leader를 ISR 중에서 정한다 |
| 토픽·파티션 관리 | 토픽 생성과 삭제, 파티션 추가, replica 배치를 처리한다 |
| ISR 관리 | 각 파티션의 ISR 변경을 메타데이터에 반영한다 |
| 설정·ACL·Quota | 동적 설정, 접근 제어 목록, 대역폭과 요청량 제한을 관리한다 |
| 메타데이터 합의 | Controller 쿼럼에서 Raft로 메타데이터 로그를 복제하고 커밋해 Broker에 전파한다 |

- 이 작업들은 모두 메타데이터 로그의 레코드로 기록되고, 커밋된 뒤에야 Broker에 반영된다. 그래서 Active Controller가 바뀌어도 결정의 이력이 남는다.

**Controller는 메시지를 처리하지 않고, 누가 어느 파티션의 leader인지 같은 사실만 결정하고 전파한다.**

<br>

### **장애 판정**

Broker 장애는 heartbeat 끊김으로, Controller 장애는 Raft 선거 타임아웃으로 판정

- Broker는 broker.heartbeat.interval.ms(default 2초)마다 Active Controller에 heartbeat를 보낸다. broker.session.timeout.ms(default 9초) 동안 heartbeat가 없으면 Controller는 그 Broker를 fenced로 판정한다.
- fenced Broker는 파티션 leader 자격을 잃는다. 그 Broker가 leader였던 파티션마다 새 leader 선출이 시작되고, ISR에서도 제외된다.
- Active Controller에 장애가 나면 나머지 Controller는 controller.quorum.election.timeout.ms(default 1초) 동안 리더의 메시지를 받지 못한 뒤 새 선거를 시작한다.
- 판정은 모두 Controller 쪽에서 이루어진다. Broker끼리 서로의 생사를 판정하지 않는다.

**Broker 장애는 Controller가 heartbeat로, Controller 장애는 쿼럼이 선거로 판정한다.**

<br>

### **failover**

파티션 leader는 ISR 중에서, Active Controller는 쿼럼 선거로 교체

1. Broker가 fenced가 되면 Controller는 그 Broker가 leader였던 파티션마다 ISR에 남은 replica 중 하나를 새 leader로 정한다.
2. 새 leader 정보가 메타데이터 로그에 커밋되어 모든 Broker에 전파된다. 클라이언트는 다음 요청의 오류 응답이나 메타데이터 갱신으로 새 leader를 알게 되어 재연결한다.
3. ISR에 남은 replica가 없으면, Kafka 4.1부터 새 클러스터에 기본으로 켜지는 ELR(Eligible Leader Replicas) 중에서 leader를 고른다. ELR도 없으면 그 파티션은 leader 없이 멈춘다. 이때 ISR 밖의 replica를 leader로 올리는 선택지(unclean leader election)와 그 대가는 다음 편에서 다룬다.
4. Active Controller에 장애가 나면 쿼럼의 나머지 Controller가 Raft 선거로 새 리더를 뽑는다. 메타데이터 로그가 이미 복제되어 있으므로 새 리더는 바로 관리 작업을 이어간다.
5. 장애가 났던 Broker가 복구되면 다시 등록하고 메타데이터를 따라잡은 뒤 follower로 복귀한다. auto.leader.rebalance.enable(default true)이 켜져 있으면 Controller가 주기적으로 원래 leader(preferred leader)에게 leader 자리를 되돌린다.

**파티션 leader는 Controller가 ISR 안에서 고르고, Active Controller는 쿼럼이 선거로 교체한다.**

<br>

### **관련 설정**

Broker와 Controller의 server.properties

```
process.roles=broker,controller
node.id=1
controller.quorum.bootstrap.servers=host1:9093,host2:9093,host3:9093
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
controller.listener.names=CONTROLLER
log.dirs=/var/lib/kafka/data
```

- process.roles : 노드의 역할이다. broker, controller, 또는 둘 다를 쉼표로 적는다.
- controller.quorum.bootstrap.servers : Broker와 Controller가 Active Controller를 찾기 위해 접속하는 Controller 목록이다. 쿼럼 구성원 자체는 format 시점에 정한다.
- controller.quorum.voters : 쿼럼 구성원을 id@host:port로 나열하는 이전 방식(정적 쿼럼)이다. 동적 쿼럼에서는 설정하지 않는다.
- broker.session.timeout.ms : heartbeat가 없을 때 Broker의 lease가 유지되는 시간이다. (default 9초) 이 시간이 지나면 fenced로 판정된다.
- broker.heartbeat.interval.ms : Broker가 Controller에 heartbeat를 보내는 주기다. (default 2초)

<br>

> Controller 쿼럼과 Broker로 이루어진 클러스터의 구성과 시작, 장애 판정과 failover를 정리했다.  
> 그렇다면 하나의 토픽은 파티션과 replica로 어떻게 나뉘고, ISR과 acks는 유실을 어디까지 막아줄까?  
> 다음 편에서는 파티션 단위의 저장과 복제를 다루는 [토픽과 파티션](/posts/kafka-topic-partition-replication/)을 다룬다.
{: .prompt-tip }

<br>

> **Kafka 아키텍처 시리즈**
>
> 1. **클러스터 구성 (KRaft, Controller, Broker)**
> 2. [토픽과 파티션 (Replication, ISR)](/posts/kafka-topic-partition-replication/)
> 3. [프로듀서 (acks, 파티셔너, 멱등성)](/posts/kafka-producer/)
> 4. [컨슈머 그룹과 오프셋 (Group Coordinator, Rebalancing)](/posts/kafka-consumer-group/)
> 5. [적용 : 컨슈머 증설과 리밸런싱 비용 (TransCourse)](/posts/kafka-rebalance-duplicate-encoding/)
{: .prompt-info }
