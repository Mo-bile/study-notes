
# SQS (Amazon Simple Queue Service) 핵심 정리

## 한 줄 정의
AWS의 완전 관리형 메시지 큐 서비스. Producer가 메시지를 넣고, Consumer가 꺼내서 처리하는 **비동기 통신** 인프라.

## 왜 쓰는가?
- 동기 처리 시 모든 작업이 끝날 때까지 클라이언트가 대기 → 느림, 장애 전파
- 메시지 큐로 후속 작업(알림, 보험판정 등)을 분리하면 **즉시 응답 + 독립 배포/스케일** 가능
- 장애 격리: Consumer가 죽어도 메시지는 큐에 남아있어 데이터 유실 없음

---

# 핵심 개념

## 1) Delivery Semantics (메시지 전달 보장)

**정의:**
메시지가 Consumer에게 몇 번 전달되는가에 대한 보장 수준.

**3가지 수준:**

| 보장 수준 | 의미 | 트레이드오프 |
|-----------|------|-------------|
| At-most-once | 최대 1번. 유실 가능 | 빠르지만 메시지 손실 위험 |
| At-least-once | 최소 1번. 중복 가능 | 안전하지만 중복 처리 대비 필요 |
| Exactly-once | 정확히 1번 | 구현 매우 어려움, 성능 비용 큼 |

**실무 포인트:**
- SQS Standard Queue = **At-least-once**
- 중복 발생 원리: SQS가 메시지를 여러 서버에 복제 저장 → 내부 동기화 지연 시 같은 메시지를 2번 반환할 수 있음
- 따라서 Consumer 측 **멱등성 처리가 필수**

**면접 포인트:**
"SQS는 왜 exactly-once가 아닌가?" → 분산 시스템에서 메시지를 여러 서버에 복제 저장하기 때문. 복제 간 동기화 지연으로 중복 전달 가능. FIFO Queue는 deduplication으로 exactly-once에 근접하지만 처리량 제한(300 TPS)이 있음.

---

## 2) Visibility Timeout

**정의:**
Consumer가 메시지를 가져간 후, 다른 Consumer에게 해당 메시지가 **보이지 않는 시간**.

**동작 원리:**

```
[메시지 큐에 도착]
     │
     ▼
[Consumer A가 Receive] ──── Visibility Timeout (30초 기본) ────
     │                                                        │
     ├── 처리 성공 → Delete 호출 → 메시지 영구 삭제 ✅          │
     │                                                        │
     └── 처리 실패 (또는 Delete 안 함)                          │
                                          타임아웃 만료 ◀──────┘
                                               │
                                               ▼
                                    [메시지 다시 visible → 재처리]
```

**실무 포인트:**
- Consumer가 메시지를 받으면 큐에서 **삭제되는 게 아님** — "안 보이게" 될 뿐
- 처리 완료 후 **명시적으로 Delete를 호출**해야 영구 삭제
- 설정 공식: `Visibility Timeout = 예상 처리 시간 × 6`
- 너무 짧으면: 처리 중인데 다른 Consumer가 중복 처리
- 너무 길면: 실패한 메시지가 오래 묶여있어 재처리 지연

---

## 3) Dead Letter Queue (DLQ)

**정의:**
반복 실패한 메시지를 **격리**하는 별도 큐.

**동작 원리:**

```
[Main Queue]
  메시지 도착 → Consumer 처리 실패 (1회) → 재노출
                Consumer 처리 실패 (2회) → 재노출
                Consumer 처리 실패 (3회 = maxReceiveCount)
                    │
                    └──── 자동 이동 ──▶ [DLQ에 격리]
                                          → 운영자 확인/수동 재처리
```

**실무 포인트:**
- `maxReceiveCount`로 최대 재시도 횟수 설정 (보통 3~5회)
- DLQ가 없으면 실패 메시지가 무한 재시도 → 리소스 낭비 + 장애 원인 파악 불가
- DLQ 모니터링은 운영 필수 — CloudWatch 알람 연동

