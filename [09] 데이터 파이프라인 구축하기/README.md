## 카프카 핵심 가이드

### 데이터 파이프라인 구축하기

데이터 파이프라인은 한 시스템에서 다른 시스템으로 데이터를 안정적으로 이동시키는 흐름

카프카는 파이프라인의 중간 버퍼이자 전달 계층으로 동작

소스와 싱크를 직접 연결하지 않고 카프카를 사이에 두면 생산자와 소비자를 느슨하게 결합할 수 있음

<br>

### 카프카가 파이프라인에 적합한 이유

카프카를 파이프라인 중심에 두면 같은 원천 데이터를 여러 목적지에서 독립적으로 사용할 수 있음

각 소비자는 자신에게 필요한 속도와 방식으로 데이터를 읽음

소스 시스템 장애와 싱크 시스템 장애를 직접 연결하지 않고 완충 가능

<br>

핵심 장점:
- 내구성 있는 로그 저장
- 높은 처리량
- 여러 소비자 그룹의 독립적인 소비
- 재처리 가능성
- 보안/권한/감사 기능과 결합 가능

<br>

### 파이프라인 설계 고려사항

좋은 데이터 파이프라인은 단순히 데이터를 복사하는 수준을 넘어서야 함

운영 관점에서는 지연시간, 신뢰성, 처리량, 데이터 형식, 보안, 장애 처리를 함께 설계해야 함

<br>

### 적시성

`timeliness`: 데이터가 생성된 뒤 목적지에 도착하기까지 허용되는 시간

파이프라인마다 요구사항은 다름

일부는 수 밀리초/초 단위의 near-real-time을 요구

일부는 시간 단위 또는 일 단위 배치로 충분

<br>

카프카 기반 파이프라인은 같은 데이터라도 소비자별로 다른 적시성 요구사항을 지원할 수 있음

<br>

### 신뢰성

데이터 파이프라인은 분석, 정산, 운영 판단의 입력이 되므로 유실이 큰 문제로 이어질 수 있음

최소한 at-least-once 전달을 목표로 설계

소스/싱크 시스템의 의미론까지 맞으면 exactly-once에 가까운 파이프라인도 가능

<br>

Kafka Connect는 소스 오프셋 저장 API를 제공하여 커넥터가 장애 후 이어서 읽을 수 있게 함

단, end-to-end exactly-once는 소스와 싱크의 특성까지 함께 검증해야 함

<br>

### 처리량

파이프라인은 평균 처리량뿐 아니라 순간적인 처리량 증가도 견뎌야 함

카프카는 소스와 싱크의 처리 속도를 분리함

싱크가 잠시 느려져도 데이터는 카프카에 쌓이고, 싱크는 나중에 따라잡을 수 있음

<br>

확장 포인트:
- 토픽 파티션 수
- 커넥터 task 수
- Connect worker 수
- 싱크 시스템의 쓰기 처리량

<br>

### 데이터 형식

파이프라인은 데이터 값뿐 아니라 스키마도 함께 고려해야 함

스키마를 잃으면 목적지 시스템에서 데이터 의미를 해석하기 어려워짐

스키마 진화를 지원하지 못하면 소스 변경이 곧 파이프라인 장애로 이어질 수 있음

<br>

Kafka Connect는 내부적으로 스키마와 값을 가진 Connect Data API 모델을 사용

실제 Kafka 레코드 형식은 converter가 결정

대표 형식:
- JSON
- Avro
- Protobuf
- JSON Schema

<br>

### 변환

파이프라인 변환 방식은 크게 두 가지로 나뉨

`ETL`: Extract -> Transform -> Load
- 파이프라인 중간에서 데이터를 변환
- 목적지에는 이미 가공된 데이터 저장

`ELT`: Extract -> Load -> Transform
- 원천에 가까운 데이터를 먼저 저장
- 목적지나 별도 처리 계층에서 변환

<br>

카프카 기반 파이프라인은 원천 데이터를 보존하고 여러 소비자가 각자 변환하는 ELT 방식과 잘 맞음

단순 필드 제거/추가 같은 가벼운 변환은 Kafka Connect의 SMT(Single Message Transformation)로 처리 가능

<br>

### 보안

파이프라인은 시스템 경계를 넘는 경우가 많기 때문에 보안 요구사항이 중요

확인할 항목:
- 소스 시스템 접근 권한
- Kafka 인증/인가
- 싱크 시스템 접근 권한
- 전송 중 암호화
- 민감 데이터 마스킹
- 커넥터 설정에 포함되는 credential 관리

<br>

Kafka Connect worker와 connector는 외부 시스템 credential을 다루므로 설정 파일과 REST API 접근도 보호해야 함

