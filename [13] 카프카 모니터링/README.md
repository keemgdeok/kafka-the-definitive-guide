## 카프카 핵심 가이드

### 카프카 모니터링

Kafka는 broker, producer, consumer, topic, partition, request, JVM에 대한 많은 metric을 제공

metric이 많다는 것은 cluster 내부를 자세히 볼 수 있다는 뜻이지만, 무엇을 alert로 삼을지 정하지 않으면 noise가 커짐

모니터링은 모든 metric을 수집하는 작업이 아니라, 장애를 빠르게 발견하고 원인을 좁히는 체계를 만드는 작업

<br>

### Metric Basics

Kafka metric은 대부분 JMX(Java Management Extensions)를 통해 노출됨

Kafka broker와 Java client는 JMX로 metric을 제공하고, monitoring agent나 metrics reporter가 이를 수집

<br>

대표 수집 방식:
- process에 붙는 JMX agent
- JMX endpoint를 읽는 외부 collector
- Prometheus JMX exporter 같은 agent
- vendor monitoring agent
- custom metrics reporter

<br>

Kafka 4.3 공식 문서 기준으로도 server와 Java client metric은 JMX와 pluggable reporter를 통해 노출됨

remote JMX는 기본적으로 비활성화되어 있으며, 운영에서 열 경우 인증/권한/네트워크 제한을 반드시 설정해야 함

JMX는 상태 조회뿐 아니라 관리 작업과 code execution surface가 될 수 있으므로 보안 없이 노출하면 안 됨

<br>

### Metric Source

Kafka 운영에서 보는 metric source는 하나가 아님

<br>

주요 source:
- application metrics
- logs
- infrastructure metrics
- synthetic clients
- client metrics

<br>

`application metrics`는 Kafka broker와 client가 직접 제공하는 metric

`logs`는 broker/client가 남기는 event와 error의 기록

`infrastructure metrics`는 network, disk, CPU, load balancer, host metric

`synthetic clients`는 실제 client처럼 produce/consume을 수행해 외부 관점의 상태를 확인

`client metrics`는 producer/consumer application이 노출하는 metric

<br>

Kafka 내부 metric만 보면 broker가 정상이라고 판단할 수 있음

하지만 client가 실제로 produce/consume할 수 있는지는 외부 관점의 metric이 더 잘 보여줌

<br>

### Alerting과 Debugging

metric은 목적에 따라 다르게 수집해야 함

alerting용 metric은 장애를 빠르게 감지해야 하므로 적고 명확해야 함

debugging용 metric은 원인 분석을 위해 더 많고 세밀해도 됨

<br>

구분:
- alerting: 짧은 보존 기간, 명확한 threshold, operator가 바로 행동 가능
- debugging: 긴 보존 기간, 세부 metric, 장애 원인 분석용
- historical metrics: capacity planning과 trend 분석용

<br>

모든 metric에 alert를 걸면 alert fatigue가 발생

operator가 신뢰할 수 있는 적은 수의 alert를 먼저 설계해야 함

<br>

### Health Check

metric 수집과 별도로 process health check가 필요

broker metric이 멈춘 것이 broker 장애인지 monitoring system 장애인지 구분해야 하기 때문

<br>

Kafka broker health check는 client가 사용하는 listener에 연결해 응답 여부를 확인하는 방식이 단순하고 실용적

client application은 process 상태, 내부 readiness, dependency 상태까지 함께 볼 수 있음

<br>

### SLI, SLO, SLA

Kafka는 infrastructure service이므로 SLO를 정하고 고객에게 기대 수준을 명확히 알려야 함

<br>

`SLI`: service reliability를 나타내는 측정값

`SLO`: SLI에 목표값과 기간을 결합한 목표

`SLA`: service provider와 client 사이의 계약

<br>

예:

```text
7일 동안 produce 요청의 99.9%가 성공해야 함
```

<br>

SLO는 broker 내부 상태보다 client 경험에 가까운 metric으로 잡는 것이 좋음

client-side metric, infrastructure metric, synthetic client metric이 좋은 SLI 후보

<br>

### 좋은 SLI 후보