**면접 포인트:**
"DLQ에 쌓인 메시지는 어떻게 처리하나?" → 원인 분석 후 수동 재처리(redrive) 또는 폐기. AWS 콘솔에서 DLQ redrive 기능 제공. 근본 원인을 먼저 수정하고 재처리해야 같은 실패 반복을 방지.

---

## 4) Idempotency (멱등성)

**정의:**
같은 메시지를 여러 번 처리해도 **결과가 동일**한 것.

**문제:**
At-least-once이므로 같은 메시지가 2번 올 수 있음. 멱등성 없으면 알림 2번 발송, 결제 2번 처리 등 사고 발생.

**해결:**
```
void process(Message msg) {
    if (이미_처리됨(msg.id)) return;  // 중복 스킵
    실제_처리(msg);
    처리_완료_기록(msg.id);           // Redis or DB에 기록
}
```

**면접 포인트:**
"처리 후 기록 사이에 서버가 죽으면?" → 재시작 후 같은 메시지를 다시 받아서 한 번 더 처리됨. 이것이 at-least-once의 본질. 실무에서는 처리+기록을 DB 트랜잭션 안에서 원자적으로 수행하거나, 처리 로직 자체를 멱등하게 설계.

---

## 5) Standard Queue vs FIFO Queue

**정의:**
SQS가 제공하는 두 가지 큐 유형.

| 항목 | Standard | FIFO |
|------|----------|------|
| 순서 보장 | X | O |
| 처리량 | 무제한 | 300 TPS (배치 시 3000) |
| 중복 가능성 | 있음 | 제거 가능 (deduplication) |
| 큐 이름 | 자유 | `.fifo` 접미사 필수 |

**실무 포인트:**
- 선택 기준: "순서가 틀리면 비즈니스 로직이 깨지는가?" → Yes: FIFO / No: Standard
- 대부분의 실무: **Standard + 멱등성** 조합이 표준 선택
- FIFO가 맞는 경우: 결제 처리, 주문 상태 변경 등 순서가 중요한 도메인

---

## 6) Long Polling vs Short Polling

**정의:**
Consumer가 큐에서 메시지를 가져오는 방식.

```
Short Polling (기본):
  Consumer → "메시지 있어?" → "없음" (빈 응답, 비용 발생)
  Consumer → "메시지 있어?" → "없음"
  Consumer → "메시지 있어?" → "있음!" → 반환

Long Polling (WaitTimeSeconds=20):
  Consumer → "메시지 있어? 20초까지 기다릴게"
              ... 서버 대기 → 메시지 도착! → 반환
```

**실무 포인트:**
- Long Polling 설정: `ReceiveMessageWaitTimeSeconds = 20` (최대 20초)
- 빈 응답 감소 → **비용 절감 + 지연 감소**
- 실무에서는 거의 항상 Long Polling 사용

---

# 실무 패턴

## 재시도 전략 (Retry Strategy)

**문제:**
일시적 장애(네트워크, 외부 API 타임아웃)로 메시지 처리 실패 시 즉시 재시도하면 같은 장애를 반복.

**해결:**
Visibility Timeout을 활용한 자연 재시도 + DLQ로 최종 격리.

```
1차 실패 → 30초 후 재노출 (visibility timeout)
2차 실패 → 30초 후 재노출
3차 실패 → DLQ로 이동 (maxReceiveCount=3)
```

**면접 포인트:**
"Exponential Backoff를 SQS에서 어떻게 구현하나?" → SQS 자체는 고정 visibility timeout이지만, Consumer가 `ChangeMessageVisibility` API로 타임아웃을 동적으로 늘릴 수 있음 (1차: 30초, 2차: 60초, 3차: 120초).

---

# 면접 대비 Q&A

