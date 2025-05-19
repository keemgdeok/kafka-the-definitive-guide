## 카프카 핵심 가이드

#### 브로커 매개변수
`broker.id`: 카프카 브로커의 정수값 식별자 - 이 값은 브로커당 고유

`listeners`: {protocol}:\//{hostname}:{port}

protocol: PLAINTEXT, SSL, SASL_PLAINTEXT, SASL_SSL 중 하나  

hostname: 브로커의 호스트명 또는 IP 주소

port: 브로커가 리스닝하는 포트 번호

ex. PLAINTEXT://localhost:9092, SSL://kafka1.example.com:9093

`zookeeper.connect`: 브로커의 메타데이터가 저장되는 주키퍼의 위치를 가리킴 
```text 
{호스트이름}:{포트}/{경로}
```
ex. 
단일 주키퍼: localhost:2181
주키퍼 앙상블: zk1.example.com:2181, zk2.example.com:2181, zk3.example.com:2181
특정 경로 사용: localhost:2181/kafka

`log.dirs`: 모든 메시지를 로그 세그먼트 단위로 묶어 저장하는 곳

`num.recovery.threads.per.data.dir`: log.dirs에 지정된 로그 데릭토리별 스레드 수(작업 병렬화)

`auto.create.topics.enable`: 쓰거나 읽고, 메타데이터 요청 시, 카프카 토픽 자동 생성 설정

`auto.leader.rebalance.enable`: 모든 토픽의 리더 역할이 하나의 브로커에게 집중될때, 균등하게 분산

`delete.topic.enable`: 클러스터의 토픽을 임의로 삭제하지 못함
<br>
  
#### 토픽별 기본값
`num.partitions`: 새로운 토픽이 생성될 때 몇 개의 파티션을 갖는지 설정

디스크 IOPS(I/O per sec), 네트워크 대역폭 등 을 고려해야함

`default.replication.factor`: 새로 생성되는 토픽의 복제팩터를 결정

min.insync.replicas + 1 을 잡아줄 것을 권장

`log.retention.ms` : 카프카가 얼마나 오랫동안 메시지를 보존해야 하는지 보존 시간 결정

`log.retention.bytes` : 카프카가 얼만큼의 크기로 메시지를 보존해야 하는지 보존 용량 결정

`log.segment.bytes`: 로그 세그먼트의 제한 크기 결정 (default 1GB)

`log.roll.ms` : 로그 세그먼트의 제한 시간 설정 

`min.insync.replicas` : 2 - 최소한 2개의 레플리카가 최신 상태로 프로듀서와 '동기화' (default 1)

`message.max.bytes` : 카프카 브로커가 쓸 수 있는 메시지의 최대 크기 (default 1MB)

커질수록 I/O처리량에 영향을 미침
<br>
  
#### 하드웨어 선택하기

`디스크 처리량`

가장 큰 영향을 미치며, 디스크 쓰기 속도가 빠를수록 메시지 처리 지연시간이 감소

HDD < SSD

`디스크 용량`

특정한 시점에 얼마나 많은 메시지들이 보전되어야 하는지 결정

10%의 오버헤드를 고려

`메모리`

디스크보다 시스템의 페이지 캐시에 저장되어 있는 메시지들을 컨슈머가 읽어오는 것이 가장 효율적

다른 애플리케이션과 함께 운영하여 페이지 캐시를 공유할 경우 컨슈머 성능 저하 가능성 존재

`네트워크`

사용 가능한 네트워크 대역폭은 카프카가 처리할 수 있는 트래픽의 최대량을 결정

`CPU`

카프카 클러스터를 매우 크게 확장하지 않는 한, 처리능력은 디스크, 메모리만큼 중요하지 않음

메시지의 압축 성능을 결정
<br>
  
#### 카프카 클러스터 설정하기

브로커 개수 결정 요소
* 디스크 용량
* 브로커당 레플리카 용량
* CPU 용량
* 네트워크 용량
* 장애 대응을 위한 여유 용량
* 확장성 요구사항

브로커 개수 계산 (기본 공식)
```text 
{디스크 용량} / {단일 브로커가 사용할 저장소 용량} * {복제 팩터 개수} = 최소 브로커 개수 
```

추가 고려사항
* 장애 대응을 위해 최소 3개 이상의 브로커 권장
* 토픽의 파티션 수와 복제 팩터에 따른 리더 분산 고려
* 네트워크 대역폭과 CPU 사용률 모니터링 필요
* 향후 확장 계획에 따른 여유 용량 확보
<br>
  
#### 브로커 설정

`필수 설정`
* 모든 브로커들이 동일한 zookeeper.connect 설정값을 가져야 함
* 카프카 클러스터 안의 모든 브로커가 유일한 broker.id 설정값을 가져야 함
* 모든 브로커의 listeners 설정이 올바르게 구성되어야 함

권장 설정
* auto.create.topics.enable: false (토픽 자동 생성 비활성화)
* log.retention.hours: 메시지 보관 기간 설정
* num.partitions: 기본 파티션 수 설정
* default.replication.factor: 기본 복제 팩터 설정
<br>
  
#### 운영체제 튜닝

`가상 메모리`

vm.swappiness 값을 1로 설정하여 스와핑 최소화

카프카는 시스템 페이지 캐시를 많이 사용하므로 스와핑이 발생하면 성능이 크게 저하됨

스와핑이 발생하면 페이지 캐시에 할당할 메모리가 부족해져 디스크 I/O가 증가

`디스크`

XFS 파일시스템 사용 권장 (ext4 대비 더 나은 성능)

디스크 마운트 옵션: noatime, nodiratime 설정

파일시스템 저널링 비활성화 고려 (성능 향상)

`네트워킹`

TCP 소켓 버퍼 크기 증가
* net.core.rmem.max = 16777216
* net.core.wmem.max = 16777216
* net.ipv4.tcp_rmem = 4096 87380 16777216
* net.ipv4.tcp_wmem = 4096 87380 16777216
