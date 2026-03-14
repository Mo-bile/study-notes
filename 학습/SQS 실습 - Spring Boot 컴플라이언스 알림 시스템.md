
# SQS 실습 - Spring Boot 컴플라이언스 알림 시스템

> 프로젝트: foreign-worker-compliance (외국인 근로자 법적 의무 관리)
> 실습 목적: SQS 핵심 개념을 실제 도메인에 적용하며 체득

관련 개념 노트: [[SQS (Amazon Simple Queue Service)]]

---

## 실습 시나리오

컴플라이언스 데드라인(비자 만료, 보험 가입 기한 등)이 임박하면 **SQS를 통해 비동기 알림**을 발송하는 시스템.

```
[ComplianceAlertService]     [SQS Main Queue]     [ComplianceAlertConsumer]
 (스케줄러가 데드라인 스캔)        │                    (알림 처리)
        │                        │                        │
        ├── 임박 데드라인 발견 ──▶ │ ── Long Polling ─────▶ │
        │   Producer.sendAlert() │                        ├── 멱등성 체크
        │                        │                        ├── 알림 발송
        │                        │                        └── Delete (자동)
        │                        │
        │                   [DLQ] ◀── 3회 실패 시 격리
```

**왜 SQS를 썼나?**
- 데드라인 스캔(무거운 조회)과 알림 발송(외부 API 호출)을 분리 → 스캔 속도와 알림 발송을 독립적으로 스케일
- 알림 서비스 장애 시에도 메시지가 큐에 보존 → 복구 후 자동 재처리

---

## 구현 파일 구조

```
foreign-worker-compliance/
├── docker-compose.yml                          # LocalStack (SQS 에뮬레이션)
├── init-sqs.sh                                 # 큐 초기화 스크립트
└── src/main/java/com/hr/fwc/
    └── infrastructure/
        ├── config/
        │   └── SqsConfig.java                  # 큐 이름 상수 + ObjectMapper
        └── messaging/
            ├── ComplianceAlertMessage.java      # 메시지 DTO (record)
            ├── ComplianceAlertProducer.java     # 메시지 발행
            ├── ComplianceAlertConsumer.java      # 메시지 소비 + 멱등성
            └── ProcessedMessageRepository.java  # 멱등성 저장소 (미완)
```

---

## Step별 실습 기록

### Step 0. 환경 세팅

**LocalStack + Docker Compose:**
```yaml
services:
  localstack:
    image: localstack/localstack:latest
    ports:
      - "4566:4566"
    environment:
      - SERVICES=sqs
```

**큐 초기화 (init-sqs.sh):**
```bash
# DLQ 먼저 생성 → Main Queue가 DLQ ARN을 참조
docker exec $CONTAINER awslocal sqs create-queue \
  --queue-name compliance-alert-dlq

# Main Queue + RedrivePolicy 연결
docker exec $CONTAINER awslocal sqs create-queue \
  --queue-name compliance-alert-queue \
  --attributes '{
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"DLQ_ARN\",\"maxReceiveCount\":\"3\"}",
    "VisibilityTimeout": "30",
    "ReceiveMessageWaitTimeSeconds": "20"
  }'
```

> **실습 포인트:** DLQ를 Main Queue보다 먼저 생성해야 한다. Main Queue의 RedrivePolicy가 DLQ의 ARN을 참조하기 때문.

**Gradle 의존성:**
```groovy
implementation platform('io.awspring.cloud:spring-cloud-aws-dependencies:3.1.1')
implementation 'io.awspring.cloud:spring-cloud-aws-starter-sqs'
```

---

### Step 1. SqsConfig — 설정 클래스

```java
@Configuration
public class SqsConfig {
    public static final String ALERT_QUEUE = "compliance-alert-queue";
    public static final String ALERT_DLQ = "compliance-alert-dlq";

    public ObjectMapper sqsObjectMapper() {
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.registerModule(new JavaTimeModule());
        return objectMapper;
    }
}
```

> **학습 포인트:** `JavaTimeModule` 등록을 안 하면 `LocalDate` 직렬화 시 `[2026, 3, 14]` 배열 형태로 나감. ISO 형식(`"2026-03-14"`)을 원하면 모듈 등록 필수.

---

### Step 2. ComplianceAlertMessage — 메시지 DTO