## Q. SQS와 Kafka의 차이점은?
**A.** SQS는 완전 관리형 큐로 메시지가 소비되면 삭제됨 (일회성). Kafka는 이벤트 로그 기반으로 메시지가 보존되어 여러 Consumer가 독립적으로 읽을 수 있음 (재처리 용이). SQS는 운영 부담 없이 빠르게 도입할 때, Kafka는 이벤트 소싱이나 다수 구독자가 필요할 때 적합.

## Q. SQS 메시지가 중복 처리되는 상황과 대응 방법은?
**A.** Visibility Timeout 내에 처리를 못 끝내거나, 내부 복제 동기화 지연으로 같은 메시지가 2번 전달될 수 있음. 대응: Consumer에 멱등성 키(messageId) 기반 중복 체크 로직 구현. Redis나 DB에 처리 완료 ID를 기록.

## Q. DLQ에 메시지가 계속 쌓이면 어떻게 대응하나?
**A.** CloudWatch 알람으로 DLQ 메시지 수 모니터링 → 알람 발생 시 원인 분석 → 근본 원인 수정 후 AWS 콘솔의 DLQ redrive 기능으로 메인 큐에 재투입.

## Q. Visibility Timeout을 어떻게 결정하나?
**A.** 예상 처리 시간의 약 6배로 설정. 평균 5초 처리라면 30초. 너무 짧으면 중복 처리, 너무 길면 장애 복구 지연. 처리 시간이 가변적이면 `ChangeMessageVisibility` API로 동적 조절.

## Q. SQS Standard와 FIFO 중 어떤 걸 선택하나?
**A.** 순서가 비즈니스 로직에 영향을 주는가가 판단 기준. 알림 발송, 로그 수집 → Standard. 결제, 주문 상태 변경 → FIFO. 대부분의 경우 Standard + 멱등성이 실무 표준.

## Q. SQS를 Spring Boot에서 어떻게 사용하나?
**A.** `spring-cloud-aws-starter-sqs` 의존성 사용. Producer는 `SqsTemplate.send()`, Consumer는 `@SqsListener` 어노테이션. 정상 리턴 시 자동 Delete, 예외 시 재시도. Spring Cloud AWS가 직렬화, 에러 핸들링, Long Polling을 자동 처리.

---

# 암기용 요약

```
At-least-once — SQS Standard 기본 보장. 중복 가능 → 멱등성 필수
Visibility Timeout — 메시지 "안 보이게" 하는 시간. 처리 후 Delete 필수
DLQ — 3회 실패 → 격리 큐. 모니터링 + redrive
Idempotency — 같은 메시지 N번 처리해도 결과 동일. messageId로 중복 체크
Standard vs FIFO — 순서 불필요=Standard, 순서 필수=FIFO
Long Polling — WaitTimeSeconds=20, 빈 응답 줄여 비용 절감
Producer — 메시지를 큐에 넣는 쪽 (SqsTemplate.send)
Consumer — 메시지를 꺼내 처리하는 쪽 (@SqsListener)
Delete — Consumer가 명시적 호출해야 영구 삭제 (Spring은 자동)
maxReceiveCount — DLQ 이동 전 최대 재시도 횟수 (보통 3~5)
```

---

# 비교 정리

| 항목 | SQS | Kafka | RabbitMQ |
|------|-----|-------|----------|
| 유형 | 메시지 큐 | 이벤트 로그 | 메시지 브로커 |
| 메시지 보존 | 소비 후 삭제 | 보존 (retention) | 소비 후 삭제 |
| 순서 보장 | FIFO만 | 파티션 내 보장 | 큐 내 보장 |
| 다중 Consumer | 경쟁 소비만 | 독립 소비 가능 | 경쟁/팬아웃 |
| 운영 부담 | 없음 (관리형) | 높음 (클러스터) | 중간 |
| 적합 케이스 | 단순 비동기 처리 | 이벤트 소싱, 스트리밍 | 복잡한 라우팅 |
