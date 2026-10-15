---
title: "컨슈머 증설과 리밸런싱 비용 (CooperativeStickyAssignor)"
description: 인코딩 중 컨슈머를 늘릴 때 다시 실행되는 인코딩을 리밸런싱 기록으로 추적해, 원인을 전체 revoke가 아닌 다른 컨슈머로 이동하는 파티션으로 좁히고 낭비를 평균 16%에서 11%로 줄인 실험 기록
date: 2026-10-15 22:00:00 +0900
categories: [Backend, Kafka]
tags: [TransCourse, Kafka, rebalancing, consumer-group, CooperativeStickyAssignor, Spring-Kafka, at-least-once]
---

### **들어가며**

강의 영상을 올리면 썸네일 생성, 메타데이터 추출, HLS 트랜스코딩이 각각 Kafka 컨슈머 그룹에서 실행된다.

- 트랜스코딩은 영상 한 건에 수십 분이 걸린다. 처리량은 컨슈머 대수로 늘리는 구조라, 인코딩 도중에 컨슈머를 늘리는 일이 운영의 기본 동작이 된다.
- 컨슈머가 늘면 리밸런싱이 일어나고, 처리 중이던 레코드가 다른 컨슈머에게 다시 전달될 수 있다.
- 작업 결과는 시도 번호(attemptCount fencing)로 지켜 두었지만, 다시 실행되며 낭비되는 인코딩 시간이 얼마인지는 몰랐다.

처음에는 default 할당 전략(RangeAssignor)이 파티션 6개를 전부 revoke하는 것이 원인이라고 봤다. 리밸런싱 기록과 레코드별 실행 이력을 대조하니, 중복은 다른 컨슈머로 실제로 이동한 파티션에서만 생기고 있었다.

**목표를 revoke 범위가 아니라 파티션 이동을 줄이는 것으로 바꿔, 인코딩 낭비를 평균 16%에서 11%로 줄였다.**

<br>

### **배경 : 긴 작업을 Kafka로 처리하는 구조**

작업 원장은 MySQL에 두고, Kafka에는 작업 ID만 전달

![](/assets/img/posts/2026-10-15-kafka-rebalance-duplicate-encoding-01.png)

처음에는 후처리를 `@Async` 스레드 풀에서 실행했다. 인코딩이 길어지면서 세 가지 한계가 드러났다.

- 처리량을 인스턴스 하나의 스레드 풀 크기 이상으로 늘릴 수 없다.
- 인스턴스가 재시작되면 진행 중인 작업이 사라진다.
- 큐가 차면 작업 제출 자체가 거부된다.

그래서 작업을 프로세스 밖에 보존하고 인스턴스 대수로 처리량을 늘리도록 Kafka로 옮겼다.

- 업로드 트랜잭션이 커밋된 뒤 작업 ID를 발행한다. 발행이 유실되면 복구 폴러가 PENDING 상태로 남은 작업을 다시 발행한다.
- 전달은 at-least-once다. 같은 작업이 두 번 이상 도착할 수 있다.

<br>

### **배경 : 인코딩 중에는 파티션을 멈춘다**

인코딩을 워커 스레드에 넘기고, 끝날 때까지 그 파티션만 pause

![](/assets/img/posts/2026-10-15-kafka-rebalance-duplicate-encoding-02.png)

인코딩이 `max.poll.interval.ms`보다 길어질 수 있어서, 리스너 스레드에서 직접 인코딩하지 않았다.

```java
// TranscodingJobConsumer.handle() 일부 (실행 기록과 실패 처리 생략)
MessageListenerContainer container = containerOf(partition);
container.pausePartition(partition);
transcodingExecutor.execute(() -> {
    try {
        JobExecutionResult result = run(job);
        ack.acknowledge();
    } finally {
        container.resumePartition(partition);
    }
});
```

- `max.poll.records=1`로 한 번에 한 건만 가져오고, 그 파티션을 멈춘 뒤 워커 스레드에 넘긴다.
- 컨슈머 스레드는 계속 poll을 돌아 그룹에서 빠지지 않는다.
- 인코딩이 끝나면 수동으로 ack(MANUAL_IMMEDIATE)하고 파티션을 다시 연다.
- 파티션마다 진행 중인 레코드는 1건이므로, 동시 실행 수의 상한은 스레드 풀이 아니라 할당받은 파티션 수다. 컨슈머를 늘린 만큼 처리량이 늘어나는 이유다.
- pause는 Consumer에 직접 걸지 않고 Spring Kafka 컨테이너 API로 요청했다. 이 선택이 뒤에서 결과를 가른다.

같은 작업이 두 번 도착하면 시도 번호로 막는다.

