## 카프카 핵심 가이드

### 카프카 운영 관리

Kafka cluster를 운영하려면 topic, consumer group, configuration, partition, replica를 관리할 수 있어야 함

Kafka는 기본 운영 작업을 위해 여러 command-line tool을 제공

이 도구들은 대부분 Kafka distribution의 `bin/` 디렉터리에 있음

<br>

운영 도구를 사용할 때 가장 먼저 확인할 점은 tool version

Kafka command-line tool은 broker version과 맞지 않으면 예상과 다르게 동작할 수 있음

가장 안전한 방법은 운영 중인 broker에 배포된 Kafka version의 tool을 사용하는 것

<br>

### 운영 도구 사용 원칙

기본 원칙:
- 실행 전 대상 cluster와 bootstrap server 확인
- broker version과 tool version 일치 확인
- 변경 명령은 먼저 describe/list로 현재 상태 확인
- 자동화에서는 idempotent 옵션을 신중하게 사용
- 삭제/재배치/offset reset은 rollback 가능성을 먼저 검토
- 변경 후 다시 describe/verify로 결과 확인

<br>

Kafka 4.x의 최신 운영 방식은 KRaft를 기준으로 함

책에는 ZooKeeper 기반 cluster의 비상 절차도 포함되어 있으므로, 최신 cluster에서는 해당 절차를 그대로 적용하지 않아야 함

<br>

### Topic Operations

`kafka-topics.sh`는 topic 생성, 조회, partition 추가, 삭제 같은 기본 topic 작업을 수행

topic configuration 변경은 `kafka-topics.sh`보다 `kafka-configs.sh`를 사용하는 것이 권장됨

<br>

대표 작업:
- create topic
- list topics
- describe topic
- add partitions
- delete topic
- diagnose problematic partitions

<br>

### Topic 생성

topic 생성에는 topic name, replication factor, partition count가 필요

<br>

```bash
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create \
  --topic my-topic \
  --replication-factor 2 \
  --partitions 8
```

<br>

`--replication-factor`는 각 partition을 몇 개의 replica로 유지할지 결정

`--partitions`는 topic을 몇 개의 partition으로 나눌지 결정

cluster가 rack-aware assignment를 사용하면 replica가 가능한 다른 rack에 배치됨

<br>

### Topic 이름 규칙

topic name에는 영문자, 숫자, underscore, dash, period를 사용할 수 있음

하지만 period는 metric 이름에서 underscore로 변환될 수 있어 충돌 가능성이 있음

double underscore로 시작하는 이름도 피하는 것이 좋음

Kafka 내부 topic이 `__consumer_offsets`처럼 double underscore를 사용하기 때문

<br>

실무에서는 topic naming convention을 미리 정해야 함

예:
- domain.event.v1
- service.entity.action
- source.system.dataset

<br>

### `--if-not-exists`와 `--if-exists`

자동화에서 topic 생성 시 `--if-not-exists`는 유용

이미 topic이 있어도 실패하지 않기 때문

<br>

반대로 변경 작업에서 `--if-exists`를 무분별하게 쓰면 문제를 숨길 수 있음

생성되어야 할 topic이 없는데도 자동화가 성공한 것처럼 보일 수 있음

<br>

### Topic 목록 조회

cluster의 topic 목록은 `--list`로 확인

<br>

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --list
```

<br>

내부 topic을 제외하려면 `--exclude-internal`을 함께 사용

운영 점검에서는 내부 topic을 볼 때와 숨길 때를 구분해야 함

<br>

### Topic 상세 조회

`--describe`는 topic의 partition count, replication factor, config override, leader, replica, ISR 정보를 보여줌

<br>

```bash
kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe \
  --topic my-topic