<br>

### 장애 처리

파이프라인 장애는 다양한 위치에서 발생 가능

대표 원인:
- 소스 시스템 장애
- 싱크 시스템 장애
- 잘못된 레코드 형식
- 스키마 불일치
- 네트워크 장애
- 커넥터 설정 오류

<br>

장애 처리 정책:
- 재시도
- 중단 후 운영자 개입
- 오류 레코드 스킵
- dead letter queue로 분리
- 모니터링/알림

<br>

장애 레코드를 조용히 버리는 정책은 데이터 품질 문제를 숨길 수 있으므로 주의

<br>

### 결합도와 민첩성

소스와 싱크를 직접 연결한 ad hoc 파이프라인은 빠르게 만들 수 있지만 장기적으로 결합도가 커짐

소스 스키마가 바뀔 때마다 여러 싱크 파이프라인을 동시에 수정해야 할 수 있음

카프카를 중간 계층으로 두면 소스는 한 번만 데이터를 발행하고, 여러 싱크는 독립적으로 소비 가능

<br>

스키마 정보와 스키마 진화를 보존하는 것이 결합도를 낮추는 핵심

<br>

### Kafka Connect를 쓸 때

Kafka Connect는 카프카와 외부 시스템을 연결하기 위한 표준 프레임워크

이미 사용 가능한 connector가 있다면 직접 producer/consumer 애플리케이션을 작성하는 것보다 Connect를 우선 고려

<br>

Kafka Connect가 적합한 경우:
- 외부 시스템에서 Kafka로 데이터를 가져오는 source pipeline
- Kafka에서 외부 시스템으로 데이터를 내보내는 sink pipeline
- 여러 커넥터를 표준 방식으로 운영해야 하는 경우
- REST API로 배포/중지/상태 확인이 필요한 경우
- task 분산과 장애 복구를 프레임워크에 맡기고 싶은 경우

<br>

직접 producer/consumer가 더 나은 경우:
- 복잡한 애플리케이션 로직이 필요
- 커넥터가 존재하지 않음
- 데이터 이동보다 도메인 처리 로직이 핵심
- Connect의 추상화가 오히려 제약이 되는 경우

<br>

### Kafka Connect 개요

Kafka Connect는 worker 프로세스 위에서 connector plugin을 실행

connector는 데이터 시스템과 연결하는 논리 단위

task는 실제 데이터 복사 작업을 수행하는 실행 단위

worker는 connector와 task를 실행하고, 상태와 설정을 Kafka 내부 토픽에 저장

<br>

구성 요소:
- source connector: 외부 시스템 -> Kafka
- sink connector: Kafka -> 외부 시스템
- worker: connector/task 실행 컨테이너
- converter: Connect 데이터와 Kafka 레코드 포맷 사이 변환
- transform: 단일 메시지 단위 변환

<br>

### Connect worker 설정

주요 worker 설정:

- `bootstrap.servers`
- `group.id`
- `plugin.path`
- `key.converter`
- `value.converter`
- `config.storage.topic`
- `offset.storage.topic`
- `status.storage.topic`

<br>

`group.id`가 같은 worker들은 하나의 Connect cluster를 구성

`plugin.path`는 connector, converter, transformation plugin을 찾는 경로

converter는 Kafka에 저장되는 실제 바이트 형식을 결정

내부 토픽은 connector 설정, 오프셋, 상태 저장에 사용되므로 복제와 보존 정책을 신중히 설정해야 함

<br>

### distributed mode와 standalone mode

`distributed mode`
- 여러 worker가 하나의 Connect cluster 구성
- connector/task가 worker 사이에 분산
- worker 장애 시 다른 worker가 작업을 이어받음
- 운영 환경에 적합

<br>

`standalone mode`
- 하나의 worker에서 connector/task 실행
- 특정 장비에 묶여야 하는 간단한 작업에 사용 가능
- 운영 확장성과 장애 복구 측면에서는 제한적

<br>

### Connect REST API

Kafka Connect는 REST API로 connector를 관리

대표 작업:
- connector 목록 조회
- connector 생성/수정
- connector 상태 조회
- connector 삭제
- connector plugin 목록 조회

<br>

예시:
```bash
curl http://localhost:8083/connector-plugins
curl http://localhost:8083/connectors
curl http://localhost:8083/connectors/{connector-name}/status
```

<br>

REST API는 운영 제어면이므로 인증, 네트워크 접근 제어, 변경 이력 관리가 필요

<br>

### 커넥터 예제 흐름

책의 예제는 단순한 FileStream connector로 Connect 동작을 먼저 확인

