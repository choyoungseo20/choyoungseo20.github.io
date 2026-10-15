---
title: "Kafka 프로듀서 (acks, 파티셔너, 멱등성)"
date: 2026-09-19 00:20:00 +0900
categories: [Backend, Kafka]
tags: [TransCourse, Kafka, producer, acks, idempotence, partitioner]
---

### **프로듀서 구성**

레코드가 갈 파티션을 정하고, 배치로 모아 파티션 leader에 보내고, 응답과 재시도를 처리하는 클라이언트

- 레코드는 토픽, 키, 값, 헤더, 타임스탬프로 이루어진다. 키와 값은 직렬화기(serializer)가 바이트로 바꾼다.
- 파티셔너(partitioner)가 레코드가 갈 파티션을 정한다. 레코드는 파티션별 배치에 쌓인다(record accumulator).
- 별도의 sender 스레드가 배치를 꺼내 각 파티션 leader Broker에 전송하고 응답을 처리한다. send()를 호출한 스레드는 배치에 넣는 즉시 돌아온다.
- 프로듀서는 bootstrap.servers로 접속해 메타데이터를 받고, 파티션 leader가 있는 Broker마다 연결을 유지한다.

**send()는 배치에 넣는 것이고, 실제 전송과 응답은 sender 스레드가 비동기로 처리한다.**

<br>

### **전송 과정**

직렬화 → 파티션 결정 → 배치 적재 → sender 전송 → leader 기록 → 응답 순으로 진행

1. send()가 호출되면 키와 값을 직렬화하고 파티셔너로 파티션을 정한다.
2. 그 파티션의 배치에 레코드를 넣는다. 배치가 batch.size에 차거나 linger.ms가 지나면 전송 대상이 된다.
3. sender 스레드가 같은 Broker로 갈 배치들을 하나의 요청으로 묶어 leader에 보낸다.
4. leader는 로그에 추가하고 acks 조건이 충족되면 응답한다. 응답에는 각 레코드의 오프셋이 담긴다.
5. 응답이 오면 send()에 넘긴 콜백이나 Future가 완료된다. 재시도 가능한 오류(leader 변경, 일시적 네트워크 오류)면 sender가 메타데이터를 갱신하고 다시 보낸다.

**배치는 batch.size와 linger.ms 중 먼저 오는 조건에서 전송된다.**

<br>

### **파티셔너**

키가 있으면 키의 해시로, 없으면 배치 단위로 파티션을 결정

- 키가 있으면 키의 murmur2 해시를 파티션 수로 나눈 나머지가 파티션이다. 같은 키는 파티션 수가 바뀌지 않는 한 항상 같은 파티션으로 가므로, 키 단위 순서가 보장된다.
- 키가 없으면 sticky 방식으로 batch.size만큼 쌓일 때까지 같은 파티션에 넣고, 다음 배치는 다른 파티션으로 옮긴다. 레코드 단위 라운드로빈보다 배치가 커져 전송 효율이 좋다. default로 켜져 있는 adaptive partitioning은 응답이 빠른 Broker의 파티션에 더 많이 배정한다.
- 파티션 수를 늘리면 해시 결과가 바뀌어 기존 키가 다른 파티션으로 간다. 키 순서가 중요한 토픽은 파티션 수를 처음부터 넉넉히 잡는다.
- 특정 키에 메시지가 몰리면 그 파티션만 커진다. 키 설계가 파티션 간 부하 균형을 정한다.

**키는 순서를 주고, 키가 없으면 효율을 얻는다.**

<br>

### **acks와 저장 보장**

leader가 응답을 돌려주는 시점을 정하는 설정

| acks | 응답 시점 | 유실 조건 | 지연 |
|---|---|---|---|
| 0 | 전송 직후, 응답을 기다리지 않음 | 네트워크 오류나 leader 장애 시 유실 | 가장 낮음 |
| 1 | leader 로그에 기록된 뒤 | 복제 전 leader 장애 시 유실 | 중간 |
| all (-1) | ISR 전체가 기록한 뒤 | ISR이 min.insync.replicas 이상이면 유실 없음 | 가장 높음 |

- acks=all은 ISR 전체를 기다리지만, ISR이 leader 하나뿐이면 leader만 기록하고 응답한다. 그래서 유실 방지는 acks=all과 min.insync.replicas 2 이상을 함께 둬야 성립한다.
- ISR이 min.insync.replicas보다 작으면 leader는 NotEnoughReplicas로 거부한다. 프로듀서는 재시도하고, 재시도 시간이 다하면 오류로 끝난다.
- acks의 default는 all이다. (Kafka 3.0부터)

**acks=all은 min.insync.replicas와 짝이어야 유실을 막는다.**

<br>

### **재시도와 멱등성**

재시도로 생기는 중복을 프로듀서 ID와 시퀀스 번호로 제거