```

<br>

확인할 항목:
- partition count
- replication factor
- leader 분포
- replica assignment
- ISR 상태
- topic-level config override

<br>

### 문제가 있는 partition 찾기

`--describe`는 cluster 진단에도 사용할 수 있음

<br>

유용한 필터:
- `--topics-with-overrides`
- `--under-replicated-partitions`
- `--at-min-isr-partitions`
- `--under-min-isr-partitions`
- `--unavailable-partitions`

<br>

`under-replicated partitions`는 일부 replica가 leader를 따라잡지 못하는 상태

배포, broker restart, reassignment 중에는 일시적으로 발생 가능

하지만 오래 지속되면 disk, network, broker load를 확인해야 함

<br>

`at-min-isr partitions`는 아직 사용 가능하지만 redundancy가 거의 사라진 상태

`under-min-isr partitions`는 produce가 실패하거나 사실상 read-only에 가까운 상태가 될 수 있음

`unavailable partitions`는 leader가 없는 상태이므로 더 심각

<br>

### Partition 추가

topic의 partition 수는 늘릴 수 있음

partition 추가는 처리량 확장이나 consumer group parallelism 확장에 사용

<br>

```bash
kafka-topics.sh --bootstrap-server localhost:9092 \
  --alter \
  --topic my-topic \
  --partitions 16
```

<br>

partition 수를 늘리면 기존 key와 partition의 mapping이 바뀔 수 있음

keyed message를 사용하는 topic은 같은 key의 ordering assumption이 깨질 수 있음

따라서 keyed topic은 처음 만들 때 필요한 partition 수를 신중히 잡는 것이 좋음

<br>

### Partition 감소

Kafka는 topic partition 수를 줄이는 것을 지원하지 않음

partition을 삭제하면 해당 partition의 data가 사라지고, offset/order 의미도 깨짐

<br>

partition 수를 줄여야 한다면 보통 새 topic을 만들고 producer traffic을 옮김

예:

```text
my-topic -> my-topic-v2
```

<br>

### Topic 삭제

topic 삭제는 data 삭제를 의미하며 되돌릴 수 없음

운영 환경에서는 삭제 전 producer/consumer, retention, backup, downstream dependency를 확인해야 함

<br>

```bash
kafka-topics.sh --bootstrap-server localhost:9092 \
  --delete \
  --topic my-topic
```

<br>

topic deletion은 비동기 작업

명령이 반환되어도 실제 log file 삭제와 metadata 정리는 시간이 걸릴 수 있음

큰 topic을 한 번에 많이 삭제하면 controller와 broker에 부담을 줄 수 있으므로 순차적으로 진행

<br>

### Consumer Groups

`kafka-consumer-groups.sh`는 consumer group 상태를 조회하고 관리하는 도구

consumer group은 topic partition을 여러 consumer가 나누어 읽기 위한 coordination 단위

<br>

가능한 작업:
- group list
- group describe
- group delete
- offset delete
- offset reset
- offset export/import

<br>

### Consumer group 목록

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
```

<br>

console consumer처럼 ad hoc consumer가 만든 group도 목록에 보일 수 있음

group 이름이 운영 애플리케이션인지, 임시 테스트인지 구분할 수 있어야 함

<br>

### Consumer group 상세 조회

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe \
  --group my-consumer
```

<br>

주요 출력 필드:
- `GROUP`
- `TOPIC`
- `PARTITION`
- `CURRENT-OFFSET`
- `LOG-END-OFFSET`
- `LAG`
- `CONSUMER-ID`
- `HOST`
- `CLIENT-ID`

<br>

`CURRENT-OFFSET`은 group이 다음에 읽을 offset

`LOG-END-OFFSET`은 broker 기준 partition의 끝 offset

`LAG`는 둘의 차이

lag가 계속 증가하면 consumer 처리량, downstream 병목, rebalance 반복 여부를 확인해야 함

<br>

### Consumer group 삭제

group 삭제는 저장된 offset을 제거

group에 active member가 있으면 삭제하면 안 됨

<br>

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --delete \
  --group my-consumer
```

<br>

특정 topic offset만 삭제하는 것도 가능

단, 삭제 후 consumer가 다시 시작될 때 `auto.offset.reset` 정책에 따라 읽기 위치가 달라질 수 있음

<br>

### Offset reset

offset reset은 consumer group의 읽기 위치를 강제로 바꾸는 작업

장애 복구, 재처리, 잘못된 배포 rollback에서 사용될 수 있음

<br>

대표 reset 기준:
- earliest
- latest
- absolute offset
- timestamp
- shift by amount
- exported offset file

<br>

실행 전에는 `--dry-run`으로 변경 결과를 확인