File source는 파일 내용을 Kafka 토픽으로 쓰고, File sink는 Kafka 토픽 내용을 다시 파일로 씀

이 예제는 학습용으로 적합하지만 운영 파이프라인에는 신뢰성/확장성 한계가 있음

<br>

실제 파이프라인 예시는 JDBC source와 Elasticsearch sink 조합

JDBC source connector가 MySQL 데이터를 Kafka로 가져오고, Elasticsearch sink connector가 Kafka 토픽을 검색 시스템에 기록

CDC가 필요한 데이터베이스 파이프라인은 단순 polling 방식보다 Debezium 같은 change data capture connector를 고려

<br>

### Source connector와 Sink connector

`Source connector`
- 외부 시스템에서 데이터를 읽음
- Connect record로 변환해 worker에 전달
- worker가 Kafka producer를 통해 토픽에 기록
- 소스 offset을 저장해 장애 후 이어 읽기 가능

<br>

`Sink connector`
- Kafka 토픽에서 데이터를 읽음
- converter를 통해 Connect record로 변환
- 외부 시스템에 기록
- 성공한 Kafka offset을 커밋

<br>

커넥터 개발자는 데이터 이동 로직에 집중하고, worker는 공통 실행/분산/오프셋 관리를 담당

<br>

### Converter

converter는 Connect Data API와 Kafka에 저장되는 직렬화 형식 사이를 변환

source 방향:
```text
External system -> Connect record -> converter -> Kafka record
```

sink 방향:
```text
Kafka record -> converter -> Connect record -> External system
```

<br>

converter 선택은 커넥터와 독립적

같은 JDBC source connector라도 JSON, Avro, Protobuf 등 다른 형식으로 Kafka에 저장 가능

스키마 관리가 중요하면 Schema Registry와 함께 Avro/Protobuf/JSON Schema 계열을 고려

<br>

### Offset 관리

Kafka Connect는 source connector가 외부 시스템의 위치 정보를 저장할 수 있게 지원

예시:
- 파일 source: 파일명과 위치
- JDBC source: 증가 컬럼이나 timestamp
- CDC source: 로그 위치

<br>

source task가 레코드를 반환할 때 source partition과 source offset 정보를 함께 제공

worker는 Kafka 쓰기가 성공하면 offset을 저장

이 구조가 장애 후 재시작과 at-least-once 전달의 기반

<br>

sink connector는 Kafka consumer offset을 기준으로 처리 위치를 관리

외부 시스템에 쓰기 성공 후 offset을 커밋해야 중복/유실 위험을 줄일 수 있음

<br>

### SMT

SMT(Single Message Transformation)는 Kafka Connect 안에서 수행하는 단일 레코드 변환

가벼운 변환에 적합:
- 필드 추가/삭제
- 토픽명 변경
- 헤더 추가
- key/value 변환
- 라우팅

<br>

SMT는 상태를 갖는 복잡한 처리에는 적합하지 않음

조인, 집계, 윈도우 처리 같은 작업은 Kafka Streams, Flink 같은 스트림 처리 계층을 고려

<br>

### 오류 처리와 DLQ

커넥터는 일부 레코드만 실패할 수 있음

운영에서는 실패 정책을 명확히 해야 함

<br>

선택지:
- 실패 시 connector 중단
- 실패 레코드 스킵
- 실패 레코드를 dead letter queue로 전송
- 실패 원인을 로그/메트릭으로 기록

<br>

DLQ를 사용하면 전체 파이프라인을 멈추지 않고 문제 레코드를 별도로 분석 가능

하지만 DLQ가 있다는 이유로 데이터 품질 검증을 생략하면 안 됨

<br>

### 대안

Kafka Connect 외에도 데이터 파이프라인을 만드는 방법은 있음

- 직접 producer/consumer 애플리케이션 작성
- Kafka Streams
- Flink
- Spark
- 기존 ETL 도구
- CDC 전문 도구

<br>

선택 기준:
- 연결해야 하는 시스템 수
- 변환 복잡도
- 운영 표준화 필요성
- 장애 복구 요구사항
- 기존 connector 생태계 활용 가능성

<br>

### 운영 관점 체크 포인트

- connector가 이미 있으면 Kafka Connect 우선 검토
- 운영은 distributed mode 기준으로 설계
- worker 내부 토픽은 복제와 ISR 설정 확인
- `plugin.path`와 connector 의존성 충돌 관리
- converter와 schema 전략을 먼저 결정
- source offset과 sink offset 커밋 의미 확인
- 오류 정책과 DLQ 운영 절차 정의
- REST API 접근 통제
- connector/task 상태와 lag 모니터링

<br>
