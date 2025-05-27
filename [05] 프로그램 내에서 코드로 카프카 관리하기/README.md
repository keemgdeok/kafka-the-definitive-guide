## 카프카 핵심 가이드

### AdminClient 개요

- 카프카 클러스터 관리를 위한 API
- **비동기적 API**: 대부분의 작업은 `Future` 객체를 반환
- **최종적 일관성**: 컨트롤러에서 브로커로 메타데이터 전파는 비동기적. API 호출 시점과 실제 클러스터 상태 반영 시점에 차이가 있을 수 있음
- **옵션 객체**: 각 AdminClient 메서드는 옵션 객체(예: `ListTopicsOptions`)를 통해 타임아웃 등의 세부 동작 지정 가능
- **클러스터 상태 변경 작업**(생성, 삭제, 변경): 컨트롤러 노드에서 수행
- **클러스터 상태 읽기 작업**(목록 조회, 상세 조회): 모든 브로커에서 수행 가능 (일반적으로 부하가 적은 브로커)

<br>

### AdminClient 사용법: 생성, 설정, 닫기

```java
// 1. AdminClient 생성 (필수 설정: bootstrap.servers)
Properties props = new Properties();
props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
// props.put(AdminClientConfig.REQUEST_TIMEOUT_MS_CONFIG, 30000); // 요청 타임아웃 (기본값 120초)

AdminClient admin = AdminClient.create(props);

// 2. AdminClient 사용 (예: 토픽 목록 조회)
try {
    ListTopicsResult topics = admin.listTopics();
    topics.names().get().forEach(System.out::println);
} catch (InterruptedException | ExecutionException e) {
    e.printStackTrace();
} finally {
    // 3. AdminClient 닫기 (리소스 해제)
    admin.close(Duration.ofSeconds(30)); // 타임아웃 지정
}
```

<br>

### 주요 토픽 관리 기능

**1. 토픽 목록 조회**
```java
ListTopicsResult topics = admin.listTopics();
Set<String> topicNames = topics.names().get(); // get()은 블로킹 호출
System.out.println("topic list: " + topicNames);
```

**2. 토픽 생성**
```java
String topicName = "my-new-topic";
int numPartitions = 3;
short replicationFactor = 1;
NewTopic newTopic = new NewTopic(topicName, numPartitions, replicationFactor);

CreateTopicsResult result = admin.createTopics(Collections.singleton(newTopic));
result.all().get(); // 생성 완료까지 대기 (블로킹)
System.out.println(topicName + " topic creation complete");
```

**3. 토픽 상세 정보 조회**
```java
DescribeTopicsResult demoTopic = admin.describeTopics(Collections.singletonList("my-topic"));
TopicDescription topicDescription = demoTopic.values().get("my-topic").get();
System.out.println("Topic description: " + topicDescription);
```

**4. 토픽 삭제**
```java
DeleteTopicsResult deleteResult = admin.deleteTopics(Collections.singletonList("my-old-topic"));
deleteResult.all().get(); // 삭제 완료까지 대기 (블로킹)
System.out.println("my-old-topic topic deletion complete");
```

**비동기 처리 예시 (`whenComplete`)**
```java
admin.describeTopics(Collections.singletonList("another-topic"))
    .values().get("another-topic")
    .whenComplete((description, throwable) -> {
        if (throwable != null) {
            System.err.println("Failed to describe topic: " + throwable.getMessage());
        } else {
            System.out.println("Async topic description: " + description);
        }
    });
// whenComplete는 논블로킹, 이후 작업은 다른 스레드에서 실행될 수 있음
```

<br>

### 설정 관리 (토픽/브로커)

- `ConfigResource` 객체로 관리 대상(토픽, 브로커) 지정
- `describeConfigs()`: 설정 조회
- `incrementalAlterConfigs()` 또는 `alterConfigs()` (Deprecated): 설정 변경

**토픽 설정 조회 예시**
```java
ConfigResource topicResource = new ConfigResource(ConfigResource.Type.TOPIC, "my-topic");
DescribeConfigsResult configsResult = admin.describeConfigs(Collections.singleton(topicResource));
Map<ConfigResource, Config> configs = configsResult.all().get();

configs.get(topicResource).entries().forEach(entry -> {
    if (!entry.isDefault()) { // 기본값이 아닌 설정만 출력
        System.out.println(entry.name() + " = " + entry.value());
    }
});
```

**토픽 설정 변경 예시 (압축 정책 변경)**
```java
ConfigResource topicResource = new ConfigResource(ConfigResource.Type.TOPIC, "my-topic");
ConfigEntry compactionEntry = new ConfigEntry(TopicConfig.CLEANUP_POLICY_CONFIG, TopicConfig.CLEANUP_POLICY_COMPACT);
AlterConfigOp alterOp = new AlterConfigOp(compactionEntry, AlterConfigOp.OpType.SET);

Map<ConfigResource, Collection<AlterConfigOp>> alterConfigs = new HashMap<>();
alterConfigs.put(topicResource, Collections.singletonList(alterOp));

admin.incrementalAlterConfigs(alterConfigs).all().get();
System.out.println("my-topic compaction policy update complete");
```

<br>