```java
public record ComplianceAlertMessage(
    String messageId,      // UUID — 멱등성 키
    Long workerId,
    Long deadlineId,
    DeadlineType deadlineType,
    DeadlineStatus status,
    LocalDate dueDate,
    String description
) {
    public static ComplianceAlertMessage from(...) {
        return new ComplianceAlertMessage(
            UUID.randomUUID().toString(), // 생성 시점에 고유 ID 부여
            workerId, deadlineId, ...);
    }
}
```

> **학습 포인트:** `messageId`를 Producer 측에서 생성하는 이유 — Consumer가 이 ID로 중복 체크(멱등성)를 수행. SQS의 MessageId와 별개로 비즈니스 레벨 멱등성 키가 필요.

---

### Step 3. ComplianceAlertProducer — 메시지 발행

```java
@Component
@ConditionalOnBean(SqsTemplate.class)  // SQS 미연결 환경에서 빈 등록 방지
public class ComplianceAlertProducer {

    private final SqsTemplate sqsTemplate;
    private final ObjectMapper objectMapper;

    public void sendAlert(ComplianceAlertMessage message) {
        String payload = objectMapper.writeValueAsString(message);
        sqsTemplate.send(ALERT_QUEUE, payload);
    }
}
```

> **학습 포인트:** `SqsTemplate.send(queueName, payload)` — Spring Cloud AWS가 내부적으로 큐 이름 → URL 변환, 직렬화, 전송을 처리. 개발자는 큐 이름과 페이로드만 신경 쓰면 됨.

---

### Step 4. ComplianceAlertConsumer — 메시지 소비

```java
@Component
@ConditionalOnBean(SqsTemplate.class)
public class ComplianceAlertConsumer {

    @SqsListener(ALERT_QUEUE)  // Long Polling 자동 적용
    public void handleAlert(String payload) {
        ComplianceAlertMessage message = objectMapper.readValue(payload, ...);
        // 정상 리턴 → Spring이 자동으로 Delete 호출
        // 예외 발생 → Delete 안 함 → Visibility Timeout 후 재노출
    }
}
```

> **학습 포인트:** `@SqsListener`의 동작 원리
> - 정상 리턴 = 처리 성공 → **자동 Delete** (메시지 영구 삭제)
> - 예외 throw = 처리 실패 → Delete 안 함 → Visibility Timeout 후 **재노출**
> - maxReceiveCount 초과 → **DLQ로 자동 이동**

---

### Step 5. 멱등성 처리 (진행 예정)

```java
@Repository
public class ProcessedMessageRepository {
    private final Set<String> processedIds = ConcurrentHashMap.newKeySet();

    public boolean isAlreadyProcessed(String messageId) {
        return processedIds.contains(messageId);
    }
    public void markAsProcessed(String messageId) {
        processedIds.add(messageId);
    }
}
```

Consumer에서 처리 전 중복 체크:
```java
if (processedMessageRepository.isAlreadyProcessed(message.messageId())) {
    log.info("중복 메시지 스킵: {}", message.messageId());
    return;  // 정상 리턴이므로 Delete 호출됨
}
processNotification(message);
processedMessageRepository.markAsProcessed(message.messageId());
```

> **학습 포인트:** 인메모리 Set은 학습용. 실무에서는 Redis TTL이나 DB unique constraint로 구현해야 서버 재시작 시에도 멱등성이 보장됨.

---

## 실습 중 만난 트러블슈팅

### 1. AWS CLI 미설치 → docker exec 전환

| 문제 | `aws: command not found` |
|------|--------------------------|
| 원인 | 로컬에 AWS CLI가 설치되지 않음 |
| 해결 | `docker exec $CONTAINER awslocal` 사용 — LocalStack 컨테이너 내부의 `awslocal` CLI 활용 |
| 배운 점 | LocalStack은 자체 CLI(`awslocal`)를 내장. 로컬 AWS CLI 설치 없이도 큐 관리 가능 |

### 2. SQS Auto-Configuration 테스트 실패