```java
// RUNNING에서의 start는 at-least-once 재전달을 새 시도로 선점하는 경우
// 먼저 돌던 시도의 결과는 시도 번호 불일치로 거부
public void start() {
    ensureStatusIn(JobStatus.PENDING, JobStatus.FAILED, JobStatus.RUNNING);
    this.attemptCount++;
    this.status = JobStatus.RUNNING;
}
```

- 새 시도가 시작되면 먼저 돌던 시도는 끝까지 실행되지만, 결과를 기록하는 순간 시도 번호가 달라 거부된다.

**결과는 지켰지만, 거부된 시도가 쓴 인코딩 시간은 그대로 낭비된다. 그 크기를 재기로 했다.**

<br>

### **실험 설계 : 인코딩 중에 컨슈머 1대 늘리기**

할당 전략만 바꿔 가며 같은 증설을 반복하고, 리밸런싱과 레코드 실행을 따로 기록

- 환경 : 단일 호스트 Docker Compose, Kafka 브로커 1대(KRaft), Docker VM 메모리 16GB, 인스턴스 JVM 힙 512MB
- 트랜스코딩 그룹 : 컨슈머 3대(앱 1, 워커 2), 파티션 6개, `group.protocol=classic`
- 부하 : 240초짜리 720p 노이즈 영상 12건 업로드
- 증설 : 3개 이상의 파티션에서 인코딩이 진행 중일 때 워커 1대 추가
- 반복 : Range, Sticky, CooperativeSticky를 회차마다 번갈아 설정당 4번

리밸런싱이 일어나는 순간과 레코드가 실행된 결과를 따로 남겼다.

| 테이블 | 기록 시점 | 내용 |
|---|---|---|
| `rebalance_event` | 파티션 revoke·할당마다 | 멤버 ID, 세대, 파티션 목록 |
| `worker_execution` | 레코드 실행마다 | 파티션, 오프셋, 멤버 ID, 세대, 결과(성공·실패·스킵·결과 거부) |

- 같은 (파티션, 오프셋)이 두 번 나오면 재전달이다.
- 결과가 거부된 실행의 시간을 모두 더해, 배치 전체 트랜스코딩 실행 시간으로 나눈 값을 낭비 비율로 정의했다.

**재전달된 레코드와 실제로 다시 실행된 인코딩을 구분해서 셀 수 있게 만들었다.**

<br>

### **첫 가설 : 전체 revoke가 원인이다**

RangeAssignor는 리밸런싱마다 파티션을 전부 revoke하므로, 6개 모두가 중복 후보라고 예상

- default 할당 방식(Eager)에서는 리밸런싱이 시작되면 모든 컨슈머가 파티션을 전부 내려놓고 다시 받는다.
- 실제로 Range는 매회 파티션 6개를 모두 revoke했다.
- 그렇다면 인코딩 중이던 파티션은 모두 다른 컨슈머에게 재전달되고, 다시 실행될 것이라고 봤다.

![](/assets/img/posts/2026-10-15-kafka-rebalance-duplicate-encoding-03.png)

기록을 대조하자 예상과 달랐다.

- 할당이 유지된 파티션 : 재전달은 일어났지만, 레코드는 기존 인코딩이 끝난 뒤에 도착했다. 작업이 이미 끝나 있으니 모두 스킵(SKIPPED)으로 처리됐다.
- 다른 컨슈머로 이동한 파티션 : 새 컨슈머가 레코드를 받자마자 새 시도로 인코딩을 시작했다. 기존 시도는 끝까지 돌고 결과만 거부됐다.

유지된 파티션의 재전달이 늦게 도착한 이유는 pause를 요청한 주체에 있었다.

- pause를 컨테이너 API로 요청했기 때문에, 컨테이너는 리밸런싱 뒤 다시 할당받은 파티션에 pause를 다시 걸었다.
- 그래서 같은 컨슈머가 다시 받은 파티션은 진행 중인 인코딩이 끝나 resume될 때까지 재전달 레코드를 꺼내지 않았다.

**중복은 revoke된 파티션이 아니라, 진행 중인 작업이 다른 컨슈머로 이동한 파티션에서만 생겼다.**

<br>

### **목표 바꾸기 : 파티션 이동 줄이기**

revoke 범위가 아니라 다른 컨슈머로 이동하는 파티션 수를 줄이는 할당 전략을 비교

![](/assets/img/posts/2026-10-15-kafka-rebalance-duplicate-encoding-04.png)

- Range : 파티션을 멤버 ID 정렬 순서대로 나눈다. 새 멤버의 ID가 정렬에서 어디에 들어가느냐에 따라 이동하는 파티션 수가 회차마다 1~5개로 흔들렸다.
- Sticky : 전부 revoke하지만, 다시 나눌 때 기존 배치를 최대한 유지한다.
- CooperativeSticky : 옮겨야 하는 파티션만 revoke하고 나머지는 계속 처리한다.