실제 적용은 `--execute`로 수행

<br>

offset reset은 consumer가 실행 중이면 충돌할 수 있으므로 group을 중지한 뒤 수행하는 것이 안전

<br>

### Dynamic Configuration

Kafka는 topic, broker, user, client의 일부 설정을 동적으로 변경할 수 있음

동적 설정 변경에는 `kafka-configs.sh`를 사용

<br>

지원 entity type은 Kafka version에 따라 다를 수 있음

작업 전 현재 version의 `--help`와 공식 문서를 확인해야 함

<br>

### Topic configuration override

cluster default와 다르게 특정 topic만 설정을 바꿀 수 있음

예를 들어 특정 topic의 retention만 1시간으로 줄일 수 있음

<br>

```bash
kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter \
  --entity-type topics \
  --entity-name my-topic \
  --add-config retention.ms=3600000
```

<br>

topic별로 자주 조정하는 설정:
- `cleanup.policy`
- `compression.type`
- `max.message.bytes`
- `min.insync.replicas`
- `retention.bytes`
- `retention.ms`
- `segment.bytes`
- `unclean.leader.election.enable`

<br>

### Client와 user quota

client와 user에는 quota를 설정할 수 있음

quota는 특정 client나 user가 cluster 자원을 과도하게 사용하는 것을 제한

<br>

대표 quota:
- `producer_byte_rate`
- `consumer_byte_rate`
- `request_percentage`

<br>

quota는 broker별로 적용됨

예를 들어 client quota가 10 MB/s이고 broker가 5개면, leadership이 균등할 때 전체 처리량은 broker 수의 영향을 받음

따라서 quota는 leadership balance와 함께 봐야 함

<br>

### Broker configuration override

broker 설정도 일부 동적으로 변경 가능

예를 들어 특정 broker의 `num.io.threads`를 임시로 조정할 수 있음

<br>

```bash
kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter \
  --entity-type brokers \
  --entity-name 1 \
  --add-config num.io.threads=16
```

<br>

동적 변경이 가능한 broker config와 재시작이 필요한 config는 Kafka version에 따라 다름

운영 변경 전 반드시 현재 version의 broker config 문서를 확인해야 함

<br>

### Configuration 조회와 삭제

config override는 `--describe`로 조회

<br>

```bash
kafka-configs.sh --bootstrap-server localhost:9092 \
  --describe \
  --entity-type topics \
  --entity-name my-topic
```

<br>

override 삭제는 `--delete-config`를 사용

<br>

```bash
kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter \
  --entity-type topics \
  --entity-name my-topic \
  --delete-config retention.ms
```

<br>

삭제 후에는 cluster default가 다시 적용됨

<br>

### Producing and Consuming

운영 점검이나 테스트에는 console producer/consumer를 사용할 수 있음

단, production topic에 직접 테스트 message를 넣는 작업은 주의해야 함

<br>

producer:

```bash
kafka-console-producer.sh --bootstrap-server localhost:9092 \
  --topic my-topic
```

<br>

consumer:

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic my-topic \
  --from-beginning
```

<br>

console consumer는 간단한 확인에는 유용하지만, 대량 data 검증이나 structured decoding에는 한계가 있음

schema가 있는 message는 적절한 formatter나 전용 consumer를 사용하는 것이 안전

<br>

### `__consumer_offsets` topic 읽기

consumer offset은 내부 topic인 `__consumer_offsets`에 저장됨

운영 디버깅에서 내부 offset record를 직접 확인해야 하는 경우가 있음

<br>

이때는 일반 string formatter가 아니라 offset record를 해석할 수 있는 formatter를 사용해야 함

내부 topic은 Kafka version에 민감하므로, 단순 조회 목적을 넘어 직접 수정하려고 하면 안 됨

<br>

### Partition Management

partition management는 cluster balance와 availability에 직접 영향을 줌

대표 작업:
- preferred leader election
- partition reassignment
- replication factor 변경
- reassignment cancel
- reassignment verify

<br>

### Preferred leader election

각 partition의 replica list에서 첫 번째 replica를 preferred replica라고 부름

장애나 restart 후 leader가 다른 replica로 옮겨진 뒤 원래 preferred replica로 돌아오지 않으면 leadership이 불균형해질 수 있음

<br>

preferred leader election은 controller에게 가능한 partition leader를 preferred replica로 되돌리도록 요청

<br>

```bash
kafka-leader-election.sh --bootstrap-server localhost:9092 \
  --election-type PREFERRED \
  --all-topic-partitions