Kafka에서 고려할 수 있는 SLI:
- availability
- latency
- quality
- security
- throughput

<br>

`availability`: client가 요청하고 응답을 받을 수 있는가

`latency`: 응답이 얼마나 빨리 돌아오는가

`quality`: 응답이 정상적인가

`security`: 인증/인가/암호화 요구사항이 지켜지는가

`throughput`: client가 필요한 양의 data를 처리할 수 있는가

<br>

SLI는 가능하면 event 단위 counter로 측정하는 것이 좋음

예를 들어 평균 latency만 보는 것보다, SLO threshold 안에 들어온 요청 비율을 보는 것이 더 직접적

<br>

### SLO 기반 alerting

SLO가 있으면 alert도 SLO burn rate를 기준으로 잡을 수 있음

짧은 시간에 error budget을 빠르게 소모하면 즉시 대응

천천히 소모하는 경우는 warning이나 ticket으로 처리

<br>

이 방식은 단순 threshold alert보다 실제 사용자 영향과 더 가까움

단, SLO를 만들려면 어떤 client 경험을 보장할지 먼저 합의해야 함

<br>

### Kafka Broker Metrics

Kafka broker는 매우 많은 metric을 제공

운영자는 모든 metric을 매일 볼 필요가 없음

우선 cluster health를 빠르게 판단할 수 있는 핵심 metric을 정해야 함

<br>

먼저 봐야 할 영역:
- under-replicated partitions
- active controller
- controller queue
- request handler idle ratio
- bytes in/out
- messages in
- partition count
- leader count
- offline partitions
- request latency breakdown
- JVM GC

<br>

### Monitoring System의 자기 의존성

Kafka로 application metrics와 log를 수집하는 구조는 흔함

하지만 Kafka 자체 monitoring도 같은 Kafka cluster에 의존하면, Kafka가 깨졌을 때 monitoring data도 같이 멈출 수 있음

<br>

대안:
- Kafka monitoring만 별도 경로로 수집
- cluster A metric을 cluster B로 전송
- 외부 monitoring system에 직접 전송
- synthetic client를 Kafka 외부에 배치

<br>

중요한 alert path는 Kafka cluster가 장애일 때도 살아 있어야 함

<br>

### Under-Replicated Partitions

`UnderReplicatedPartitions`는 Kafka에서 가장 중요한 broker health metric 중 하나

leader replica는 있지만 follower replica가 ISR을 따라오지 못하는 partition 수를 나타냄

정상적인 안정 상태에서는 0이어야 함

<br>

JMX 예:

```text
kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
```

<br>

값이 0보다 크면 가능한 원인:
- broker down
- broker restart
- disk I/O 병목
- network 병목
- replica fetch 지연
- reassignment 진행 중
- GC pause
- controller 또는 metadata issue

<br>

일시적인 배포나 재시작 중에는 증가할 수 있음

하지만 값이 계속 유지되거나 증가하면 즉시 원인을 좁혀야 함

<br>

### ISR 관련 metric

Kafka 4.x 공식 문서에는 ISR 상태를 더 직접적으로 보는 metric도 있음

<br>

확인할 metric:
- `UnderReplicatedPartitions`
- `UnderMinIsrPartitionCount`
- `AtMinIsrPartitionCount`
- `IsrShrinksPerSec`
- `IsrExpandsPerSec`

<br>

`UnderMinIsrPartitionCount`가 0보다 크면 produce availability에 직접 영향을 줄 수 있음

`AtMinIsrPartitionCount`는 아직 동작하지만 redundancy가 거의 남지 않은 상태

`IsrShrinksPerSec`가 자주 발생하면 replica 안정성을 확인해야 함

<br>

### Active Controller Count

ZooKeeper 기반 Kafka에서는 cluster 전체에 active controller가 하나만 있어야 함

책의 metric 예:

```text
kafka.controller:type=KafkaController,name=ActiveControllerCount
```

<br>

Kafka 4.x KRaft 기반 cluster에서는 controller 구조와 metric이 달라짐

controller quorum metric, active controller, fenced broker, metadata log 상태를 Kafka version에 맞춰 확인해야 함

