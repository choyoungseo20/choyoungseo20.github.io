---
title: "Kafka 컨슈머 그룹과 오프셋 (Group Coordinator, Rebalancing)"
date: 2026-09-19 00:30:00 +0900
categories: [Backend, Kafka]
tags: [TransCourse, Kafka, consumer-group, offset, rebalancing, group-coordinator]
---

### **컨슈머 그룹 구성**

프로듀서가 보낸 메시지를 가져가는 쪽으로, 같은 group.id를 가진 인스턴스들이 파티션을 나눠 갖고 처리한 지점을 오프셋으로 기억하는 구성

![](/assets/img/posts/2026-09-19-kafka-consumer-group-01.png)

- 한 파티션은 그룹 안에서 인스턴스 하나에만 할당된다. 인스턴스가 파티션보다 많으면 남는 인스턴스는 할당을 받지 못한다.
- 그룹이 다르면 같은 파티션을 각자 처음부터 끝까지 읽는다. 토픽 : 그룹은 N : M이고, 그룹마다 오프셋이 따로 관리된다.
- 그룹마다 Group Coordinator 역할을 하는 Broker가 하나 있다. Coordinator는 멤버십, heartbeat, 오프셋 커밋을 담당한다.
- Coordinator는 group.id로 정해지는 \_\_consumer\_offsets 파티션(구현상 group.id 해시를 파티션 수로 나눈 나머지)의 leader Broker다. 그림에서 Service 0의 Coordinator와 Service 1의 Coordinator가 다른 Broker인 이유다.
- \_\_consumer\_offsets는 default로 파티션 50개, RF 3인 내부 토픽이다. 오프셋 토픽 파티션 하나가 여러 그룹을 담당하므로 파티션 : 그룹은 1 : N이다.

**그룹은 파티션을 나눠 갖는 단위이고, Coordinator는 오프셋 토픽 파티션의 leader가 맡는다.**

<br>

### **그룹 참여 과정**

FindCoordinator → JoinGroup → SyncGroup → Heartbeat → Fetch 순으로 진행

1. 컨슈머는 bootstrap.servers의 Broker에 FindCoordinator를 보내 자기 그룹의 Coordinator Broker를 알아낸다.
2. Coordinator에 JoinGroup을 보낸다. 새 멤버가 들어오면 Coordinator는 리밸런싱을 시작하고, 기존 멤버들도 다시 JoinGroup을 보낸다.
3. 멤버 중 하나가 그룹 리더로 지정되어 파티션 할당(assignment)을 계산하고 SyncGroup으로 Coordinator에 보낸다. Coordinator는 각 멤버에게 자기 몫을 돌려준다.
4. 컨슈머는 할당받은 파티션의 마지막 커밋 오프셋을 Coordinator에서 받아 그 지점부터 fetch를 시작한다. 커밋 오프셋이 없으면 auto.offset.reset(default latest)을 따른다.
5. 이후 heartbeat.interval.ms(default 3초)마다 Coordinator에 heartbeat를 보내 멤버십을 유지한다.
6. Kafka 4.0의 새 그룹 프로토콜(KIP-848)은 할당 계산을 Coordinator가 맡고, JoinGroup과 SyncGroup을 heartbeat 하나로 대신한다. 컨슈머가 group.protocol=consumer를 설정해야 쓴다.

**할당은 그룹이 합의로 정하고, 그 결과를 Coordinator가 나눠 준다.**

<br>

### **조회와 오프셋 커밋**

poll()로 파티션 leader에서 메시지를 가져오고, 처리한 오프셋을 Coordinator에 커밋

- poll()은 할당된 파티션들의 leader Broker에 fetch 요청을 보내 배치로 메시지를 받는다. Coordinator는 fetch 경로에 관여하지 않는다.
- 오프셋 커밋은 다음에 조회할 오프셋을 \_\_consumer\_offsets에 기록하는 일이다. Coordinator는 오프셋 토픽 파티션의 ISR 전체에 복제된 뒤에야 커밋 성공을 응답한다. 컨슈머가 재시작되거나 파티션이 다른 인스턴스로 넘어가도 이 지점부터 이어간다.
- enable.auto.commit(default true)이면 auto.commit.interval.ms(default 5초)마다 poll() 안에서 자동 커밋된다. 처리 중 컨슈머가 종료되면 마지막 커밋 이후 메시지가 다시 조회되어 중복 처리가 생기고, 처리 전에 커밋되면 유실이 생긴다.
- 수동 커밋(commitSync, commitAsync)은 처리를 끝낸 뒤 커밋하므로 at-least-once가 된다. 중복은 처리 쪽의 멱등성으로 막는다.
- 커밋 오프셋은 offsets.retention.minutes(default 7일) 동안 그룹이 비어 있으면 삭제된다.