```

<br>

특정 topic/partition만 대상으로 지정할 수도 있음

운영 환경에서는 leader movement로 인한 짧은 영향이 있을 수 있으므로 traffic이 낮은 시간에 수행하는 것이 안전

<br>

### Partition reassignment

`kafka-reassign-partitions.sh`는 partition replica assignment를 변경하는 도구

사용 사례:
- broker 추가 후 기존 data 이동
- broker 제거 전 replica 이동
- broker 장애 후 replica 재배치
- topic replication factor 변경
- cluster load balance

<br>

reassignment는 일반적으로 세 단계로 다룸

1. 계획 생성 또는 JSON 작성
2. `--execute`로 실행
3. `--verify`로 완료 확인

<br>

```bash
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --execute
```

<br>

```bash
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --verify
```

<br>

### Reassignment 주의사항

partition 이동은 network, disk, broker CPU에 큰 부하를 줄 수 있음

큰 topic을 많이 옮기면 client latency와 replication lag가 증가할 수 있음

<br>

주의사항:
- 이동 전 current assignment 저장
- rollback용 JSON 보관
- throttle 설정 고려
- broker별 disk 사용량 확인
- leader imbalance 확인
- 완료 후 `--verify` 수행
- throttle이 남아 있지 않은지 확인

<br>

### Replication factor 변경

기존 topic의 replication factor도 reassignment로 변경

replica list에 broker를 추가하면 RF가 증가하고, replica list에서 broker를 제거하면 RF가 감소

<br>

RF를 늘리는 작업은 availability를 높일 수 있지만, 전체 data copy가 발생하므로 cluster 부하를 고려해야 함

RF를 줄이는 작업은 비용을 낮출 수 있지만, failure tolerance가 줄어듦

<br>

### Reassignment cancel

진행 중인 reassignment는 `--cancel`로 취소 가능

과거에는 ZooKeeper node를 직접 조작해야 했지만, 최신 도구는 cancel 기능을 제공

<br>

취소는 원래 replica set으로 되돌리는 것을 목표로 하지만, 장애 broker나 과부하 broker가 관련된 경우 상태를 더 신중히 확인해야 함

<br>

### Dumping Log Segments

`kafka-dump-log.sh`는 broker log segment file을 직접 decode하는 도구

poison pill message, corrupt record, index 문제를 조사할 때 사용할 수 있음

<br>

```bash
kafka-dump-log.sh --files /tmp/kafka-logs/my-topic-0/00000000000000000000.log
```

<br>

payload까지 출력하려면 `--print-data-log`를 사용

<br>

```bash
kafka-dump-log.sh --files /tmp/kafka-logs/my-topic-0/00000000000000000000.log \
  --print-data-log