설정당 4번의 평균은 다음과 같다.

| 항목 | Range | Sticky | CooperativeSticky |
|---|---|---|---|
| revoke된 파티션 | 6 | 6 | 1 |
| 이동한 파티션 | 2 (회차별 5, 1, 1, 1) | 1 | 1 |
| 중복 인코딩 건수 | 1.8 | 1 | 1 |
| 낭비된 인코딩 시간 | 376.6초 | 204.1초 | 197.0초 |
| 낭비 비율 | 16.2% (최대 35.7%) | 11.2% (최대 11.8%) | 11.3% (최대 11.8%) |
| 할당이 유지된 파티션의 재전달 스킵 | 3.5 | 4.5 | 0 |

![](/assets/img/posts/2026-10-15-kafka-rebalance-duplicate-encoding-05.png)

- Range의 평균을 끌어올린 것은 파티션 5개가 이동한 1회차(35.7%)다. 이동이 1개인 회차는 다른 전략과 비슷했다.
- Sticky와 CooperativeSticky는 이동 1개로 같았고, 낭비도 11% 수준으로 같았다.

**낭비의 크기는 revoke된 파티션 수가 아니라 이동한 파티션 수를 따라갔다.**

<br>

### **Sticky와 CooperativeSticky 중 고르기**

비용이 같다면, 결과가 무엇에 기대고 있는지로 판단

- Sticky도 매회 파티션 6개를 revoke한다. 할당이 유지된 파티션의 재전달이 회당 평균 4.5건 있었고, 모두 스킵으로 끝났다.
- 그 재전달이 중복 실행으로 번지지 않은 것은 컨테이너가 다시 할당된 파티션에 pause를 다시 걸어 준 덕분이다. 프레임워크 내부 동작에 결과가 기대고 있다.
- CooperativeSticky는 revoke 자체가 1개로 줄어, 할당이 유지된 파티션에는 재전달이 일어나지 않는다.

**같은 비용이라면 프레임워크 내부 동작에 덜 기대는 쪽을 택해, CooperativeStickyAssignor를 선택했다.**

<br>

### **남은 11%와 한계**

남은 낭비는 새 컨슈머에게 넘어가는 최소 1개 파티션의 중복 실행 비용

- 컨슈머가 1대 늘면 새 컨슈머도 파티션을 받아야 하므로, 최소 1개의 이동은 피할 수 없다. 그 파티션에서 인코딩 중이던 작업은 다시 실행된다.
- revoke되는 파티션의 진행 중인 ffmpeg를 취소하는 방법은 구현하지 않았다. 프로세스 종료, 임시 파일 정리, 상태 전이를 함께 조율해야 해서, 증설 빈도에 비해 부담이 크다고 보고 남은 비용을 받아들였다.
- 모든 수치는 단일 호스트, 브로커 1대, 설정당 4번 실행의 결과다. 방향을 확인한 수준이고, 회차별 편차는 증설 시점에 어떤 인코딩이 진행 중이었는지에 따라 달라진다.
- `group.protocol=classic` 기준이다. Kafka 4.0의 새 컨슈머 그룹 프로토콜(KIP-848)과는 비교하지 않았다.

<br>

### **마치며**

처음 질문은 증설할 때 낭비되는 인코딩이 얼마인가였다.

- revoke 범위만 보면 Range의 파티션 6개 전부가 문제처럼 보였다.
- 재전달된 레코드와 실제로 다시 실행된 인코딩을 나눠 세자, 비용은 다른 컨슈머로 이동한 파티션에만 있었다.

> 리밸런싱 비용은 revoke 범위가 아니라 진행 중인 작업이 다른 컨슈머로 이동하는 파티션 수가 결정한다. 재전달과 중복 실행을 나눠 관찰해야 개선 지점이 보인다.
{: .prompt-tip }

- 코드와 실험 스크립트 : [choyoungseo20/transcourse](https://github.com/choyoungseo20/transcourse/tree/kafka-rebalance-2026-10-04)
- 컨슈머 그룹 동작 정리 : [Kafka 컨슈머 그룹과 오프셋](/posts/kafka-consumer-group/)

<br>

> **Kafka 아키텍처 시리즈**
>
> 1. [클러스터 구성 (KRaft, Controller, Broker)](/posts/kafka-cluster-kraft/)
> 2. [토픽과 파티션 (Replication, ISR)](/posts/kafka-topic-partition-replication/)
> 3. [프로듀서 (acks, 파티셔너, 멱등성)](/posts/kafka-producer/)
> 4. [컨슈머 그룹과 오프셋 (Group Coordinator, Rebalancing)](/posts/kafka-consumer-group/)
> 5. **적용 : 컨슈머 증설과 리밸런싱 비용 (TransCourse)**
{: .prompt-info }