<br>

핵심은 controller가 하나의 일관된 metadata authority로 동작하는지 확인하는 것

controller 관련 alert는 Kafka version과 mode에 맞춰 설계해야 함

<br>

### Controller Queue

controller queue는 controller가 처리해야 할 metadata event가 얼마나 밀려 있는지 보여줌

topic 생성/삭제, broker 장애, partition reassignment, leader election이 많으면 queue가 증가할 수 있음

<br>

책의 JMX 예:

```text
kafka.controller:type=ControllerEventManager,name=EventQueueSize
```

<br>

KRaft에서는 controller quorum과 metadata log 관련 metric도 함께 봐야 함

queue가 오래 쌓이면 controller processing 지연으로 partition leadership이나 metadata propagation이 늦어질 수 있음

<br>

### Request Handler Idle Ratio

request handler idle ratio는 broker request handler thread pool이 얼마나 여유 있는지 보여줌

값이 낮을수록 broker가 request 처리로 바쁨

<br>

JMX 예:

```text
kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent
```

<br>

이 값이 지속적으로 낮으면 확인할 항목:
- client request rate
- produce/fetch latency
- request queue time
- CPU 사용률
- quota 적용 여부
- partition leadership imbalance

<br>

### Bytes In/Out

traffic metric은 capacity와 이상 징후를 보는 기본 metric

<br>

대표 JMX:

```text
kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec
kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec
```

<br>

`BytesInPerSec`: producer와 follower replication을 통해 broker로 들어오는 byte rate

`BytesOutPerSec`: consumer와 follower replication을 통해 broker에서 나가는 byte rate

<br>

consumer가 많으면 bytes out이 bytes in보다 훨씬 클 수 있음

replication traffic은 client traffic과 구분해서 보는 것이 좋음

<br>

### Messages In

message rate는 broker가 받아들이는 message 수를 보여줌

<br>

```text
kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec
```

<br>

message size가 변하면 messages/sec와 bytes/sec가 다르게 움직일 수 있음

따라서 throughput은 message rate와 byte rate를 함께 봐야 함

<br>

### Partition Count와 Leader Count

partition count는 broker가 보유한 partition 수

leader count는 broker가 leader로 처리하는 partition 수

<br>

```text
kafka.server:type=ReplicaManager,name=PartitionCount
kafka.server:type=ReplicaManager,name=LeaderCount
```

<br>

partition count와 leader count는 broker 간에 대체로 균형을 이루어야 함

특정 broker에 leader가 몰리면 produce/fetch request가 해당 broker로 집중될 수 있음

<br>

leader imbalance가 보이면 preferred leader election이나 reassignment를 검토

단, 자동으로 움직이기 전에 traffic, disk, rack, replica placement를 함께 확인해야 함

<br>

### Offline Partitions

offline partition은 leader가 없는 partition

client가 해당 partition을 읽거나 쓸 수 없으므로 매우 심각한 상태

<br>

책의 JMX 예:

```text
kafka.controller:type=KafkaController,name=OfflinePartitionsCount
```

<br>

KRaft 기반 Kafka에서는 controller metric 이름과 위치가 version에 따라 달라질 수 있음

하지만 의미는 동일함

leader 없는 partition은 즉시 대응해야 함

<br>

### Request Metrics

Kafka는 request type별 latency breakdown을 제공

특정 produce/fetch 지연이 어디서 발생하는지 분해해서 볼 수 있음

<br>

대표 request metric:
- `TotalTimeMs`
- `RequestQueueTimeMs`
- `LocalTimeMs`
- `RemoteTimeMs`
- `ThrottleTimeMs`
- `ResponseQueueTimeMs`
- `ResponseSendTimeMs`
- `RequestsPerSec`

<br>

JMX 예:

```text
kafka.network:type=RequestMetrics,name=TotalTimeMs,request=FetchConsumer
```

<br>

`RequestQueueTimeMs`가 높으면 request handler나 network thread 쪽 병목 가능성

`LocalTimeMs`가 높으면 broker local processing 병목 가능성