| 문제 | `SdkClientException` — 테스트 시 SQS 연결 시도 후 실패 |
|------|--------------------------------------------------------|
| 원인 | `spring-cloud-aws-starter-sqs`가 자동으로 SQS 클라이언트를 생성하려 함 |
| 해결 | 테스트 properties에 `spring.autoconfigure.exclude=io.awspring.cloud.autoconfigure.sqs.SqsAutoConfiguration` 추가 |
| 배운 점 | Spring Boot auto-configuration은 의존성만 있어도 동작함. 테스트 환경에서는 명시적 제외 필요 |

### 3. SqsTemplate 빈 부재로 Producer/Consumer 등록 실패

| 문제 | `NoSuchBeanDefinitionException: SqsTemplate` |
|------|-----------------------------------------------|
| 원인 | 테스트에서 SQS auto-config를 제외했으므로 SqsTemplate 빈이 없음 → Producer/Consumer 생성자 주입 실패 |
| 시도 | `@ConditionalOnClass(SqsTemplateAutoConfiguration.class)` → 클래스 자체가 존재하지 않아 실패 |
| 해결 | `@ConditionalOnBean(SqsTemplate.class)` — SqsTemplate 빈이 있을 때만 등록 |
| 배운 점 | `@ConditionalOnBean` vs `@ConditionalOnClass` 차이. Bean은 런타임 존재 여부, Class는 클래스패스 존재 여부. 외부 인프라 의존 컴포넌트에는 `@ConditionalOnBean`이 더 안전 |

---

## 핵심 개념 ↔ 실습 매핑

| 개념 | 이론 | 실습에서 체감한 것 |
|------|------|-------------------|
| At-least-once | 중복 전달 가능 | Consumer에 멱등성 로직을 안 넣으면 알림이 2번 갈 수 있음 |
| Visibility Timeout | 30초 동안 숨김 | `init-sqs.sh`에서 `VisibilityTimeout=30` 설정. 처리 시간 5초 × 6배 |
| DLQ | 3회 실패 → 격리 | `maxReceiveCount=3` + DLQ ARN 연결. DLQ를 먼저 만들어야 함 |
| Long Polling | 20초 대기 | `ReceiveMessageWaitTimeSeconds=20`. `@SqsListener`가 자동 적용 |
| Delete 메커니즘 | 명시적 삭제 필요 | Spring은 정상 리턴 시 자동 Delete. 예외 시 재노출 |
| 멱등성 | messageId로 중복 체크 | `ProcessedMessageRepository`로 구현. 인메모리 → 실무는 Redis |

---

## 면접 인출용 — 이 실습으로 답할 수 있는 질문들

**Q. SQS를 Spring Boot에서 어떻게 연동하나?**
→ `spring-cloud-aws-starter-sqs` + `@SqsListener` + `SqsTemplate.send()`. 자동 Delete, Long Polling, 에러 핸들링 모두 프레임워크가 처리.

**Q. 테스트 환경에서 외부 인프라 의존성은 어떻게 처리하나?**
→ `@ConditionalOnBean`으로 인프라 컴포넌트를 조건부 등록 + `spring.autoconfigure.exclude`로 auto-config 비활성화. 테스트는 인프라 없이 돌아가야 한다.

**Q. 멱등성을 왜, 어떻게 구현하나?**
→ SQS Standard는 At-least-once → 중복 가능. messageId 기반 중복 체크 로직 필수. 학습에서는 ConcurrentHashMap, 실무는 Redis SET + TTL 또는 DB unique constraint.

**Q. DLQ는 왜 Main Queue보다 먼저 생성하나?**
→ Main Queue의 RedrivePolicy가 DLQ의 ARN을 참조하기 때문. ARN이 없으면 정책 설정 자체가 실패.

**Q. `@ConditionalOnBean` vs `@ConditionalOnClass` 차이?**
→ `OnClass`는 클래스패스에 클래스가 있는지, `OnBean`은 스프링 컨텍스트에 빈이 등록되었는지. 외부 인프라 컴포넌트는 클래스는 있지만 빈이 없을 수 있으므로 `OnBean`이 더 안전.

---

## 남은 실습

- [ ] Step 5: `ProcessedMessageRepository` + Consumer 멱등성 적용
- [ ] Step 6: `ComplianceAlertService` — 스케줄러로 데드라인 스캔 + Producer 호출
- [ ] Step 7: `AlertTestController` — 수동 테스트 엔드포인트
- [ ] 통합 테스트: docker-compose up → 큐 생성 → API 호출 → 로그 확인