### 컨슈머 그룹 관리

**1. 컨슈머 그룹 목록 조회**
```java
ListConsumerGroupsResult groupsResult = admin.listConsumerGroups();
groupsResult.valid().get().forEach(group -> System.out.println("Group ID: " + group.groupId()));
```

**2. 컨슈머 그룹 상세 정보 조회**
```java
String groupId = "my-consumer-group";
DescribeConsumerGroupsResult groupResult = admin.describeConsumerGroups(Collections.singletonList(groupId));
ConsumerGroupDescription groupDescription = groupResult.describedGroups().get(groupId).get();
System.out.println("Group details: " + groupDescription);
// 멤버 정보, 할당된 파티션, 코디네이터 정보 등 포함
```

**3. 컨슈머 그룹 오프셋 조회 및 Lag 계산**
```java
String groupId = "my-consumer-group";
Map<TopicPartition, OffsetAndMetadata> committedOffsets = admin.listConsumerGroupOffsets(groupId)
                                                              .partitionsToOffsetAndMetadata().get();

Map<TopicPartition, OffsetSpec> latestOffsetSpecs = new HashMap<>();
committedOffsets.keySet().forEach(tp -> latestOffsetSpecs.put(tp, OffsetSpec.latest()));
Map<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo> latestOffsets = admin.listOffsets(latestOffsetSpecs).all().get();

committedOffsets.forEach((tp, offsetAndMetadata) -> {
    long committed = offsetAndMetadata.offset();
    long latest = latestOffsets.get(tp).offset();
    long lag = latest - committed;
    System.out.println("Group: " + groupId + ", Topic: " + tp.topic() + 
                       ", Partition: " + tp.partition() + ", Committed Offset: " + committed +
                       ", Latest Offset: " + latest + ", Lag: " + lag);
});
```

**4. 컨슈머 그룹 오프셋 변경 (주의해서 사용)**
- 컨슈머 그룹이 활성 상태가 아닐 때 주로 사용 (데이터 유실 또는 중복 처리 가능성).
```java
String groupId = "my-consumer-group";
TopicPartition tp = new TopicPartition("my-topic", 0);
OffsetAndMetadata newOffset = new OffsetAndMetadata(100L); // 특정 오프셋으로 변경

Map<TopicPartition, OffsetAndMetadata> offsetsToReset = new HashMap<>();
offsetsToReset.put(tp, newOffset);

try {
    admin.alterConsumerGroupOffsets(groupId, offsetsToReset).all().get();
    System.out.println(groupId + " offset change complete");
} catch (ExecutionException e) {
    if (e.getCause() instanceof UnknownMemberIdException) {
        System.err.println("Check if consumer group is still active.");
    } else {
        e.printStackTrace();
    }
}
```

<br>

### 클러스터 메타데이터 조회

```java
DescribeClusterResult cluster = admin.describeCluster();
System.out.println("Cluster ID: " + cluster.clusterId().get());
System.out.println("Broker list:");
cluster.nodes().get().forEach(node -> System.out.println(" - " + node));
System.out.println("Controller: " + cluster.controller().get());
```

<br>

### 고급 어드민 작업 (개념 요약)

- **토픽에 파티션 추가**: `createPartitions()` 사용, 기존 파티션 키 배분 방식에 영향 줄 수 있으므로 주의
- **토픽에서 레코드 삭제**: `deleteRecords()` 사용, 특정 오프셋 이전 레코드 삭제 표시 (실제 디스크 삭제는 비동기)
- **리더 선출**: `electLeaders()` 사용, 선호 리더 선출(PREFERRED) 또는 언클린 리더 선출(UNCLEAN - 데이터 유실 가능성)
- **레플리카 재할당**: `alterPartitionReassignments()` 사용, 브로커 간 데이터 이동 발생

<br>

### 테스트하기 (MockAdminClient)

- `MockAdminClient`를 사용하여 실제 클러스터 없이 AdminClient 로직 테스트 가능

```java
// 테스트 클래스 예시
public class TopicCreator {
    private AdminClient admin;
    public TopicCreator(AdminClient admin) { this.admin = admin; }

    public void maybeCreateTestTopic(String topicName) throws Exception {
        if (topicName.toLowerCase().startsWith("test")) {
            admin.createTopics(Collections.singletonList(
                new NewTopic(topicName, 1, (short) 1))).all().get();
        }
    }
}

// JUnit 테스트 예시
/*
@Test
public void testCreateTestTopicWithMock() throws Exception {
    Node broker = new Node(0, "localhost", 9092);
    // MockAdminClient spy/mock 설정은 복잡할 수 있어 간략화
    // 실제 사용 시에는 필요한 메서드 mock 필요
    try (AdminClient mockAdmin = new MockAdminClient(Collections.singletonList(broker), broker)) {
        TopicCreator tc = new TopicCreator(mockAdmin);
        String testTopicName = "test.new.topic";
        tc.maybeCreateTestTopic(testTopicName);

        // 생성된 토픽 확인 (MockAdminClient는 내부적으로 상태를 가짐)
        Set<String> topics = mockAdmin.listTopics().names().get();
        assertTrue(topics.contains(testTopicName));
    }
}
*/