`RemoteTimeMs`가 높으면 replication acknowledgment 대기 가능성

`ThrottleTimeMs`가 높으면 quota나 throttling 영향 가능성

<br>

### Topic and Partition Metrics

cluster 전체 metric만 보면 특정 topic 문제를 놓칠 수 있음

topic별 metric은 특정 tenant, application, pipeline의 이상을 찾는 데 유용

<br>

topic별 대표 metric:
- bytes in rate
- bytes out rate
- messages in rate
- failed fetch request rate
- failed produce request rate
- total fetch request rate
- total produce request rate

<br>

예:

```text
kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec,topic=<topic>
kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec,topic=<topic>
```

<br>

partition별 metric은 cardinality가 매우 큼

필요한 topic/partition에만 제한적으로 수집하거나 debugging 시점에 사용하는 것이 좋음

<br>

### JVM Monitoring

Kafka broker는 JVM 위에서 동작하므로 JVM metric도 중요

<br>

확인할 항목:
- heap usage
- direct memory
- GC count
- GC time
- GC pause
- thread count
- file descriptor

<br>

GC pause가 길어지면 request latency, replica fetch, ISR 유지에 영향을 줄 수 있음

GC metric은 broker metric과 함께 봐야 원인을 더 빨리 좁힐 수 있음

<br>

### Logs

Kafka log는 metric으로 보이지 않는 문제를 보여줌

controller event, broker failure, authentication failure, authorization denial, disk error, network error는 log로 확인해야 함

<br>

운영에서 중요한 log:
- broker server log
- controller log
- authorizer log
- request log
- GC log
- OS/kernel log

<br>

log alert는 너무 넓게 잡으면 noise가 큼

반복되는 error pattern, fatal error, disk/network 관련 error부터 좁게 시작하는 것이 좋음

<br>

### Client Monitoring

Kafka broker가 정상이어도 client application이 정상이라고 볼 수 없음

producer와 consumer는 각각 client-side metric을 수집해야 함

<br>

client metric은 application별로 달라질 수 있음

Java client 기준 metric 이름을 사용하더라도, 다른 language client는 노출 방식이 다를 수 있음

<br>

### Producer Metrics

producer는 broker로 record를 쓰는 client

producer metric은 latency, error, retry, throughput, batching 상태를 보여줌

<br>

대표 metric:
- `record-send-rate`
- `record-error-rate`
- `record-retry-rate`
- `request-latency-avg`
- `record-queue-time-avg`
- `batch-size-avg`
- `compression-rate-avg`
- `outgoing-byte-rate`
- `produce-throttle-time-avg`

<br>

가장 먼저 볼 metric은 error와 latency

`record-error-rate`가 증가하면 broker availability, authorization, serialization, message size, timeout을 확인

`request-latency-avg`가 증가하면 broker, network, acknowledgement, quota를 함께 확인

<br>

### Producer batching

producer latency와 throughput은 batching 설정의 영향을 받음

`batch.size`와 `linger.ms`는 producer가 request를 언제 닫고 broker로 보낼지 결정

<br>

`record-queue-time-avg`는 record가 producer 내부 queue에서 얼마나 기다리는지 보여줌

latency 요구사항을 맞추려면 queue time, request latency, batch size를 함께 봐야 함

<br>

### Consumer Metrics

consumer는 broker에서 record를 읽는 client

consumer metric은 fetch latency, consumed rate, group coordination, commit latency를 보여줌

<br>

대표 metric:
- `fetch-latency-avg`
- `bytes-consumed-rate`
- `records-consumed-rate`
- `fetch-rate`
- `fetch-size-avg`
- `records-per-request-avg`
- `sync-time-avg`
- `sync-rate`
- `commit-latency-avg`
- `assigned-partitions`
- `fetch-throttle-time-avg`

<br>

`fetch-latency-avg`는 fetch request가 broker에서 응답을 받기까지 걸리는 시간

단, `fetch.min.bytes`와 `fetch.max.wait.ms` 설정 때문에 traffic이 적은 topic에서는 latency가 자연스럽게 들쭉날쭉할 수 있음