| 커밋 방식 | 시점 | 장애 시 결과 |
|---|---|---|
| 자동 커밋 | 주기적으로 poll() 안에서 | 처리 전 커밋이면 유실, 처리 후 커밋 전이면 중복 |
| 수동 커밋 (처리 후) | 처리를 마친 뒤 | 중복은 가능하지만 유실은 없다 (at-least-once) |
| 트랜잭션 커밋 | 결과 전송과 함께 원자적으로 | 중복도 유실도 없다 (exactly-once, Kafka 안에서만) |

**오프셋 커밋은 어디까지 처리했다는 기록이고, 커밋 시점이 중복과 유실 중 무엇을 감수할지 정한다.**

<br>

### **리밸런싱**

그룹 멤버나 파티션 수가 바뀔 때 파티션 할당을 다시 계산하는 과정

- 트리거는 멤버 합류, 멤버 이탈(정상 종료 또는 heartbeat 끊김), 구독 토픽의 파티션 수 변경이다.
- eager 방식은 리밸런싱 동안 모든 멤버가 모든 파티션을 내려놓고 다시 받는다. 그 사이 그룹 전체의 조회가 멈춘다.
- cooperative 방식(CooperativeStickyAssignor)은 옮겨야 하는 파티션만 내려놓고 나머지는 계속 처리한다. 두 번의 리밸런싱으로 나뉘지만 정지 구간이 줄어든다.
- 정적 멤버십(group.instance.id)을 주면 인스턴스가 재시작해도 session.timeout.ms 안에 같은 ID로 돌아오면 리밸런싱 없이 같은 파티션을 돌려받는다. 배포 중 불필요한 리밸런싱을 줄인다.
- 리밸런싱 직전에 커밋하지 않은 오프셋은 새로 할당받은 컨슈머가 다시 처리한다. 리밸런싱이 잦으면 중복 처리도 잦아진다.

| 방식 | 정지 범위 | 특징 |
|---|---|---|
| eager | 그룹 전체 | 단순하지만 멤버 하나의 변화에도 모든 멤버의 조회가 멈춘다 |
| cooperative | 옮겨지는 파티션만 | 리밸런싱이 두 단계로 나뉘고 나머지 파티션은 계속 처리된다 |
| 새 프로토콜 (KIP-848) | 옮겨지는 파티션만 | Coordinator가 할당을 계산하고 heartbeat로 점진 적용한다. 할당 방식과 세션 타임아웃도 서버 설정이 정한다 |

**리밸런싱은 할당을 다시 맞추는 비용이고, 방식에 따라 멈추는 범위가 다르다.**

<br>

### **장애 판정**

컨슈머 장애는 Coordinator의 heartbeat 판정으로, 처리 지연은 poll 간격으로 판정

- 컨슈머는 heartbeat.interval.ms(default 3초)마다 heartbeat를 보낸다. session.timeout.ms(default 45초) 동안 heartbeat가 없으면 Coordinator는 그 멤버를 제거하고 리밸런싱을 시작한다.
- heartbeat는 별도 스레드가 보내므로, 처리 코드가 느려도 heartbeat는 유지된다. 대신 poll() 호출 간격이 max.poll.interval.ms(default 5분)를 넘으면 컨슈머 스스로 그룹을 떠난다.
- 처리 지연으로 그룹을 떠나면 리밸런싱이 일어나고, 그 멤버가 처리 중이던 메시지는 커밋되지 않아 다른 멤버가 다시 처리한다. 한 번에 처리하는 양(max.poll.records)이나 max.poll.interval.ms를 조정해 막는다.
- 정상 종료(close)는 LeaveGroup을 보내 즉시 리밸런싱을 유도하므로 session.timeout.ms를 기다리지 않는다.

**살아 있는지는 heartbeat로, 처리하고 있는지는 poll 간격으로 판정한다.**

<br>

### **failover**

컨슈머 장애는 리밸런싱으로, Coordinator 장애는 오프셋 토픽 파티션의 leader 이동으로 복구

1. 멤버가 제거되면 Coordinator가 리밸런싱을 시작하고, 그 멤버의 파티션은 남은 멤버에게 재할당된다.
2. 새로 할당받은 컨슈머는 그 파티션의 마지막 커밋 오프셋부터 조회를 시작한다. 이전 컨슈머가 커밋하지 못한 구간은 다시 처리된다.
3. Coordinator Broker에 장애가 나면 \_\_consumer\_offsets 파티션의 leader가 다른 Broker로 옮겨지고(2편의 failover), 그 Broker가 새 Coordinator가 된다.
4. 컨슈머는 Coordinator 오류를 받으면 FindCoordinator를 다시 보내 새 Coordinator를 찾고 heartbeat와 커밋을 재개한다. 오프셋 토픽이 RF 3으로 복제되어 있으므로 커밋 기록은 유지된다.
5. 그룹 전체가 중단됐다 재시작되면 커밋 오프셋이 남아 있는 한 그 지점부터 이어간다. offsets.retention.minutes를 넘겨 오프셋이 지워졌으면 auto.offset.reset을 따른다.