```

<br>

운영 환경에서는 payload에 민감 정보가 포함될 수 있음

출력 로그를 저장하거나 공유할 때 보안 정책을 확인해야 함

<br>

### Index 검증

log segment와 함께 index file도 문제를 일으킬 수 있음

`kafka-dump-log.sh`는 index sanity check와 verify index 기능을 제공

<br>

대표 옵션:
- `--index-sanity-check`
- `--verify-index-only`
- `--value-decoder-class`

<br>

broker가 비정상 종료된 뒤 시작할 때도 log/index 검증이 수행되지만, 필요하면 수동으로 확인 가능

<br>

### Replica Verification

`kafka-replica-verification.sh`는 replica들이 같은 message를 가지고 있는지 확인하는 도구

지정된 broker와 topic pattern을 대상으로 replica 간 max lag를 출력

<br>

이 도구는 오래된 offset부터 message를 읽어 검증하므로 cluster에 큰 부하를 줄 수 있음

운영 중 전체 topic에 무심코 실행하면 reassignment에 준하는 영향을 만들 수 있음

<br>

사용 시 주의사항:
- 대상 topic pattern 제한
- broker list 제한
- traffic이 낮은 시간에 실행
- lag와 broker load 모니터링
- 장시간 실행 여부 확인

<br>

### Other Tools

Kafka distribution에는 이 장에서 자세히 다루지 않은 도구도 있음

대표 예:
- `kafka-acls.sh`
- `kafka-mirror-maker.sh`
- `kafka-broker-api-versions.sh`
- producer performance test script
- consumer performance test script
- `trogdor.sh`

<br>

보안은 11장, cluster 간 mirroring은 10장, monitoring은 13장과 연결됨

운영 자동화를 만들 때는 각 도구의 출력 format이 version에 따라 달라질 수 있음을 고려해야 함

<br>

### Unsafe Operations

책에는 ZooKeeper metadata를 직접 조작하는 비상 절차가 소개됨

예:
- controller 강제 이동
- stuck topic deletion 제거
- topic metadata 수동 삭제

<br>

이 절차들은 정상 운영 절차가 아님

대부분 undocumented/unsupported에 가깝고 data loss나 cluster inconsistency를 만들 수 있음

<br>

특히 최신 KRaft 기반 Kafka에서는 ZooKeeper znode가 없으므로, 책의 ZooKeeper 직접 조작 절차를 그대로 적용할 수 없음

Kafka 4.x 운영에서는 KRaft controller quorum, metadata log, Admin API, 공식 recovery guide를 기준으로 판단해야 함

<br>

### Controller 관련 비상 작업

ZooKeeper 기반 cluster에서는 `/controller` 또는 `/admin/controller` 같은 znode를 조작해 controller election에 영향을 줄 수 있었음

하지만 이는 마지막 수단이며, 정상적인 운영 작업으로 취급하면 안 됨

<br>

KRaft 기반 cluster에서는 controller가 Raft quorum으로 동작

controller 문제는 quorum 상태, controller listener, metadata log, broker/controller role을 기준으로 진단해야 함

<br>

### Topic deletion 비상 처리

책은 ZooKeeper 기반 cluster에서 stuck delete request를 제거하는 절차를 다룸

하지만 topic deletion은 data를 제거하는 작업이고, metadata 직접 조작은 cluster를 불안정하게 만들 수 있음

<br>

정상적인 우선순위:
- deletion이 실제로 enable되어 있는지 확인
- broker와 controller log 확인
- offline replica나 broker 장애 확인
- topic describe로 상태 확인
- 공식 tool과 Admin API 기준으로 복구
- metadata 직접 조작은 vendor/support/운영 책임자 승인 후 최후 수단으로만 고려

<br>

### 운영 자동화 관점

Kafka CLI는 운영 자동화에 유용하지만, 단순 shell wrapping만으로는 위험할 수 있음

변경 작업은 항상 pre-check, execute, post-check 구조로 만들어야 함

<br>

권장 흐름:
- 대상 cluster 식별
- 현재 상태 snapshot
- 변경 계획 생성
- dry-run 또는 validation
- 실행
- verify
- rollback 자료 보관
- audit log 기록

<br>

### 운영 관점 체크 포인트

- Kafka CLI tool version과 broker version 일치
- `--bootstrap-server` 대상 cluster 확인
- topic 생성 시 partition/RF/name convention 검토
- keyed topic은 partition 증가 영향 확인
- partition 감소는 지원하지 않음
- topic 삭제는 비가역 작업으로 취급
- consumer group lag는 `CURRENT-OFFSET`, `LOG-END-OFFSET`, `LAG`를 함께 확인
- offset reset은 consumer 중지 후 dry-run 먼저 수행
- topic config 변경은 `kafka-configs.sh` 사용
- quota는 broker별 적용이므로 leadership balance와 함께 확인
- reassignment 전 current assignment와 rollback JSON 보관
- reassignment는 throttle과 verify를 함께 사용
- `kafka-dump-log.sh` 출력의 민감 정보 노출 주의
- replica verification은 cluster 부하를 고려해 제한적으로 실행
- ZooKeeper 직접 조작 절차는 구버전 비상 절차로만 취급
- KRaft cluster는 현재 Kafka version의 공식 운영 문서를 기준으로 복구

<br>
