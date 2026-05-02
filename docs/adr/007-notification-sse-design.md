# ADR-007: 알람(Notification) 기능 — SSE + DB 테이블 방식 채택

## 상태
<!-- 제안됨 | 검토중 | 승인됨 | 거부됨 | 폐기됨 | 대체됨 -->
승인됨 — 구현 완료 (2026-05-02)

## 날짜
2026-04-14

## 맥락

결제 요청·승인 등 업무 흐름에서 관련 담당자가 적시에 상황을 인지하지 못하는 문제가 있다.
예를 들어 견적 승인 요청이 제출되어도 승인권자가 직접 화면을 열어보기 전까지는 인지가 불가능하다.

현재 인프라:
- 백엔드: Spring Boot + Spring Modulith (도메인 이벤트 기반 모듈간 통신 이미 구축)
- 프론트엔드: Vue 3 + Pinia. `AppHeader`에 Bell 아이콘이 이미 존재하나 기능 없음
- 별도 메시지 브로커(Kafka, RabbitMQ 등) 없음

알람을 발생시켜야 하는 주요 이벤트:

| 이벤트 | 수신자 |
|---|---|
| 견적 승인 요청 제출 | 승인권자 |
| 견적 승인 / 반려 | 견적 담당자 |
| 구매 요청 → 발주 전환 | 구매 요청자 |
| 발주 취소 | 구매 요청자 |
| 입고 확정 | 구매 담당자 |

## 검토한 선택지

- **선택지 A — Polling**: 클라이언트가 주기적으로 `/notifications/unread` 를 호출
- **선택지 B — WebSocket (STOMP)**: 양방향 연결로 실시간 푸시
- **선택지 C — SSE (Server-Sent Events) + DB 알람 테이블**: 서버→클라이언트 단방향 스트림, 알람 레코드를 DB에 영속

## 결정

**선택지 C (SSE + DB 테이블)** 를 채택한다.

알람은 서버→클라이언트 단방향 전달로 충분하므로 WebSocket의 양방향 복잡도가 불필요하다.
Polling 대비 SSE는 연결 유지 비용이 낮고 지연이 없다.
DB 테이블에 알람을 영속함으로써 SSE 연결이 끊겼을 때도 미확인 알람을 재조회할 수 있다.
추가 인프라(브로커) 없이 Spring의 `SseEmitter`만으로 구현 가능하다.

## 구현 구조

### 백엔드 — `notification` 모듈 신설

```
notification/
  domain/
    Notification.java               # id, recipientUsername, type, message, referenceId, isRead, createdAt
    NotificationRepository.java
  application/
    NotificationService.java        # create(), markRead(), getUnread()
    NotificationEventHandler.java   # 기존 도메인 이벤트 수신 → Notification 생성
  api/
    NotificationController.java     # GET /notifications, POST /notifications/{id}/read
    NotificationSseController.java  # GET /notifications/stream (SSE)
```

- `NotificationEventHandler`는 기존 Spring Modulith 이벤트(`QuoteConvertedToOrderEvent`,
  `PurchaseOrderCreatedFromPREvent`, `PurchaseOrderCancelledEvent` 등)를
  `@ApplicationModuleListener`로 수신해 알람 레코드를 생성한다.
- **기존 도메인 코드 변경 없음**.

### 프론트엔드

```
stores/notification.ts           # SSE 연결 관리, unreadCount, notifications 상태
components/NotificationPanel.vue # Bell 클릭 시 드롭다운 패널
```

`AppHeader`의 Bell 버튼에 미확인 건수 배지를 추가하고, 클릭 시 `NotificationPanel`을 표시한다.

### DB 스키마 (예시)

```sql
CREATE TABLE notification (
  id               BIGSERIAL PRIMARY KEY,
  recipient_username VARCHAR(100) NOT NULL,
  type             VARCHAR(50)  NOT NULL,  -- QUOTE_APPROVAL_REQUESTED 등
  message          TEXT         NOT NULL,
  reference_id     BIGINT,                 -- 관련 엔티티 PK (견적 ID, 발주 ID 등)
  is_read          BOOLEAN      NOT NULL DEFAULT FALSE,
  created_at       TIMESTAMP    NOT NULL DEFAULT now()
);
CREATE INDEX idx_notification_recipient ON notification(recipient_username, is_read, created_at DESC);
```

## 구현 순서

1. DB 마이그레이션 + `Notification` 도메인 모델
2. `NotificationService` + `NotificationEventHandler` — 기존 이벤트 연결
3. REST + SSE 엔드포인트
4. 프론트 Pinia 스토어 (`stores/notification.ts`)
5. `NotificationPanel.vue` + `AppHeader` Bell 연결

## 결과

### 긍정적
- 추가 인프라 없이 실시간 알람 구현 가능
- 기존 Spring Modulith 이벤트 구조 그대로 활용 — 기존 코드 변경 최소
- DB 영속으로 미확인 알람 유실 없음 (SSE 재연결 후 재수신 가능)
- `AppHeader` Bell 아이콘이 이미 준비되어 있어 프론트 변경 범위 작음

### 부정적 / 트레이드오프
- SseEmitter 관리(연결 만료, 서버 재시작 시 재연결)를 클라이언트에서 처리해야 함
- 서버 인스턴스가 여러 개일 경우(스케일아웃) SseEmitter가 인스턴스 로컬이므로 로드밸런서 sticky session 또는 별도 pub/sub 계층 필요 (현재는 단일 인스턴스 운영이므로 해당 없음)

## 관련 문서
- 관련 ADR: [ADR-004](./004-inventory-transaction-design.md) — 도메인 이벤트 기반 재고 처리 (이벤트 구조 참고)