- leader가 기록은 했는데 응답이 유실되면 프로듀서는 같은 배치를 재전송한다. 멱등성이 없으면 같은 메시지가 두 번 저장된다.
- 멱등 프로듀서는 기동 시 Broker에서 PID(Producer ID)를 받고, 파티션마다 시퀀스 번호를 붙여 보낸다. Broker는 파티션별로 마지막 시퀀스를 기억해 중복 배치를 버린다.
- 재시도 중 배치 순서가 뒤바뀌는 문제도 시퀀스 번호로 막는다. 그래서 max.in.flight.requests.per.connection을 5 이하로 두면서도 순서가 유지된다.
- 멱등성은 한 프로듀서 세션, 한 파티션 안에서만 성립한다. 프로듀서를 재시작하면 PID가 바뀌어 이전 세션의 중복은 걸러지지 않는다.
- enable.idempotence의 default는 true이고(Kafka 3.0부터), 그에 맞춰 acks=all, retries는 최대값이 default다. acks가 all이 아니거나 retries=0이거나 max.in.flight.requests.per.connection이 5를 넘으면, enable.idempotence를 명시하지 않은 경우 멱등성이 조용히 꺼진다. true로 명시한 경우에만 설정 오류가 난다.

**멱등성은 재시도 중복을 없애지만, 세션과 파티션 경계를 넘는 중복은 트랜잭션의 몫이다.**

<br>

### **트랜잭션**

여러 파티션에 걸친 전송을 하나의 단위로 커밋하거나 중단

- transactional.id를 설정하면 프로듀서 재시작 후에도 같은 ID로 이전 세션의 미완료 트랜잭션을 중단하고 이어갈 수 있다. 멱등성의 세션 경계를 넘는 수단이다.
- Broker의 Transaction Coordinator가 트랜잭션 상태를 내부 토픽(\_\_transaction\_state)에 기록한다.
- 컨슈머는 isolation.level을 read_committed로 두면 커밋된 메시지만 조회한다. default는 read_uncommitted다.
- 대표적인 용도는 consume-transform-produce다. 조회한 오프셋의 커밋과 결과 메시지의 전송을 한 트랜잭션에 묶어 정확히 한 번 처리를 만든다.

**트랜잭션은 여러 파티션과 오프셋 커밋을 원자적으로 묶는다.**

<br>

### **장애 발생과 대응**

leader 변경, Broker 장애, 저장 거부 각각에 대한 프로듀서의 동작

- leader가 바뀌면 이전 leader에 보낸 요청은 NotLeaderOrFollower 오류를 받는다. 프로듀서는 메타데이터를 갱신하고 새 leader로 재전송한다. 재시도 가능한 오류라 애플리케이션에는 보이지 않는다.
- Broker 연결이 끊기면 request.timeout.ms(default 30초) 뒤 재시도한다. delivery.timeout.ms(default 2분) 안에 성공하지 못하면 send()가 오류로 완료된다.
- 배치 적재 공간(buffer.memory, default 32MB)이 차면 send()가 max.block.ms(default 1분) 동안 블로킹된다. 이때는 애플리케이션 스레드가 멈추므로 전송 속도보다 생산 속도가 빠른 상황을 뜻한다.
- 재시도 가능하지 않은 오류(직렬화 실패, 메시지 크기 초과, 인가 실패)는 즉시 오류로 완료된다.

**재시도 가능한 오류는 delivery.timeout.ms 안에서 프로듀서가 알아서 처리한다.**

<br>

### **관련 설정**

프로듀서 설정

```
bootstrap.servers=host1:9092,host2:9092,host3:9092
acks=all
enable.idempotence=true
linger.ms=5
batch.size=16384
compression.type=lz4
```

- acks : 응답 시점이다. (default all, Kafka 3.0 이후)
- enable.idempotence : 멱등 프로듀서를 켠다. (default true, Kafka 3.0 이후)
- retries : 재시도 횟수다. (default 최대값) 실제 상한은 delivery.timeout.ms가 정한다.
- delivery.timeout.ms : send() 호출부터 성공 또는 실패 판정까지의 전체 제한 시간이다. (default 2분)
- max.in.flight.requests.per.connection : 응답 없이 보낼 수 있는 요청 수다. (default 5) 멱등성이 켜져 있으면 5 이하에서 순서가 유지된다.
- linger.ms : 배치가 차지 않아도 전송하기까지 기다리는 시간이다. (default 5ms) Kafka 4.0에서 0에서 5로 바뀌었다. 배치가 커져도 지연은 비슷하거나 오히려 줄기 때문이다.
- batch.size : 파티션별 배치의 최대 크기다. (default 16KB)
- compression.type : 배치 단위 압축이다. (default none) lz4, snappy, zstd, gzip 중 고른다.

<br>

> 프로듀서가 파티션을 정하고 응답을 기다리고 중복을 거르는 방식을 정리했다.  
> 그렇다면 메시지를 가져가는 쪽은 파티션을 어떻게 나눠 갖고, 어디까지 처리했는지를 어떻게 기억할까?  
> 다음 편에서는 [컨슈머 그룹과 오프셋](/posts/kafka-consumer-group/)을 다룬다.
{: .prompt-tip }

<br>

> **Kafka 아키텍처 시리즈**
>
> 1. [클러스터 구성 (KRaft, Controller, Broker)](/posts/kafka-cluster-kraft/)
> 2. [토픽과 파티션 (Replication, ISR)](/posts/kafka-topic-partition-replication/)
> 3. **프로듀서 (acks, 파티셔너, 멱등성)**
> 4. [컨슈머 그룹과 오프셋 (Group Coordinator, Rebalancing)](/posts/kafka-consumer-group/)
> 5. [적용 : 컨슈머 증설과 리밸런싱 비용 (TransCourse)](/posts/kafka-rebalance-duplicate-encoding/)
{: .prompt-info }