<br>

### Consumer group coordination

consumer group은 rebalance와 heartbeat를 통해 partition assignment를 유지

rebalance가 자주 발생하면 consumption pause가 생김

<br>

확인할 metric:
- `sync-time-avg`
- `sync-rate`
- `commit-latency-avg`
- `assigned-partitions`

<br>

stable group에서는 `sync-rate`가 대부분 0에 가까워야 함

`commit-latency-avg`가 증가하면 offset commit 지연이 커지고, 장애 후 재처리 범위가 늘어날 수 있음

<br>

### Quotas

Kafka quota는 producer/consumer client가 broker 자원을 과도하게 사용하는 것을 제한

quota에 걸린 client는 error를 받는 것이 아니라 broker response가 지연됨

따라서 client는 throttling metric을 봐야 quota 영향을 알 수 있음

<br>

확인할 metric:
- producer: `produce-throttle-time-avg`
- consumer: `fetch-throttle-time-avg`

<br>

quota를 현재 쓰지 않더라도 이 metric은 수집해두는 것이 좋음

나중에 quota를 켰을 때 client 영향도를 바로 확인할 수 있기 때문

<br>

### Lag Monitoring

consumer lag는 Kafka consumer에서 가장 중요한 metric 중 하나

lag는 broker의 log end offset과 consumer group의 committed offset 차이

<br>

client의 `records-lag-max`는 참고는 가능하지만 한계가 있음

가장 뒤처진 partition 하나만 보여주고, consumer가 정상적으로 동작해야 계산 가능하기 때문

<br>

권장 방식은 외부 lag monitoring

외부 프로세스가 broker의 partition end offset과 consumer group committed offset을 함께 읽어야 함

<br>

확인할 항목:
- group별 lag
- topic/partition별 lag
- lag 증가 속도
- lag가 줄어드는지 여부
- consumer가 progress를 만드는지 여부
- stalled/stopped 상태

<br>

Burrow 같은 도구는 lag 절대값보다 consumer group의 progress를 보고 상태를 판단

threshold를 topic마다 직접 관리하기 어려운 큰 cluster에서 유용

<br>

### End-to-End Monitoring

broker metric과 client metric만으로는 Kafka cluster를 실제로 사용할 수 있는지 완전히 알기 어려움

synthetic client가 직접 produce와 consume을 수행하면 외부 관점의 availability와 latency를 측정할 수 있음

<br>

확인해야 할 질문:
- Kafka cluster에 message를 쓸 수 있는가
- Kafka cluster에서 message를 읽을 수 있는가
- produce-to-consume latency가 허용 범위 안인가
- broker별로 client request가 성공하는가

<br>

이런 end-to-end monitoring은 Kafka 자체가 보고할 수 없는 사용자 관점의 상태를 보여줌

특히 Kafka를 여러 애플리케이션의 공통 인프라로 운영할 때 중요

<br>

### 운영 관점 체크 포인트

- remote JMX를 열면 반드시 인증/권한/네트워크 제한 적용
- alert용 metric과 debugging용 metric을 구분
- Kafka 자체 monitoring이 Kafka cluster에만 의존하지 않도록 설계
- health check와 metric stale alert를 함께 사용
- SLO는 client 경험에 가까운 SLI로 설계
- `UnderReplicatedPartitions`는 안정 상태에서 0 유지
- `UnderMinIsrPartitionCount`는 produce availability와 직접 연결
- active controller/controller quorum metric은 Kafka mode와 version에 맞춰 확인
- request latency는 total time보다 queue/local/remote/throttle breakdown으로 분석
- bytes/sec와 messages/sec를 함께 확인
- partition count와 leader count의 broker별 balance 확인
- offline partition은 즉시 대응
- JVM GC pause와 broker latency를 함께 봄
- producer error/retry/latency/throttle metric 수집
- consumer lag는 client 내부 metric보다 외부 lag monitoring 우선
- consumer rebalance와 commit latency를 함께 확인
- quota를 쓰지 않아도 throttle metric은 미리 수집
- synthetic produce/consume으로 end-to-end 상태 확인

<br>
