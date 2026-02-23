## 카프카 핵심 가이드

### 카프카 내부 매커니즘

카프카의 고성능/고가용성은 아래 4가지로 설명할 수 있음:
- 메타데이터/컨트롤러(클러스터 상태 관리)
- 복제(Replication, ISR/HW)
- 요청 처리(Request Processing)
- 로그 저장 구조(세그먼트/보존 정책)

<br>

#### 핵심 용어

- `LEO(Log End Offset)`: 레플리카가 가진 마지막 오프셋(+1)

<img src="./img/Kafka_%20The%20Definitive%20Guide_page175_image.png" width="80%" height="80%"> <br>

- `HW(High Watermark)`: 컨슈머에게 "커밋된" 것으로 노출되는 경계(일반적으로 ISR에 복제된 지점)

<br>

### 클러스터 멤버십 & 컨트롤러

- ZooKeeper 기반(전통적)
  - 브로커는 `/brokers/ids`에 임시 노드(Ephemeral Node)로 등록
  - 컨트롤러는 `/controller` 임시 노드로 선출
  - `controller epoch`로 좀비 컨트롤러(Zombie fencing) 방지
- KRaft 기반(개념)
  - 주키퍼를 제거하고, Raft 기반 컨트롤러 쿼럼이 메타데이터를 로그로 관리
- 컨트롤러 역할
  - 파티션 리더 선출(Leader Election)
  - ISR 변경, 레플리카 재할당, 장애 브로커 대응
  - 토픽/파티션/설정 변경 반영(05 AdminClient 작업과 연결)

<br>

### 복제(Replication) & ISR

- 레플리카 구성
  - `Leader Replica`: 쓰기(Produce)는 리더에만 append(03장)
  - `Follower Replica`: 리더를 Fetch해서 로그를 따라감
- `ISR(In-Sync Replicas)`
  - 리더와 충분히 동기화된 레플리카 집합
  - 뒤처지면 ISR에서 제외되고, 복구하면 다시 포함
  - 관련 설정: `replica.lag.time.max.ms`, `min.insync.replicas`(02장)
- `acks` 트레이드오프(03장)
  - `acks=1`: 리더 기록 후 응답(팔로워 복제는 비동기)
  - `acks=all(-1)`: ISR에 `min.insync.replicas` 조건을 만족할 만큼 복제된 뒤 응답
- `unclean leader election`
  - ISR에 리더 후보가 없을 때 ISR 밖 레플리카를 리더로 올리면 가용성은 올라가지만 데이터 유실 가능

<br>

### 요청 처리(Request Processing)

- 카프카는 TCP 바이너리 프로토콜로 요청을 처리

<img src="./img/Kafka_%20The%20Definitive%20Guide_page172_image.png" width="80%" height="80%"> <br>

- 메타데이터 요청 흐름(개념)
  - 클라이언트는 임의의 브로커에 `Metadata request`를 보내고, 응답으로 리더 브로커 위치 등을 갱신
  - 이후 Produce/Fetch 요청은 해당 파티션의 리더 브로커로 라우팅

<img src="./img/Kafka_%20The%20Definitive%20Guide_page171_image.png" width="80%" height="80%"> <br>

- 스레드 모델(개념)
  - `Acceptor`(연결 수락) → `Network/Processor`(요청 수신/응답 전송) → `Request Handler(I/O)`(요청 처리)

- `Purgatory(지연 응답 버퍼)`
  - 조건을 만족할 때까지 대기해야 하는 요청을 임시 보관(예: `acks=all`, `fetch.min.bytes`)
- Produce 요청(요약)
  - 리더 검증(권한/ISR/크기 등) → 로그 append → 복제 진행 → acks 조건 충족 시 응답

<img src="./img/Kafka_%20The%20Definitive%20Guide_page174_image.png" width="80%" height="80%"> <br>

- Fetch 요청(요약)
  - 브로커는 조건(`fetch.min.bytes`, `fetch.max.wait.ms`)에 따라 잠시 대기한 뒤 응답할 수 있음
  - 기본적으로 `HW`를 넘는 레코드는 노출되지 않음
  - 성능 최적화를 위해 OS 페이지 캐시/제로 카피 전송을 적극 활용

<br>

### 물리적 저장소(로그) 구조

- 토픽-파티션은 브로커 디스크에 append-only 로그로 저장(`log.dirs`)

<img src="./img/Kafka_%20The%20Definitive%20Guide_page180_image.png" width="80%" height="80%"> <br>

- 랙 인식(Rack-awareness)
  - 동일한 파티션 레플리카가 같은 랙에 몰리지 않도록 배치(랙 단위 장애 대비)

- 세그먼트(Segment)
  - 파티션 로그는 여러 세그먼트 파일로 쪼개짐
  - 롤링 기준: `log.segment.bytes`, `log.roll.ms`(02장)
- 보존(retention)
  - 시간/용량 기준으로 오래된 세그먼트를 삭제: `log.retention.ms`, `log.retention.bytes`
- 인덱스
  - 오프셋 인덱스/타임 인덱스를 유지(오프셋→파일 위치 매핑)

<br>

### 보존 정책: Delete vs Compact

- `cleanup.policy=delete`
  - retention 기준에 따라 세그먼트 단위로 삭제
- `cleanup.policy=compact`

  <img src="./img/Kafka_%20The%20Definitive%20Guide_page185_image.png" width="80%" height="80%"> <br>
  - 키 기준으로 최신 값만 남기고 과거 레코드를 정리

  <img src="./img/Kafka_%20The%20Definitive%20Guide_page186_image.png" width="80%" height="80%"> <br>
  - 컴팩션 중에는 "정리된 구간(clean)" / "아직 정리 안 된 구간(dirty)"로 나뉘어 동작
  - Tombstone(값이 null인 레코드)로 "키 삭제"를 표현할 수 있음(일정 시간 이후 완전 제거)

<br>

### (옵션) 계층형 스토리지(Tiered Storage)

- 로컬 계층(빠름) + 원격 스토리지(저렴/대용량)를 조합해 retention을 늘리는 접근
- 최신 데이터는 로컬, 오래된 데이터는 원격에 보관하는 형태로 이해

<br>

### 운영 관점 체크 포인트

- 내구성: `acks=all` + `min.insync.replicas` 조합, ISR 축소/복구 모니터링
- 가용성: 리더 선출/언클린 리더 선출 정책의 트레이드오프 이해
- 성능: 네트워크/디스크/페이지 캐시, Request Handler 적체 여부
- 저장소: 세그먼트 롤링/retention/컴팩션 정책이 디스크 사용량에 미치는 영향