**컨슈머 failover의 단위는 파티션 재할당이고, 커밋 오프셋이 이어받는 지점을 정한다.**

<br>

### **관련 설정**

컨슈머 설정과 오프셋 토픽 설정

```
bootstrap.servers=host1:9092,host2:9092,host3:9092
group.id=service-0
enable.auto.commit=false
auto.offset.reset=earliest
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

- group.id : 컨슈머 그룹 이름이다. 같은 이름끼리 파티션을 나눠 갖는다.
- enable.auto.commit : 자동 커밋 여부다. (default true) 처리 후 커밋하려면 끄고 수동 커밋한다.
- auto.offset.reset : 커밋 오프셋이 없을 때 시작 지점이다. (default latest) earliest는 처음부터 읽는다.
- session.timeout.ms : heartbeat가 끊긴 뒤 멤버를 제거하기까지의 시간이다. (default 45초)
- heartbeat.interval.ms : heartbeat 주기다. (default 3초) session.timeout.ms의 3분의 1 이하로 둔다.
- max.poll.interval.ms : poll() 호출 사이의 최대 간격이다. (default 5분) 넘기면 그룹을 떠난다.
- max.poll.records : poll() 한 번에 가져오는 최대 레코드 수다. (default 500)
- partition.assignment.strategy : 할당 방식이다. default는 RangeAssignor, CooperativeStickyAssignor 목록이고 앞의 RangeAssignor(eager)가 적용된다. cooperative 리밸런싱을 쓰려면 CooperativeStickyAssignor만 지정한다.
- group.protocol : classic 또는 consumer다. (default classic) consumer로 두면 KIP-848 프로토콜을 쓴다.
- offsets.topic.num.partitions, offsets.topic.replication.factor : 오프셋 토픽의 파티션 수와 RF다. (default 50, 3) Broker 설정이며 토픽 생성 후에는 바꾸기 어렵다.

<br>

### **시리즈 정리**

편별 핵심과, 네 편의 설정이 합쳐져 정해지는 전달 보장

| 편 | 다루는 것 | 핵심 |
|---|---|---|
| 클러스터 구성 | Controller 쿼럼과 Broker | 메타데이터는 Raft로, 메시지는 파티션 leader로 |
| 토픽과 파티션 | 복제, ISR, HW | 안전성은 RF가 아니라 ISR과 min.insync.replicas가 정한다 |
| 프로듀서 | acks, 파티셔너, 멱등성 | acks=all과 min.insync.replicas가 짝이고, 중복은 PID와 시퀀스로 거른다 |
| 컨슈머 그룹 | Coordinator, 오프셋, 리밸런싱 | 커밋 시점이 중복과 유실을 정하고, 리밸런싱이 failover 단위다 |

- 전달 보장은 한 곳의 설정이 아니라 프로듀서의 acks, Broker의 min.insync.replicas, 컨슈머의 커밋 시점이 함께 정한다.

| 보장 | 유실 | 중복 | 성립 조건 |
|---|---|---|---|
| at-most-once | 있다 | 없다 | 프로듀서 재시도 끔(retries=0), 처리 전 커밋 |
| at-least-once | 없다 | 있다 | acks=all과 min.insync.replicas 2 이상, 처리 후 커밋 |
| exactly-once | 없다 | 없다 | 트랜잭션 프로듀서(transactional.id), 컨슈머의 isolation.level=read_committed, 오프셋을 트랜잭션 안에서 커밋(sendOffsetsToTransaction). Kafka 안에서만 성립하고, 외부 시스템까지 이어지려면 처리 쪽 멱등성이 필요하다 |

**대부분의 서비스는 at-least-once를 기본으로 두고, 중복은 처리 쪽 멱등성으로 막는다.**

<br>

> 컨슈머 그룹이 파티션을 나눠 갖고 오프셋으로 진행을 기억하는 방식을 정리했다.  
> Kafka는 유실과 중복을 각 단계의 설정으로 나눠 제어하며, 정확히 한 번 처리는 프로듀서 트랜잭션과 처리 쪽 멱등성이 함께 있어야 성립한다.  
> 실제 서비스에 적용할 때는 acks, min.insync.replicas, 커밋 방식 세 가지를 먼저 정한 뒤 나머지를 맞추면 된다.
{: .prompt-tip }

<br>

> **Kafka 아키텍처 시리즈**
>
> 1. [클러스터 구성 (KRaft, Controller, Broker)](/posts/kafka-cluster-kraft/)
> 2. [토픽과 파티션 (Replication, ISR)](/posts/kafka-topic-partition-replication/)
> 3. [프로듀서 (acks, 파티셔너, 멱등성)](/posts/kafka-producer/)
> 4. **컨슈머 그룹과 오프셋 (Group Coordinator, Rebalancing)**
> 5. [적용 : 컨슈머 증설과 리밸런싱 비용 (TransCourse)](/posts/kafka-rebalance-duplicate-encoding/)
{: .prompt-info }
