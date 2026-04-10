# Snaps 구독 서비스 Java 백엔드 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 스냅스 월정액 구독 서비스의 Java Spring Boot 백엔드 — 구독/결제/포토북/공유/배송 API와 정기 배치 잡 전체 구현

**Architecture:** Spring Boot 3.x 멀티모듈(domain / api / batch), 스냅스 기존 회원/DB 공유, 서비스 로직은 독립 모듈로 분리 가능한 구조. 배치 간 연쇄는 DB 폴링(@Scheduled) 방식으로 처리, 동일 JVM 내 알림은 ApplicationEventPublisher + @Async.

**Tech Stack:** Java 17, Spring Boot 3.x, Spring Data JPA, Spring Batch, Spring Security (JWT), MySQL 8, Redis, OpenFeign (PG·카카오·SMS), Swagger/OpenAPI 3

---

## 📐 서브시스템 분리 제안

이 플랜은 6개 독립 서브시스템을 포함합니다. 실행 시 아래 순서로 분리 실행을 권장합니다.

| 순서 | 서브시스템 | 설명 |
|------|-----------|------|
| 1 | Domain & 공통 | Entity, Repository, 공통 응답/예외 |
| 2 | 구독·결제 API | 가입/해지/변경/결제수단 |
| 3 | 포토북 API | 생성·뷰어·커버커스텀 |
| 4 | 공유·수신자 API | 대상자 관리·발송이력 |
| 5 | 배송 API | POD 연동·배송현황 |
| 6 | Batch 잡 | 정기결제·생성트리거·발송·발주 |

---

## 📁 File Structure

```
src/
├── main/java/com/snaps/subscription/
│   ├── domain/
│   │   ├── subscription/
│   │   │   ├── Subscription.java          # 구독 엔티티
│   │   │   ├── SubscriptionPlan.java      # 요금제 엔티티
│   │   │   ├── SubscriptionRepository.java
│   │   │   └── SubscriptionStatus.java    # ACTIVE/PAUSED/CANCELLED
│   │   ├── payment/
│   │   │   ├── PaymentMethod.java         # 결제수단
│   │   │   ├── PaymentHistory.java        # 결제내역
│   │   │   └── PaymentRepository.java
│   │   ├── photobook/
│   │   │   ├── Photobook.java             # 포토북 엔티티
│   │   │   ├── PhotobookPage.java         # 페이지
│   │   │   └── PhotobookRepository.java
│   │   ├── share/
│   │   │   ├── ShareRecipient.java        # 공유 대상자
│   │   │   ├── ShareHistory.java          # 발송 이력
│   │   │   └── ShareRepository.java
│   │   └── delivery/
│   │       ├── DeliveryOrder.java         # 배송 주문
│   │       └── DeliveryRepository.java
│   ├── api/
│   │   ├── subscription/
│   │   │   ├── SubscriptionController.java
│   │   │   ├── SubscriptionService.java
│   │   │   └── dto/                       # Request/Response DTO
│   │   ├── payment/
│   │   │   ├── PaymentController.java
│   │   │   └── PaymentService.java
│   │   ├── photobook/
│   │   │   ├── PhotobookController.java
│   │   │   └── PhotobookService.java
│   │   ├── share/
│   │   │   ├── ShareController.java
│   │   │   └── ShareService.java
│   │   ├── delivery/
│   │   │   ├── DeliveryController.java
│   │   │   └── DeliveryService.java
│   │   └── admin/
│   │       └── AdminSubscriptionController.java
│   ├── batch/
│   │   ├── billing/
│   │   │   ├── MonthlyBillingJobConfig.java
│   │   │   ├── BillingItemReader.java
│   │   │   ├── BillingItemProcessor.java
│   │   │   └── BillingItemWriter.java
│   │   ├── photobook/
│   │   │   └── PhotobookGenerationJobConfig.java
│   │   ├── share/
│   │   │   └── ShareNotificationJobConfig.java
│   │   └── delivery/
│   │       └── DeliveryOrderJobConfig.java
│   ├── infra/
│   │   ├── pg/PgClient.java               # 결제대행사 OpenFeign
│   │   ├── kakao/KakaoAlimtalkClient.java  # 카카오 알림톡
│   │   ├── sms/SmsClient.java             # SMS fallback
│   │   ├── ai/PhotoAnalysisClient.java    # AI 분석 엔진
│   │   └── pod/PodClient.java            # POD 파이프라인
│   └── common/
│       ├── response/ApiResponse.java
│       ├── exception/GlobalExceptionHandler.java
│       └── security/JwtAuthFilter.java
└── resources/
    ├── application.yml
    └── db/migration/                      # Flyway 스키마
```

---

## ═══════════════════════════════════════
## SUBSYSTEM 1: Domain & 공통 기반
## ═══════════════════════════════════════

### Task 1: 공통 응답 구조 & 예외 처리

**Files:**
- Create: `src/main/java/com/snaps/subscription/common/response/ApiResponse.java`
- Create: `src/main/java/com/snaps/subscription/common/exception/GlobalExceptionHandler.java`
- Create: `src/main/java/com/snaps/subscription/common/exception/SubscriptionException.java`
- Test: `src/test/java/com/snaps/subscription/common/ApiResponseTest.java`

- [ ] **Step 1: 테스트 작성**

```java
// ApiResponseTest.java
@Test
void success_wraps_data() {
    ApiResponse<String> res = ApiResponse.success("hello");
    assertThat(res.getCode()).isEqualTo("SUCCESS");
    assertThat(res.getData()).isEqualTo("hello");
}

@Test
void error_wraps_message() {
    ApiResponse<Void> res = ApiResponse.error("E001", "구독 없음");
    assertThat(res.getCode()).isEqualTo("E001");
    assertThat(res.getMessage()).isEqualTo("구독 없음");
}
```

- [ ] **Step 2: 테스트 실패 확인**

```bash
./mvnw test -pl api -Dtest=ApiResponseTest
# Expected: FAIL - ApiResponse not found
```

- [ ] **Step 3: ApiResponse 구현**

```java
@Getter
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class ApiResponse<T> {
    private String code;
    private String message;
    private T data;

    public static <T> ApiResponse<T> success(T data) {
        ApiResponse<T> r = new ApiResponse<>();
        r.code = "SUCCESS";
        r.data = data;
        return r;
    }

    public static <T> ApiResponse<T> error(String code, String message) {
        ApiResponse<T> r = new ApiResponse<>();
        r.code = code;
        r.message = message;
        return r;
    }
}
```

- [ ] **Step 4: SubscriptionException 구현**

```java
@Getter
public class SubscriptionException extends RuntimeException {
    private final String errorCode;

    public SubscriptionException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
}
```

- [ ] **Step 5: GlobalExceptionHandler 구현**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(SubscriptionException.class)
    public ResponseEntity<ApiResponse<Void>> handleSubscriptionException(SubscriptionException e) {
        return ResponseEntity
            .badRequest()
            .body(ApiResponse.error(e.getErrorCode(), e.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleException(Exception e) {
        return ResponseEntity
            .internalServerError()
            .body(ApiResponse.error("INTERNAL_ERROR", "서버 오류가 발생했습니다."));
    }
}
```

- [ ] **Step 6: 테스트 통과 확인 후 커밋**

```bash
./mvnw test -pl api -Dtest=ApiResponseTest
git add src/main/java/com/snaps/subscription/common/ src/test/
git commit -m "feat: 공통 API 응답 구조 및 예외 처리 추가"
```

---

### Task 2: DB 스키마 (Flyway)

**Files:**
- Create: `src/main/resources/db/migration/V1__subscription_schema.sql`
- Create: `src/main/resources/db/migration/V2__share_delivery_schema.sql`

- [ ] **Step 1: V1 — 구독/결제/포토북 테이블**

```sql
-- V1__subscription_schema.sql

CREATE TABLE subscription_plan (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    plan_code   VARCHAR(20) NOT NULL UNIQUE COMMENT 'DIGITAL|LIGHT|SEASON|FREE_TRIAL',
    name        VARCHAR(50) NOT NULL,
    price       INT         NOT NULL COMMENT '월 가격(원)',
    delivery_frequency VARCHAR(20) COMMENT 'MONTHLY|QUARTERLY|NONE',
    created_at  DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO subscription_plan (plan_code, name, price, delivery_frequency)
VALUES
    ('DIGITAL',    '디지털',    2900, 'NONE'),
    ('SEASON',     '시즌',      3900, 'QUARTERLY'),
    ('LIGHT',      '라이트',    5900, 'MONTHLY'),
    ('FREE_TRIAL', '무료체험',  0,    'NONE');

CREATE TABLE subscription (
    id                  BIGINT AUTO_INCREMENT PRIMARY KEY,
    member_id           BIGINT      NOT NULL COMMENT '스냅스 회원 ID',
    plan_code           VARCHAR(20) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE' COMMENT 'ACTIVE|PAUSED|CANCELLED|PAYMENT_FAILED',
    started_at          DATETIME    NOT NULL,
    next_billing_date   DATE        NOT NULL COMMENT '다음 결제일',
    next_generation_date DATE       NOT NULL COMMENT '다음 포토북 생성일',
    paused_at           DATETIME,
    pause_resume_at     DATE        COMMENT '일시정지 자동 해제일',
    cancelled_at        DATETIME,
    cancel_reason       VARCHAR(255),
    onboarding_cover_type VARCHAR(20) COMMENT 'CLASSIC|MODERN|MINIMAL',
    created_at          DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_member_id (member_id),
    INDEX idx_status_next_billing (status, next_billing_date)
);

CREATE TABLE payment_method (
    id                  BIGINT AUTO_INCREMENT PRIMARY KEY,
    member_id           BIGINT      NOT NULL,
    pg_type             VARCHAR(20) NOT NULL COMMENT 'TOSS|KCP|INICIS',
    pg_billing_key      VARCHAR(200) NOT NULL COMMENT 'PG사 빌링키',
    masked_card_no      VARCHAR(20),
    card_company        VARCHAR(30),
    is_default          TINYINT(1)  NOT NULL DEFAULT 1,
    created_at          DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_member_id (member_id)
);

CREATE TABLE payment_history (
    id                  BIGINT AUTO_INCREMENT PRIMARY KEY,
    subscription_id     BIGINT      NOT NULL,
    member_id           BIGINT      NOT NULL,
    amount              INT         NOT NULL,
    status              VARCHAR(20) NOT NULL COMMENT 'SUCCESS|FAILED|REFUNDED',
    pg_transaction_id   VARCHAR(100),
    billing_date        DATE        NOT NULL,
    fail_reason         VARCHAR(255),
    retry_count         INT         NOT NULL DEFAULT 0,
    created_at          DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_subscription_id (subscription_id),
    INDEX idx_billing_date (billing_date)
);

CREATE TABLE photobook (
    id                  BIGINT AUTO_INCREMENT PRIMARY KEY,
    subscription_id     BIGINT      NOT NULL,
    member_id           BIGINT      NOT NULL,
    title               VARCHAR(100),
    cover_color         VARCHAR(30) COMMENT '커버 색상코드',
    cover_text          VARCHAR(20) COMMENT '타이틀 최대 20자',
    cover_date          VARCHAR(20) COMMENT '자동 표기 날짜',
    status              VARCHAR(20) NOT NULL DEFAULT 'GENERATING' COMMENT 'GENERATING|READY|PUBLISHED|EXPIRED',
    viewer_url          VARCHAR(500),
    share_token         VARCHAR(100) UNIQUE COMMENT '비회원 뷰어 토큰',
    generated_at        DATETIME,
    published_at        DATETIME,
    target_year_month   VARCHAR(7)  NOT NULL COMMENT 'YYYY-MM',
    created_at          DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_subscription_id (subscription_id),
    INDEX idx_member_id_month (member_id, target_year_month)
);

CREATE TABLE photobook_page (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    photobook_id BIGINT     NOT NULL,
    page_no     INT         NOT NULL,
    image_url   VARCHAR(500),
    caption     VARCHAR(200),
    layout_type VARCHAR(30),
    INDEX idx_photobook_id (photobook_id)
);
```

- [ ] **Step 2: V2 — 공유/배송 테이블**

```sql
-- V2__share_delivery_schema.sql

CREATE TABLE share_recipient (
    id                  BIGINT AUTO_INCREMENT PRIMARY KEY,
    member_id           BIGINT      NOT NULL,
    subscription_id     BIGINT      NOT NULL,
    nickname            VARCHAR(30) NOT NULL COMMENT '별명',
    contact_type        VARCHAR(10) NOT NULL COMMENT 'KAKAO|PHONE',
    contact_value       VARCHAR(50) NOT NULL COMMENT '카카오 UUID 또는 전화번호',
    notification_enabled TINYINT(1) NOT NULL DEFAULT 1,
    created_at          DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_member_id (member_id),
    INDEX idx_subscription_id (subscription_id)
);

CREATE TABLE share_history (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    photobook_id    BIGINT      NOT NULL,
    recipient_id    BIGINT      NOT NULL,
    channel         VARCHAR(10) NOT NULL COMMENT 'KAKAO|SMS',
    status          VARCHAR(20) NOT NULL COMMENT 'SENT|FAILED|READ',
    sent_at         DATETIME,
    read_at         DATETIME,
    created_at      DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_photobook_id (photobook_id),
    INDEX idx_recipient_id (recipient_id)
);

CREATE TABLE delivery_order (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    subscription_id BIGINT      NOT NULL,
    member_id       BIGINT      NOT NULL,
    photobook_id    BIGINT,
    order_type      VARCHAR(20) NOT NULL COMMENT 'AUTO|RECIPIENT_ORDER|EXTRA',
    status          VARCHAR(20) NOT NULL DEFAULT 'PENDING'
                    COMMENT 'PENDING|MANUFACTURING|SHIPPING|DELIVERED|CANCELLED',
    recipient_name  VARCHAR(50),
    phone           VARCHAR(20),
    address         VARCHAR(300),
    address_detail  VARCHAR(100),
    zipcode         VARCHAR(10),
    pod_order_id    VARCHAR(100) COMMENT 'POD 파이프라인 주문번호',
    tracking_no     VARCHAR(100),
    courier         VARCHAR(30),
    ordered_at      DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    delivered_at    DATETIME,
    INDEX idx_subscription_id (subscription_id),
    INDEX idx_status (status)
);
```

- [ ] **Step 3: Flyway 실행 확인**

```bash
./mvnw flyway:migrate -Dflyway.url=jdbc:mysql://localhost:3306/snaps_subscription
# Expected: Successfully applied 2 migrations
```

- [ ] **Step 4: 커밋**

```bash
git add src/main/resources/db/migration/
git commit -m "feat: 구독 서비스 DB 스키마 초기 생성 (Flyway V1, V2)"
```

---

### Task 3: JPA Entity 및 Repository

**Files:**
- Create: `src/main/java/com/snaps/subscription/domain/subscription/Subscription.java`
- Create: `src/main/java/com/snaps/subscription/domain/subscription/SubscriptionRepository.java`
- Create: `src/main/java/com/snaps/subscription/domain/payment/PaymentMethod.java`
- Create: `src/main/java/com/snaps/subscription/domain/payment/PaymentHistory.java`
- Create: `src/main/java/com/snaps/subscription/domain/payment/PaymentRepository.java`
- Create: `src/main/java/com/snaps/subscription/domain/photobook/Photobook.java`
- Create: `src/main/java/com/snaps/subscription/domain/photobook/PhotobookRepository.java`
- Create: `src/main/java/com/snaps/subscription/domain/share/ShareRecipient.java`
- Create: `src/main/java/com/snaps/subscription/domain/share/ShareHistory.java`
- Create: `src/main/java/com/snaps/subscription/domain/delivery/DeliveryOrder.java`
- Create: `src/main/java/com/snaps/subscription/domain/delivery/DeliveryRepository.java`
- Test: `src/test/java/com/snaps/subscription/domain/SubscriptionRepositoryTest.java`

- [ ] **Step 1: Repository 슬라이스 테스트 작성**

```java
@DataJpaTest
class SubscriptionRepositoryTest {

    @Autowired
    SubscriptionRepository subscriptionRepository;

    @Test
    void findActiveByNextBillingDate_returns_due_subscriptions() {
        Subscription sub = Subscription.builder()
            .memberId(1001L)
            .planCode("DIGITAL")
            .status(SubscriptionStatus.ACTIVE)
            .startedAt(LocalDateTime.now().minusDays(30))
            .nextBillingDate(LocalDate.now())
            .nextGenerationDate(LocalDate.now())
            .build();
        subscriptionRepository.save(sub);

        List<Subscription> result = subscriptionRepository
            .findByStatusAndNextBillingDateLessThanEqual(SubscriptionStatus.ACTIVE, LocalDate.now());

        assertThat(result).hasSize(1);
        assertThat(result.get(0).getMemberId()).isEqualTo(1001L);
    }
}
```

- [ ] **Step 2: 테스트 실패 확인**

```bash
./mvnw test -Dtest=SubscriptionRepositoryTest
# Expected: FAIL - Subscription not found
```

- [ ] **Step 3: Subscription Entity 구현**

```java
@Entity
@Table(name = "subscription")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Builder
@AllArgsConstructor
public class Subscription {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private Long memberId;

    @Column(nullable = false, length = 20)
    private String planCode;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private SubscriptionStatus status;

    @Column(nullable = false)
    private LocalDateTime startedAt;

    @Column(nullable = false)
    private LocalDate nextBillingDate;

    @Column(nullable = false)
    private LocalDate nextGenerationDate;

    private LocalDateTime pausedAt;
    private LocalDate pauseResumeAt;
    private LocalDateTime cancelledAt;
    private String cancelReason;
    private String onboardingCoverType;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    private LocalDateTime updatedAt;

    @PrePersist
    void prePersist() { this.createdAt = this.updatedAt = LocalDateTime.now(); }

    @PreUpdate
    void preUpdate() { this.updatedAt = LocalDateTime.now(); }

    // 도메인 메서드
    public void cancel(String reason) {
        this.status = SubscriptionStatus.CANCELLED;
        this.cancelledAt = LocalDateTime.now();
        this.cancelReason = reason;
    }

    public void pause(LocalDate resumeAt) {
        this.status = SubscriptionStatus.PAUSED;
        this.pausedAt = LocalDateTime.now();
        this.pauseResumeAt = resumeAt;
    }

    public void resume() {
        this.status = SubscriptionStatus.ACTIVE;
        this.pausedAt = null;
        this.pauseResumeAt = null;
    }

    public void changePlan(String newPlanCode) {
        this.planCode = newPlanCode;
    }

    public void advanceBillingDate(int months) {
        this.nextBillingDate = this.nextBillingDate.plusMonths(months);
        this.nextGenerationDate = this.nextGenerationDate.plusMonths(months);
    }

    public void markPaymentFailed() {
        this.status = SubscriptionStatus.PAYMENT_FAILED;
    }
}
```

- [ ] **Step 4: SubscriptionRepository 구현**

```java
public interface SubscriptionRepository extends JpaRepository<Subscription, Long> {

    Optional<Subscription> findByMemberIdAndStatusIn(Long memberId, List<SubscriptionStatus> statuses);

    // 배치용: 결제 대상 조회
    List<Subscription> findByStatusAndNextBillingDateLessThanEqual(
        SubscriptionStatus status, LocalDate date);

    // 배치용: 일시정지 자동 해제 대상
    List<Subscription> findByStatusAndPauseResumeAtLessThanEqual(
        SubscriptionStatus status, LocalDate date);
}
```

- [ ] **Step 5: PaymentMethod, PaymentHistory Entity 구현**

```java
@Entity
@Table(name = "payment_method")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Builder
@AllArgsConstructor
public class PaymentMethod {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long memberId;
    private String pgType;
    private String pgBillingKey;
    private String maskedCardNo;
    private String cardCompany;
    private boolean isDefault;
    private LocalDateTime createdAt;

    @PrePersist
    void prePersist() { this.createdAt = LocalDateTime.now(); }
}

@Entity
@Table(name = "payment_history")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Builder
@AllArgsConstructor
public class PaymentHistory {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long subscriptionId;
    private Long memberId;
    private int amount;
    @Enumerated(EnumType.STRING)
    private PaymentStatus status;    // SUCCESS|FAILED|REFUNDED
    private String pgTransactionId;
    private LocalDate billingDate;
    private String failReason;
    private int retryCount;
    private LocalDateTime createdAt;

    @PrePersist
    void prePersist() { this.createdAt = LocalDateTime.now(); }
}
```

- [ ] **Step 6: Photobook Entity 구현**

```java
@Entity
@Table(name = "photobook")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Builder
@AllArgsConstructor
public class Photobook {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long subscriptionId;
    private Long memberId;
    private String title;
    private String coverColor;
    @Column(length = 20)
    private String coverText;
    private String coverDate;
    @Enumerated(EnumType.STRING)
    private PhotobookStatus status;  // GENERATING|READY|PUBLISHED|EXPIRED
    private String viewerUrl;
    @Column(unique = true)
    private String shareToken;
    private LocalDateTime generatedAt;
    private LocalDateTime publishedAt;
    private String targetYearMonth;  // YYYY-MM
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    @PrePersist
    void prePersist() { this.createdAt = this.updatedAt = LocalDateTime.now(); }
    @PreUpdate
    void preUpdate() { this.updatedAt = LocalDateTime.now(); }

    public void markReady(String viewerUrl, String shareToken) {
        this.status = PhotobookStatus.READY;
        this.viewerUrl = viewerUrl;
        this.shareToken = shareToken;
        this.generatedAt = LocalDateTime.now();
    }

    public void publish() {
        this.status = PhotobookStatus.PUBLISHED;
        this.publishedAt = LocalDateTime.now();
    }

    public void updateCover(String color, String text) {
        if (text != null && text.length() > 20)
            throw new SubscriptionException("INVALID_COVER_TEXT", "커버 텍스트는 최대 20자입니다.");
        this.coverColor = color;
        this.coverText = text;
    }
}
```

- [ ] **Step 7: ShareRecipient, ShareHistory Entity 구현**

```java
@Entity
@Table(name = "share_recipient")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Builder
@AllArgsConstructor
public class ShareRecipient {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long memberId;
    private Long subscriptionId;
    private String nickname;
    private String contactType;   // KAKAO|PHONE
    private String contactValue;
    private boolean notificationEnabled;
    private LocalDateTime createdAt;

    @PrePersist
    void prePersist() { this.createdAt = LocalDateTime.now(); }

    public void update(String nickname, boolean notificationEnabled) {
        this.nickname = nickname;
        this.notificationEnabled = notificationEnabled;
    }
}

@Entity
@Table(name = "share_history")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Builder
@AllArgsConstructor
public class ShareHistory {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long photobookId;
    private Long recipientId;
    private String channel;   // KAKAO|SMS
    @Enumerated(EnumType.STRING)
    private ShareStatus status;  // SENT|FAILED|READ
    private LocalDateTime sentAt;
    private LocalDateTime readAt;
    private LocalDateTime createdAt;

    @PrePersist
    void prePersist() { this.createdAt = LocalDateTime.now(); }
}
```

- [ ] **Step 8: DeliveryOrder Entity 구현**

```java
@Entity
@Table(name = "delivery_order")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Builder
@AllArgsConstructor
public class DeliveryOrder {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long subscriptionId;
    private Long memberId;
    private Long photobookId;
    private String orderType;      // AUTO|RECIPIENT_ORDER|EXTRA
    @Enumerated(EnumType.STRING)
    private DeliveryStatus status; // PENDING|MANUFACTURING|SHIPPING|DELIVERED|CANCELLED
    private String recipientName;
    private String phone;
    private String address;
    private String addressDetail;
    private String zipcode;
    private String podOrderId;
    private String trackingNo;
    private String courier;
    private LocalDateTime orderedAt;
    private LocalDateTime deliveredAt;

    @PrePersist
    void prePersist() { this.orderedAt = LocalDateTime.now(); }

    public void updateStatus(DeliveryStatus status) {
        this.status = status;
        if (status == DeliveryStatus.DELIVERED) this.deliveredAt = LocalDateTime.now();
    }

    public void setPodOrderId(String podOrderId) { this.podOrderId = podOrderId; }
    public void setTracking(String courier, String trackingNo) {
        this.courier = courier;
        this.trackingNo = trackingNo;
    }
}
```

- [ ] **Step 9: 테스트 통과 확인 후 커밋**

```bash
./mvnw test -Dtest=SubscriptionRepositoryTest
git add src/main/java/com/snaps/subscription/domain/
git commit -m "feat: JPA Entity 및 Repository 전체 구현 (구독/결제/포토북/공유/배송)"
```

---

## ═══════════════════════════════════════
## SUBSYSTEM 2: 구독·결제 API
## ═══════════════════════════════════════

### Task 4: 구독 가입 API

**Endpoint:** `POST /api/v1/subscriptions`

**Files:**
- Create: `src/main/java/com/snaps/subscription/api/subscription/SubscriptionController.java`
- Create: `src/main/java/com/snaps/subscription/api/subscription/SubscriptionService.java`
- Create: `src/main/java/com/snaps/subscription/api/subscription/dto/SubscribeRequest.java`
- Create: `src/main/java/com/snaps/subscription/api/subscription/dto/SubscriptionResponse.java`
- Test: `src/test/java/com/snaps/subscription/api/subscription/SubscriptionControllerTest.java`

**Request Body:**
```json
{
  "planCode": "LIGHT",
  "pgType": "TOSS",
  "pgBillingKey": "billing_key_from_pg",
  "maskedCardNo": "1234-****-****-5678",
  "cardCompany": "신한",
  "onboardingCoverType": "CLASSIC"
}
```

**Response:**
```json
{
  "code": "SUCCESS",
  "data": {
    "subscriptionId": 1,
    "planCode": "LIGHT",
    "status": "ACTIVE",
    "nextBillingDate": "2026-05-10",
    "nextGenerationDate": "2026-05-10"
  }
}
```

**비즈니스 규칙:**
- 이미 ACTIVE/PAUSED 구독이 있으면 `E_ALREADY_SUBSCRIBED` 오류
- 결제수단 등록 → 구독 생성 → 첫 결제 즉시 시도 (FREE_TRIAL 제외)

- [ ] **Step 1: Controller 슬라이스 테스트 작성**

```java
@WebMvcTest(SubscriptionController.class)
class SubscriptionControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean SubscriptionService subscriptionService;
    @Autowired ObjectMapper objectMapper;

    @Test
    @WithMockUser(username = "1001")
    void subscribe_returns_201() throws Exception {
        SubscribeRequest req = new SubscribeRequest(
            "LIGHT", "TOSS", "bk_test_key", "1234-****-5678", "신한", "CLASSIC");

        SubscriptionResponse res = new SubscriptionResponse(
            1L, "LIGHT", "ACTIVE", LocalDate.now().plusMonths(1), LocalDate.now().plusMonths(1));

        given(subscriptionService.subscribe(eq(1001L), any())).willReturn(res);

        mockMvc.perform(post("/api/v1/subscriptions")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(req)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.code").value("SUCCESS"))
            .andExpect(jsonPath("$.data.subscriptionId").value(1));
    }

    @Test
    @WithMockUser(username = "1001")
    void subscribe_duplicate_returns_400() throws Exception {
        given(subscriptionService.subscribe(eq(1001L), any()))
            .willThrow(new SubscriptionException("E_ALREADY_SUBSCRIBED", "이미 활성 구독이 있습니다."));

        mockMvc.perform(post("/api/v1/subscriptions")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(
                    new SubscribeRequest("LIGHT", "TOSS", "key", "no", "신한", "CLASSIC"))))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.code").value("E_ALREADY_SUBSCRIBED"));
    }
}
```

- [ ] **Step 2: 테스트 실패 확인**

```bash
./mvnw test -Dtest=SubscriptionControllerTest
# Expected: FAIL
```

- [ ] **Step 3: DTO 구현**

```java
// SubscribeRequest.java
public record SubscribeRequest(
    @NotBlank String planCode,
    @NotBlank String pgType,
    @NotBlank String pgBillingKey,
    String maskedCardNo,
    String cardCompany,
    String onboardingCoverType
) {}

// SubscriptionResponse.java
public record SubscriptionResponse(
    Long subscriptionId,
    String planCode,
    String status,
    LocalDate nextBillingDate,
    LocalDate nextGenerationDate
) {}
```

- [ ] **Step 4: SubscriptionService 구현**

```java
@Service
@RequiredArgsConstructor
@Transactional
public class SubscriptionService {

    private final SubscriptionRepository subscriptionRepository;
    private final PaymentMethodRepository paymentMethodRepository;
    private final PgClient pgClient;

    public SubscriptionResponse subscribe(Long memberId, SubscribeRequest req) {
        // 중복 구독 체크
        subscriptionRepository.findByMemberIdAndStatusIn(
            memberId, List.of(SubscriptionStatus.ACTIVE, SubscriptionStatus.PAUSED))
            .ifPresent(s -> { throw new SubscriptionException("E_ALREADY_SUBSCRIBED", "이미 활성 구독이 있습니다."); });

        // 결제수단 저장
        PaymentMethod method = PaymentMethod.builder()
            .memberId(memberId)
            .pgType(req.pgType())
            .pgBillingKey(req.pgBillingKey())
            .maskedCardNo(req.maskedCardNo())
            .cardCompany(req.cardCompany())
            .isDefault(true)
            .build();
        paymentMethodRepository.save(method);

        // 구독 생성
        LocalDate today = LocalDate.now();
        Subscription sub = Subscription.builder()
            .memberId(memberId)
            .planCode(req.planCode())
            .status(SubscriptionStatus.ACTIVE)
            .startedAt(LocalDateTime.now())
            .nextBillingDate(today.plusMonths(1))
            .nextGenerationDate(today.plusMonths(1))
            .onboardingCoverType(req.onboardingCoverType())
            .build();
        subscriptionRepository.save(sub);

        // FREE_TRIAL 아니면 즉시 첫 결제
        if (!"FREE_TRIAL".equals(req.planCode())) {
            pgClient.chargeImmediately(req.pgBillingKey(), getPlanPrice(req.planCode()));
        }

        return new SubscriptionResponse(
            sub.getId(), sub.getPlanCode(), sub.getStatus().name(),
            sub.getNextBillingDate(), sub.getNextGenerationDate());
    }

    private int getPlanPrice(String planCode) {
        return switch (planCode) {
            case "DIGITAL" -> 2900;
            case "SEASON"  -> 3900;
            case "LIGHT"   -> 5900;
            default -> 0;
        };
    }
}
```

- [ ] **Step 5: Controller 구현**

```java
@RestController
@RequestMapping("/api/v1/subscriptions")
@RequiredArgsConstructor
public class SubscriptionController {

    private final SubscriptionService subscriptionService;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ApiResponse<SubscriptionResponse> subscribe(
            @AuthenticationPrincipal Long memberId,
            @RequestBody @Valid SubscribeRequest req) {
        return ApiResponse.success(subscriptionService.subscribe(memberId, req));
    }
}
```

- [ ] **Step 6: 테스트 통과 확인 후 커밋**

```bash
./mvnw test -Dtest=SubscriptionControllerTest
git add src/main/java/com/snaps/subscription/api/subscription/
git commit -m "feat: 구독 가입 API 구현 (POST /api/v1/subscriptions)"
```

---

### Task 5: 구독 조회 & 대시보드 API

**Endpoints:**
- `GET /api/v1/subscriptions/me` — 내 구독 현황
- `GET /api/v1/subscriptions/me/dashboard` — 대시보드 종합

**Files:**
- Modify: `src/main/java/com/snaps/subscription/api/subscription/SubscriptionController.java`
- Modify: `src/main/java/com/snaps/subscription/api/subscription/SubscriptionService.java`
- Create: `src/main/java/com/snaps/subscription/api/subscription/dto/DashboardResponse.java`

**GET /api/v1/subscriptions/me Response:**
```json
{
  "code": "SUCCESS",
  "data": {
    "subscriptionId": 1,
    "planCode": "LIGHT",
    "planName": "라이트",
    "status": "ACTIVE",
    "nextBillingDate": "2026-05-10",
    "nextGenerationDate": "2026-05-10",
    "daysUntilNextGeneration": 30
  }
}
```

**GET /api/v1/subscriptions/me/dashboard Response:**
```json
{
  "code": "SUCCESS",
  "data": {
    "subscription": { ... },
    "latestPhotobook": {
      "photobookId": 10,
      "targetYearMonth": "2026-04",
      "thumbnailUrl": "https://...",
      "status": "PUBLISHED"
    },
    "recentDelivery": {
      "deliveryId": 5,
      "status": "SHIPPING",
      "trackingNo": "1234567890"
    },
    "unreadShareCount": 2
  }
}
```

- [ ] **Step 1: 테스트 작성**

```java
@Test
@WithMockUser(username = "1001")
void getMySubscription_returns_active_sub() throws Exception {
    SubscriptionDetailResponse res = new SubscriptionDetailResponse(
        1L, "LIGHT", "라이트", "ACTIVE",
        LocalDate.now().plusMonths(1), LocalDate.now().plusMonths(1), 30);
    given(subscriptionService.getMySubscription(1001L)).willReturn(res);

    mockMvc.perform(get("/api/v1/subscriptions/me"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.data.planCode").value("LIGHT"))
        .andExpect(jsonPath("$.data.daysUntilNextGeneration").value(30));
}
```

- [ ] **Step 2: Service 메서드 추가**

```java
@Transactional(readOnly = true)
public SubscriptionDetailResponse getMySubscription(Long memberId) {
    Subscription sub = subscriptionRepository.findByMemberIdAndStatusIn(
        memberId, List.of(SubscriptionStatus.ACTIVE, SubscriptionStatus.PAUSED))
        .orElseThrow(() -> new SubscriptionException("E_NO_SUBSCRIPTION", "활성 구독이 없습니다."));

    long daysUntil = ChronoUnit.DAYS.between(LocalDate.now(), sub.getNextGenerationDate());
    String planName = getPlanName(sub.getPlanCode());

    return new SubscriptionDetailResponse(
        sub.getId(), sub.getPlanCode(), planName, sub.getStatus().name(),
        sub.getNextBillingDate(), sub.getNextGenerationDate(), (int) daysUntil);
}
```

- [ ] **Step 3: Controller 엔드포인트 추가**

```java
@GetMapping("/me")
public ApiResponse<SubscriptionDetailResponse> getMySubscription(
        @AuthenticationPrincipal Long memberId) {
    return ApiResponse.success(subscriptionService.getMySubscription(memberId));
}

@GetMapping("/me/dashboard")
public ApiResponse<DashboardResponse> getDashboard(
        @AuthenticationPrincipal Long memberId) {
    return ApiResponse.success(subscriptionService.getDashboard(memberId));
}
```

- [ ] **Step 4: 테스트 통과 후 커밋**

```bash
./mvnw test -Dtest=SubscriptionControllerTest
git commit -m "feat: 구독 현황 및 대시보드 조회 API 추가"
```

---

### Task 6: 구독 변경·해지·일시정지 API

**Endpoints:**
- `PATCH /api/v1/subscriptions/me/plan` — 요금제 변경
- `POST /api/v1/subscriptions/me/cancel` — 해지
- `POST /api/v1/subscriptions/me/pause` — 일시정지
- `POST /api/v1/subscriptions/me/resume` — 재개

**Request/Response 상세:**

```json
// PATCH /api/v1/subscriptions/me/plan
{ "planCode": "SEASON" }

// POST /api/v1/subscriptions/me/cancel
{ "cancelReason": "가격이 부담돼요" }

// POST /api/v1/subscriptions/me/pause
{ "pauseResumeAt": "2026-07-01" }
```

**비즈니스 규칙:**
- 다운그레이드는 다음 결제일부터 적용
- 업그레이드는 즉시 적용 + 차액 즉시 결제
- 해지 시 리텐션 오퍼 로직 트리거 (오퍼 발송 이벤트 발행)

- [ ] **Step 1: 테스트 작성**

```java
@Test
@WithMockUser(username = "1001")
void cancel_subscription_returns_200() throws Exception {
    mockMvc.perform(post("/api/v1/subscriptions/me/cancel")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"cancelReason\": \"가격이 부담돼요\"}"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.code").value("SUCCESS"));
}
```

- [ ] **Step 2: Service 메서드 구현**

```java
public void changePlan(Long memberId, String newPlanCode) {
    Subscription sub = getActiveSubscription(memberId);
    boolean isUpgrade = getPlanPrice(newPlanCode) > getPlanPrice(sub.getPlanCode());

    if (isUpgrade) {
        PaymentMethod method = paymentMethodRepository
            .findByMemberIdAndIsDefaultTrue(memberId)
            .orElseThrow(() -> new SubscriptionException("E_NO_PAYMENT", "결제수단이 없습니다."));
        int diff = getPlanPrice(newPlanCode) - getPlanPrice(sub.getPlanCode());
        pgClient.chargeImmediately(method.getPgBillingKey(), diff);
    }

    sub.changePlan(newPlanCode);
}

public void cancel(Long memberId, String reason) {
    Subscription sub = getActiveSubscription(memberId);
    sub.cancel(reason);
    // 리텐션 오퍼 이벤트 발행 (Kafka)
    eventPublisher.publishEvent(new SubscriptionCancelledEvent(memberId, reason));
}

public void pause(Long memberId, LocalDate resumeAt) {
    Subscription sub = getActiveSubscription(memberId);
    sub.pause(resumeAt);
}

public void resume(Long memberId) {
    Subscription sub = subscriptionRepository
        .findByMemberIdAndStatusIn(memberId, List.of(SubscriptionStatus.PAUSED))
        .orElseThrow(() -> new SubscriptionException("E_NOT_PAUSED", "일시정지 상태가 아닙니다."));
    sub.resume();
}

private Subscription getActiveSubscription(Long memberId) {
    return subscriptionRepository.findByMemberIdAndStatusIn(
        memberId, List.of(SubscriptionStatus.ACTIVE))
        .orElseThrow(() -> new SubscriptionException("E_NO_SUBSCRIPTION", "활성 구독이 없습니다."));
}
```

- [ ] **Step 3: Controller 엔드포인트 추가**

```java
@PatchMapping("/me/plan")
public ApiResponse<Void> changePlan(
        @AuthenticationPrincipal Long memberId,
        @RequestBody @Valid ChangePlanRequest req) {
    subscriptionService.changePlan(memberId, req.planCode());
    return ApiResponse.success(null);
}

@PostMapping("/me/cancel")
public ApiResponse<Void> cancel(
        @AuthenticationPrincipal Long memberId,
        @RequestBody CancelRequest req) {
    subscriptionService.cancel(memberId, req.cancelReason());
    return ApiResponse.success(null);
}

@PostMapping("/me/pause")
public ApiResponse<Void> pause(
        @AuthenticationPrincipal Long memberId,
        @RequestBody PauseRequest req) {
    subscriptionService.pause(memberId, req.pauseResumeAt());
    return ApiResponse.success(null);
}

@PostMapping("/me/resume")
public ApiResponse<Void> resume(@AuthenticationPrincipal Long memberId) {
    subscriptionService.resume(memberId);
    return ApiResponse.success(null);
}
```

- [ ] **Step 4: 커밋**

```bash
./mvnw test -Dtest=SubscriptionControllerTest
git commit -m "feat: 구독 요금제 변경·해지·일시정지·재개 API 구현"
```

---

### Task 7: 결제수단 & 결제내역 API

**Endpoints:**
- `GET /api/v1/payments/methods` — 결제수단 조회
- `PUT /api/v1/payments/methods` — 결제수단 변경
- `GET /api/v1/payments/history` — 결제내역 (페이징)

**Files:**
- Create: `src/main/java/com/snaps/subscription/api/payment/PaymentController.java`
- Create: `src/main/java/com/snaps/subscription/api/payment/PaymentService.java`

**GET /api/v1/payments/history Response:**
```json
{
  "code": "SUCCESS",
  "data": {
    "content": [
      {
        "paymentId": 1,
        "amount": 5900,
        "status": "SUCCESS",
        "billingDate": "2026-04-10",
        "pgTransactionId": "txn_abc123"
      }
    ],
    "page": 0,
    "size": 20,
    "totalElements": 5
  }
}
```

- [ ] **Step 1: 테스트 작성**

```java
@Test
@WithMockUser(username = "1001")
void getPaymentHistory_returns_paged_results() throws Exception {
    mockMvc.perform(get("/api/v1/payments/history?page=0&size=20"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.data.content").isArray());
}
```

- [ ] **Step 2: PaymentService 구현**

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class PaymentService {

    private final PaymentMethodRepository paymentMethodRepository;
    private final PaymentHistoryRepository paymentHistoryRepository;
    private final PgClient pgClient;

    public PaymentMethodResponse getMethod(Long memberId) {
        PaymentMethod m = paymentMethodRepository.findByMemberIdAndIsDefaultTrue(memberId)
            .orElseThrow(() -> new SubscriptionException("E_NO_PAYMENT", "결제수단이 없습니다."));
        return new PaymentMethodResponse(m.getMaskedCardNo(), m.getCardCompany(), m.getPgType());
    }

    @Transactional
    public void updateMethod(Long memberId, UpdatePaymentMethodRequest req) {
        // 기존 billing key 삭제 (PG사 요청)
        paymentMethodRepository.findByMemberIdAndIsDefaultTrue(memberId)
            .ifPresent(m -> pgClient.deleteBillingKey(m.getPgBillingKey()));

        // 새 결제수단 저장
        PaymentMethod newMethod = PaymentMethod.builder()
            .memberId(memberId)
            .pgType(req.pgType())
            .pgBillingKey(req.pgBillingKey())
            .maskedCardNo(req.maskedCardNo())
            .cardCompany(req.cardCompany())
            .isDefault(true)
            .build();
        paymentMethodRepository.save(newMethod);
    }

    public Page<PaymentHistoryResponse> getHistory(Long memberId, Pageable pageable) {
        return paymentHistoryRepository
            .findByMemberIdOrderByBillingDateDesc(memberId, pageable)
            .map(h -> new PaymentHistoryResponse(
                h.getId(), h.getAmount(), h.getStatus().name(),
                h.getBillingDate(), h.getPgTransactionId()));
    }
}
```

- [ ] **Step 3: Controller 구현 및 커밋**

```java
@RestController
@RequestMapping("/api/v1/payments")
@RequiredArgsConstructor
public class PaymentController {

    private final PaymentService paymentService;

    @GetMapping("/methods")
    public ApiResponse<PaymentMethodResponse> getMethod(@AuthenticationPrincipal Long memberId) {
        return ApiResponse.success(paymentService.getMethod(memberId));
    }

    @PutMapping("/methods")
    public ApiResponse<Void> updateMethod(
            @AuthenticationPrincipal Long memberId,
            @RequestBody @Valid UpdatePaymentMethodRequest req) {
        paymentService.updateMethod(memberId, req);
        return ApiResponse.success(null);
    }

    @GetMapping("/history")
    public ApiResponse<Page<PaymentHistoryResponse>> getHistory(
            @AuthenticationPrincipal Long memberId,
            @PageableDefault(size = 20) Pageable pageable) {
        return ApiResponse.success(paymentService.getHistory(memberId, pageable));
    }
}
```

```bash
./mvnw test -Dtest=PaymentControllerTest
git commit -m "feat: 결제수단 조회/변경, 결제내역 API 구현"
```

---

## ═══════════════════════════════════════
## SUBSYSTEM 3: 포토북 API
## ═══════════════════════════════════════

### Task 8: 포토북 목록·상세 조회 API

**Endpoints:**
- `GET /api/v1/photobooks` — 포토북 목록 (월별 타임라인)
- `GET /api/v1/photobooks/{id}` — 상세
- `GET /api/v1/photobooks/viewer/{shareToken}` — 비회원 뷰어 (토큰 기반)

**Files:**
- Create: `src/main/java/com/snaps/subscription/api/photobook/PhotobookController.java`
- Create: `src/main/java/com/snaps/subscription/api/photobook/PhotobookService.java`

**GET /api/v1/photobooks Response:**
```json
{
  "code": "SUCCESS",
  "data": [
    {
      "photobookId": 10,
      "targetYearMonth": "2026-04",
      "title": "4월의 기억",
      "thumbnailUrl": "https://cdn.snaps.com/pb/10/cover.jpg",
      "status": "PUBLISHED",
      "publishedAt": "2026-04-10T09:00:00"
    }
  ]
}
```

**GET /api/v1/photobooks/viewer/{shareToken} Response:**
```json
{
  "code": "SUCCESS",
  "data": {
    "photobookId": 10,
    "title": "4월의 기억",
    "pages": [
      { "pageNo": 1, "imageUrl": "https://...", "caption": "우리 가족" }
    ],
    "canOrderPhysical": true,
    "physicalOrderUrl": "/api/v1/deliveries/recipient-order"
  }
}
```

- [ ] **Step 1: 테스트 작성**

```java
@Test
void viewer_accessible_without_auth() throws Exception {
    given(photobookService.getViewerByToken("valid_token"))
        .willReturn(viewerResponse);

    mockMvc.perform(get("/api/v1/photobooks/viewer/valid_token"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.data.canOrderPhysical").value(true));
}
```

- [ ] **Step 2: PhotobookService 구현**

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class PhotobookService {

    private final PhotobookRepository photobookRepository;
    private final PhotobookPageRepository pageRepository;

    public List<PhotobookSummaryResponse> getMyPhotobooks(Long memberId) {
        return photobookRepository
            .findByMemberIdOrderByTargetYearMonthDesc(memberId)
            .stream()
            .map(pb -> new PhotobookSummaryResponse(
                pb.getId(), pb.getTargetYearMonth(), pb.getTitle(),
                buildThumbnailUrl(pb), pb.getStatus().name(), pb.getPublishedAt()))
            .toList();
    }

    public PhotobookViewerResponse getViewerByToken(String shareToken) {
        Photobook pb = photobookRepository.findByShareToken(shareToken)
            .orElseThrow(() -> new SubscriptionException("E_NOT_FOUND", "포토북을 찾을 수 없습니다."));

        List<PhotobookPageDto> pages = pageRepository.findByPhotobookIdOrderByPageNo(pb.getId())
            .stream()
            .map(p -> new PhotobookPageDto(p.getPageNo(), p.getImageUrl(), p.getCaption()))
            .toList();

        return new PhotobookViewerResponse(
            pb.getId(), pb.getTitle(), pages,
            true,   // canOrderPhysical: 수신자는 항상 실물 주문 가능
            "/api/v1/deliveries/recipient-order");
    }

    private String buildThumbnailUrl(Photobook pb) {
        return "https://cdn.snaps.com/pb/" + pb.getId() + "/cover.jpg";
    }
}
```

- [ ] **Step 3: Controller 구현 및 커밋**

```java
@RestController
@RequestMapping("/api/v1/photobooks")
@RequiredArgsConstructor
public class PhotobookController {

    private final PhotobookService photobookService;

    @GetMapping
    public ApiResponse<List<PhotobookSummaryResponse>> getMyPhotobooks(
            @AuthenticationPrincipal Long memberId) {
        return ApiResponse.success(photobookService.getMyPhotobooks(memberId));
    }

    @GetMapping("/{id}")
    public ApiResponse<PhotobookDetailResponse> getPhotobook(
            @AuthenticationPrincipal Long memberId,
            @PathVariable Long id) {
        return ApiResponse.success(photobookService.getPhotobook(memberId, id));
    }

    // 비회원 접근 — Security permitAll 설정 필요
    @GetMapping("/viewer/{shareToken}")
    public ApiResponse<PhotobookViewerResponse> getViewer(@PathVariable String shareToken) {
        return ApiResponse.success(photobookService.getViewerByToken(shareToken));
    }
}
```

```bash
git commit -m "feat: 포토북 목록·상세·비회원 뷰어 API 구현"
```

---

### Task 9: 커버 커스텀 API

**Endpoint:** `PATCH /api/v1/photobooks/{id}/cover`

**Request:**
```json
{
  "coverColor": "#F5F0E8",
  "coverText": "우리 가족의 4월"
}
```

**비즈니스 규칙:**
- 커버텍스트 최대 20자
- PUBLISHED 상태는 수정 불가

- [ ] **Step 1: 테스트 작성**

```java
@Test
@WithMockUser(username = "1001")
void updateCover_too_long_text_returns_400() throws Exception {
    String tooLong = "이건 21자가 넘는 텍스트입니다!!!!!";
    mockMvc.perform(patch("/api/v1/photobooks/1/cover")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"coverColor\":\"#FFF\",\"coverText\":\"" + tooLong + "\"}"))
        .andExpect(status().isBadRequest());
}
```

- [ ] **Step 2: Service 메서드 구현**

```java
@Transactional
public void updateCover(Long memberId, Long photobookId, UpdateCoverRequest req) {
    Photobook pb = photobookRepository.findById(photobookId)
        .filter(p -> p.getMemberId().equals(memberId))
        .orElseThrow(() -> new SubscriptionException("E_NOT_FOUND", "포토북을 찾을 수 없습니다."));

    if (pb.getStatus() == PhotobookStatus.PUBLISHED) {
        throw new SubscriptionException("E_ALREADY_PUBLISHED", "발행된 포토북은 수정할 수 없습니다.");
    }

    pb.updateCover(req.coverColor(), req.coverText()); // 도메인에서 20자 검증
}
```

- [ ] **Step 3: Controller 엔드포인트 추가 후 커밋**

```java
@PatchMapping("/{id}/cover")
public ApiResponse<Void> updateCover(
        @AuthenticationPrincipal Long memberId,
        @PathVariable Long id,
        @RequestBody UpdateCoverRequest req) {
    photobookService.updateCover(memberId, id, req);
    return ApiResponse.success(null);
}
```

```bash
git commit -m "feat: 포토북 커버 커스텀 API 구현"
```

---

## ═══════════════════════════════════════
## SUBSYSTEM 4: 공유·수신자 API
## ═══════════════════════════════════════

### Task 10: 공유 대상자 CRUD API

**Endpoints:**
- `GET /api/v1/share/recipients` — 목록
- `POST /api/v1/share/recipients` — 추가
- `PATCH /api/v1/share/recipients/{id}` — 수정 (별명, 발송 ON/OFF)
- `DELETE /api/v1/share/recipients/{id}` — 삭제

**Files:**
- Create: `src/main/java/com/snaps/subscription/api/share/ShareController.java`
- Create: `src/main/java/com/snaps/subscription/api/share/ShareService.java`

**POST /api/v1/share/recipients Request:**
```json
{
  "nickname": "엄마",
  "contactType": "KAKAO",
  "contactValue": "kakao_uuid_xxx",
  "notificationEnabled": true
}
```

- [ ] **Step 1: 테스트 작성**

```java
@Test
@WithMockUser(username = "1001")
void addRecipient_returns_201() throws Exception {
    AddRecipientRequest req = new AddRecipientRequest("엄마", "KAKAO", "uuid_xxx", true);
    given(shareService.addRecipient(eq(1001L), any())).willReturn(new RecipientResponse(1L, "엄마", "KAKAO", true));

    mockMvc.perform(post("/api/v1/share/recipients")
            .contentType(MediaType.APPLICATION_JSON)
            .content(objectMapper.writeValueAsString(req)))
        .andExpect(status().isCreated())
        .andExpect(jsonPath("$.data.nickname").value("엄마"));
}
```

- [ ] **Step 2: ShareService 구현**

```java
@Service
@RequiredArgsConstructor
@Transactional
public class ShareService {

    private final ShareRecipientRepository recipientRepository;
    private final ShareHistoryRepository historyRepository;
    private final SubscriptionRepository subscriptionRepository;

    public List<RecipientResponse> getRecipients(Long memberId) {
        return recipientRepository.findByMemberId(memberId)
            .stream()
            .map(r -> new RecipientResponse(r.getId(), r.getNickname(),
                r.getContactType(), r.isNotificationEnabled()))
            .toList();
    }

    public RecipientResponse addRecipient(Long memberId, AddRecipientRequest req) {
        Subscription sub = subscriptionRepository.findByMemberIdAndStatusIn(
            memberId, List.of(SubscriptionStatus.ACTIVE))
            .orElseThrow(() -> new SubscriptionException("E_NO_SUBSCRIPTION", "활성 구독이 없습니다."));

        ShareRecipient recipient = ShareRecipient.builder()
            .memberId(memberId)
            .subscriptionId(sub.getId())
            .nickname(req.nickname())
            .contactType(req.contactType())
            .contactValue(req.contactValue())
            .notificationEnabled(req.notificationEnabled())
            .build();
        recipientRepository.save(recipient);

        return new RecipientResponse(recipient.getId(), recipient.getNickname(),
            recipient.getContactType(), recipient.isNotificationEnabled());
    }

    public void updateRecipient(Long memberId, Long recipientId, UpdateRecipientRequest req) {
        ShareRecipient r = recipientRepository.findById(recipientId)
            .filter(rec -> rec.getMemberId().equals(memberId))
            .orElseThrow(() -> new SubscriptionException("E_NOT_FOUND", "대상자를 찾을 수 없습니다."));
        r.update(req.nickname(), req.notificationEnabled());
    }

    public void deleteRecipient(Long memberId, Long recipientId) {
        ShareRecipient r = recipientRepository.findById(recipientId)
            .filter(rec -> rec.getMemberId().equals(memberId))
            .orElseThrow(() -> new SubscriptionException("E_NOT_FOUND", "대상자를 찾을 수 없습니다."));
        recipientRepository.delete(r);
    }
}
```

- [ ] **Step 3: Controller 구현 후 커밋**

```java
@RestController
@RequestMapping("/api/v1/share")
@RequiredArgsConstructor
public class ShareController {

    private final ShareService shareService;

    @GetMapping("/recipients")
    public ApiResponse<List<RecipientResponse>> getRecipients(@AuthenticationPrincipal Long memberId) {
        return ApiResponse.success(shareService.getRecipients(memberId));
    }

    @PostMapping("/recipients")
    @ResponseStatus(HttpStatus.CREATED)
    public ApiResponse<RecipientResponse> addRecipient(
            @AuthenticationPrincipal Long memberId,
            @RequestBody @Valid AddRecipientRequest req) {
        return ApiResponse.success(shareService.addRecipient(memberId, req));
    }

    @PatchMapping("/recipients/{id}")
    public ApiResponse<Void> updateRecipient(
            @AuthenticationPrincipal Long memberId,
            @PathVariable Long id,
            @RequestBody UpdateRecipientRequest req) {
        shareService.updateRecipient(memberId, id, req);
        return ApiResponse.success(null);
    }

    @DeleteMapping("/recipients/{id}")
    public ApiResponse<Void> deleteRecipient(
            @AuthenticationPrincipal Long memberId,
            @PathVariable Long id) {
        shareService.deleteRecipient(memberId, id);
        return ApiResponse.success(null);
    }
}
```

```bash
git commit -m "feat: 공유 대상자 CRUD API 구현"
```

---

### Task 11: 공유 발송이력 & 추가 공유 API

**Endpoints:**
- `GET /api/v1/share/history` — 발송 이력
- `POST /api/v1/photobooks/{id}/share` — 추가 공유 (카카오/링크 복사)
- `POST /api/v1/share/history/{id}/read` — 열람 확인 (수신자 웹뷰어에서 호출)

**POST /api/v1/photobooks/{id}/share Request:**
```json
{
  "channel": "KAKAO",
  "recipientIds": [1, 2]
}
```

- [ ] **Step 1: Service 메서드 구현**

```java
public List<ShareHistoryResponse> getHistory(Long memberId) {
    return historyRepository.findByRecipientMemberIdOrderBySentAtDesc(memberId)
        .stream()
        .map(h -> new ShareHistoryResponse(
            h.getId(), h.getPhotobookId(), h.getChannel(),
            h.getStatus().name(), h.getSentAt(), h.getReadAt()))
        .toList();
}

public void sendExtra(Long memberId, Long photobookId, ExtraShareRequest req) {
    Photobook pb = photobookRepository.findById(photobookId)
        .filter(p -> p.getMemberId().equals(memberId))
        .orElseThrow(() -> new SubscriptionException("E_NOT_FOUND", "포토북을 찾을 수 없습니다."));

    req.recipientIds().forEach(recipientId -> {
        ShareRecipient recipient = recipientRepository.findById(recipientId)
            .orElseThrow(() -> new SubscriptionException("E_NOT_FOUND", "대상자 없음"));

        shareNotificationClient.send(pb.getShareToken(), recipient, req.channel());

        ShareHistory history = ShareHistory.builder()
            .photobookId(photobookId)
            .recipientId(recipientId)
            .channel(req.channel())
            .status(ShareStatus.SENT)
            .sentAt(LocalDateTime.now())
            .build();
        historyRepository.save(history);
    });
}

public void markRead(Long historyId) {
    ShareHistory h = historyRepository.findById(historyId)
        .orElseThrow(() -> new SubscriptionException("E_NOT_FOUND", "이력 없음"));
    h = ShareHistory.builder()
        .id(h.getId()).photobookId(h.getPhotobookId()).recipientId(h.getRecipientId())
        .channel(h.getChannel()).status(ShareStatus.READ)
        .sentAt(h.getSentAt()).readAt(LocalDateTime.now())
        .build();
    historyRepository.save(h);
}
```

- [ ] **Step 2: Controller 엔드포인트 추가 후 커밋**

```bash
git commit -m "feat: 공유 발송이력 조회, 추가 공유, 열람 확인 API 구현"
```

---

## ═══════════════════════════════════════
## SUBSYSTEM 5: 배송 API
## ═══════════════════════════════════════

### Task 12: 배송 조회 & 수신자 실물 주문 API

**Endpoints:**
- `GET /api/v1/deliveries` — 내 배송 목록
- `GET /api/v1/deliveries/{id}` — 배송 상세 (트래킹 포함)
- `POST /api/v1/deliveries/recipient-order` — 수신자 실물 주문

**Files:**
- Create: `src/main/java/com/snaps/subscription/api/delivery/DeliveryController.java`
- Create: `src/main/java/com/snaps/subscription/api/delivery/DeliveryService.java`

**GET /api/v1/deliveries/{id} Response:**
```json
{
  "code": "SUCCESS",
  "data": {
    "deliveryId": 5,
    "orderType": "AUTO",
    "status": "SHIPPING",
    "trackingNo": "1234567890",
    "courier": "CJ대한통운",
    "steps": [
      { "step": "PENDING", "completed": true },
      { "step": "MANUFACTURING", "completed": true },
      { "step": "SHIPPING", "completed": true },
      { "step": "DELIVERED", "completed": false }
    ]
  }
}
```

**POST /api/v1/deliveries/recipient-order Request:**
```json
{
  "shareToken": "abc_share_token",
  "recipientName": "홍길동",
  "phone": "010-1234-5678",
  "zipcode": "06000",
  "address": "서울시 강남구 테헤란로 123",
  "addressDetail": "101동 202호",
  "pgBillingKey": "billing_key_xxx",
  "amount": 5900
}
```

- [ ] **Step 1: DeliveryService 구현**

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class DeliveryService {

    private final DeliveryRepository deliveryRepository;
    private final PhotobookRepository photobookRepository;
    private final PodClient podClient;
    private final PgClient pgClient;

    public List<DeliveryResponse> getMyDeliveries(Long memberId) {
        return deliveryRepository.findByMemberIdOrderByOrderedAtDesc(memberId)
            .stream()
            .map(d -> new DeliveryResponse(d.getId(), d.getOrderType(),
                d.getStatus().name(), d.getTrackingNo(), d.getCourier()))
            .toList();
    }

    public DeliveryDetailResponse getDeliveryDetail(Long memberId, Long deliveryId) {
        DeliveryOrder order = deliveryRepository.findById(deliveryId)
            .filter(d -> d.getMemberId().equals(memberId))
            .orElseThrow(() -> new SubscriptionException("E_NOT_FOUND", "배송 정보를 찾을 수 없습니다."));

        List<DeliveryStepDto> steps = buildSteps(order.getStatus());
        return new DeliveryDetailResponse(
            order.getId(), order.getOrderType(), order.getStatus().name(),
            order.getTrackingNo(), order.getCourier(), steps);
    }

    @Transactional
    public void createRecipientOrder(RecipientOrderRequest req) {
        // 토큰으로 포토북 조회
        Photobook pb = photobookRepository.findByShareToken(req.shareToken())
            .orElseThrow(() -> new SubscriptionException("E_NOT_FOUND", "포토북을 찾을 수 없습니다."));

        // 결제
        pgClient.chargeImmediately(req.pgBillingKey(), req.amount());

        // 배송 주문 생성
        DeliveryOrder order = DeliveryOrder.builder()
            .subscriptionId(pb.getSubscriptionId())
            .memberId(pb.getMemberId())
            .photobookId(pb.getId())
            .orderType("RECIPIENT_ORDER")
            .status(DeliveryStatus.PENDING)
            .recipientName(req.recipientName())
            .phone(req.phone())
            .zipcode(req.zipcode())
            .address(req.address())
            .addressDetail(req.addressDetail())
            .build();
        deliveryRepository.save(order);

        // POD 파이프라인에 발주
        String podOrderId = podClient.createOrder(pb.getId(), order);
        order.setPodOrderId(podOrderId);
    }

    private List<DeliveryStepDto> buildSteps(DeliveryStatus current) {
        List<DeliveryStatus> steps = List.of(
            DeliveryStatus.PENDING, DeliveryStatus.MANUFACTURING,
            DeliveryStatus.SHIPPING, DeliveryStatus.DELIVERED);
        int currentIdx = steps.indexOf(current);
        return IntStream.range(0, steps.size())
            .mapToObj(i -> new DeliveryStepDto(steps.get(i).name(), i <= currentIdx))
            .toList();
    }
}
```

- [ ] **Step 2: Controller 구현 후 커밋**

```bash
git commit -m "feat: 배송 조회, 수신자 실물 주문 API 구현"
```

---

### Task 13: Admin 구독관리 API

**Endpoints:**
- `GET /api/v1/admin/subscriptions` — 전체 구독자 목록 (검색·페이징)
- `GET /api/v1/admin/subscriptions/{id}` — 상세
- `PATCH /api/v1/admin/subscriptions/{id}` — 상태 수동 변경
- `GET /api/v1/admin/deliveries` — 전체 배송 목록
- `PATCH /api/v1/admin/deliveries/{id}/tracking` — 배송 추적 정보 수동 입력

**Files:**
- Create: `src/main/java/com/snaps/subscription/api/admin/AdminSubscriptionController.java`
- Create: `src/main/java/com/snaps/subscription/api/admin/AdminDeliveryController.java`

**GET /api/v1/admin/subscriptions Query Params:**
- `memberId`, `status`, `planCode`, `page`, `size`

**PATCH /api/v1/admin/deliveries/{id}/tracking Request:**
```json
{
  "courier": "CJ대한통운",
  "trackingNo": "1234567890",
  "status": "SHIPPING"
}
```

- [ ] **Step 1: Admin Security 설정**

```java
// SecurityConfig에 Admin 역할 제한 추가
.requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
```

- [ ] **Step 2: AdminSubscriptionController 구현 후 커밋**

```bash
git commit -m "feat: 어드민 구독관리 API 구현"
```

---

## ═══════════════════════════════════════
## SUBSYSTEM 6: Spring Batch 잡
## ═══════════════════════════════════════

### Task 14: 월정액 정기결제 배치

**파일:**
- Create: `src/main/java/com/snaps/subscription/batch/billing/MonthlyBillingJobConfig.java`
- Create: `src/main/java/com/snaps/subscription/batch/billing/BillingItemReader.java`
- Create: `src/main/java/com/snaps/subscription/batch/billing/BillingItemProcessor.java`
- Create: `src/main/java/com/snaps/subscription/batch/billing/BillingItemWriter.java`
- Test: `src/test/java/com/snaps/subscription/batch/billing/MonthlyBillingJobTest.java`

**실행 일정:** 매월 1일 오전 9시 (또는 구독 가입일 기준 — 협의 필요)
**Cron:** `0 0 9 1 * *`

**처리 흐름:**
1. `status=ACTIVE AND next_billing_date <= today` 구독 조회
2. 결제수단으로 PG사 정기결제 요청
3. 성공: `payment_history` 저장, `next_billing_date += 1개월`
4. 실패: retry_count++, 3회 초과 시 구독 `PAYMENT_FAILED` 상태로 변경
5. 결제 결과 알림 발송 (카카오/SMS)

- [ ] **Step 1: 배치 통합 테스트 작성**

```java
@SpringBatchTest
@SpringBootTest
class MonthlyBillingJobTest {

    @Autowired JobLauncherTestUtils jobLauncherTestUtils;
    @Autowired SubscriptionRepository subscriptionRepository;
    @Autowired PaymentHistoryRepository paymentHistoryRepository;
    @MockBean PgClient pgClient;

    @Test
    void billing_creates_payment_history_on_success() throws Exception {
        Subscription sub = subscriptionRepository.save(Subscription.builder()
            .memberId(9001L).planCode("LIGHT").status(SubscriptionStatus.ACTIVE)
            .startedAt(LocalDateTime.now().minusDays(30))
            .nextBillingDate(LocalDate.now()).nextGenerationDate(LocalDate.now())
            .build());

        given(pgClient.chargeImmediately(any(), eq(5900)))
            .willReturn(new PgResponse("txn_test_001", "SUCCESS"));

        JobExecution execution = jobLauncherTestUtils.launchJob();

        assertThat(execution.getStatus()).isEqualTo(BatchStatus.COMPLETED);

        List<PaymentHistory> histories = paymentHistoryRepository.findBySubscriptionId(sub.getId());
        assertThat(histories).hasSize(1);
        assertThat(histories.get(0).getStatus()).isEqualTo(PaymentStatus.SUCCESS);

        Subscription updated = subscriptionRepository.findById(sub.getId()).get();
        assertThat(updated.getNextBillingDate()).isEqualTo(LocalDate.now().plusMonths(1));
    }

    @Test
    void billing_marks_payment_failed_after_3_retries() throws Exception {
        // 이미 retry_count=2인 구독 준비
        // pgClient 실패 응답 mock
        // 배치 실행 후 status=PAYMENT_FAILED 확인
    }
}
```

- [ ] **Step 2: BillingItemReader 구현**

```java
@Component
@StepScope
public class BillingItemReader implements ItemReader<Subscription> {

    private final SubscriptionRepository subscriptionRepository;
    private Iterator<Subscription> iterator;

    public BillingItemReader(SubscriptionRepository subscriptionRepository) {
        this.subscriptionRepository = subscriptionRepository;
    }

    @Override
    public Subscription read() {
        if (iterator == null) {
            List<Subscription> targets = subscriptionRepository
                .findByStatusAndNextBillingDateLessThanEqual(
                    SubscriptionStatus.ACTIVE, LocalDate.now());
            iterator = targets.iterator();
        }
        return iterator.hasNext() ? iterator.next() : null;
    }
}
```

- [ ] **Step 3: BillingItemProcessor 구현**

```java
@Component
@StepScope
@RequiredArgsConstructor
public class BillingItemProcessor implements ItemProcessor<Subscription, PaymentResult> {

    private final PaymentMethodRepository paymentMethodRepository;
    private final PgClient pgClient;

    @Override
    public PaymentResult process(Subscription sub) {
        Optional<PaymentMethod> method = paymentMethodRepository
            .findByMemberIdAndIsDefaultTrue(sub.getMemberId());

        if (method.isEmpty()) {
            return new PaymentResult(sub, null, PaymentStatus.FAILED, "결제수단 없음");
        }

        int amount = switch (sub.getPlanCode()) {
            case "DIGITAL" -> 2900;
            case "SEASON"  -> 3900;
            case "LIGHT"   -> 5900;
            default -> 0;
        };

        try {
            PgResponse pgRes = pgClient.chargeImmediately(method.get().getPgBillingKey(), amount);
            return new PaymentResult(sub, pgRes.transactionId(), PaymentStatus.SUCCESS, null);
        } catch (PgException e) {
            return new PaymentResult(sub, null, PaymentStatus.FAILED, e.getMessage());
        }
    }
}
```

- [ ] **Step 4: BillingItemWriter 구현**

```java
@Component
@StepScope
@RequiredArgsConstructor
public class BillingItemWriter implements ItemWriter<PaymentResult> {

    private final SubscriptionRepository subscriptionRepository;
    private final PaymentHistoryRepository paymentHistoryRepository;
    private final ApplicationEventPublisher eventPublisher;

    private static final int MAX_RETRY = 3;

    @Override
    public void write(Chunk<? extends PaymentResult> chunk) {
        chunk.getItems().forEach(result -> {
            PaymentHistory history = PaymentHistory.builder()
                .subscriptionId(result.subscription().getId())
                .memberId(result.subscription().getMemberId())
                .amount(result.subscription().getPlanCode().equals("LIGHT") ? 5900
                    : result.subscription().getPlanCode().equals("SEASON") ? 3900 : 2900)
                .status(result.status())
                .pgTransactionId(result.transactionId())
                .billingDate(LocalDate.now())
                .failReason(result.failReason())
                .build();
            paymentHistoryRepository.save(history);

            if (result.status() == PaymentStatus.SUCCESS) {
                result.subscription().advanceBillingDate(1);
                subscriptionRepository.save(result.subscription());
                eventPublisher.publishEvent(new BillingSuccessEvent(result.subscription().getMemberId()));
            } else {
                // 실패 처리: 다음 결제 재시도 또는 구독 정지
                int retryCount = paymentHistoryRepository
                    .countBySubscriptionIdAndStatus(result.subscription().getId(), PaymentStatus.FAILED);

                if (retryCount >= MAX_RETRY) {
                    result.subscription().markPaymentFailed();
                    subscriptionRepository.save(result.subscription());
                    eventPublisher.publishEvent(new BillingFailedEvent(result.subscription().getMemberId()));
                }
            }
        });
    }
}
```

- [ ] **Step 5: Job Configuration**

```java
@Configuration
@EnableBatchProcessing
@RequiredArgsConstructor
public class MonthlyBillingJobConfig {

    private final JobRepository jobRepository;
    private final PlatformTransactionManager transactionManager;
    private final BillingItemReader reader;
    private final BillingItemProcessor processor;
    private final BillingItemWriter writer;

    @Bean
    public Job monthlyBillingJob() {
        return new JobBuilder("monthlyBillingJob", jobRepository)
            .start(billingStep())
            .build();
    }

    @Bean
    public Step billingStep() {
        return new StepBuilder("billingStep", jobRepository)
            .<Subscription, PaymentResult>chunk(10, transactionManager)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .faultTolerant()
            .skip(PgException.class)
            .skipLimit(100)
            .build();
    }
}
```

- [ ] **Step 6: Cron 스케줄 등록**

```java
@Component
@RequiredArgsConstructor
public class BillingJobScheduler {

    private final JobLauncher jobLauncher;
    private final Job monthlyBillingJob;

    @Scheduled(cron = "0 0 9 1 * *") // 매월 1일 오전 9시
    public void runMonthlyBilling() throws Exception {
        JobParameters params = new JobParametersBuilder()
            .addLocalDate("runDate", LocalDate.now())
            .toJobParameters();
        jobLauncher.run(monthlyBillingJob, params);
    }
}
```

- [ ] **Step 7: 테스트 통과 후 커밋**

```bash
./mvnw test -Dtest=MonthlyBillingJobTest
git commit -m "feat: 월정액 정기결제 Spring Batch 잡 구현"
```

---

### Task 15: 포토북 자동 생성 트리거 배치

**파일:**
- Create: `src/main/java/com/snaps/subscription/batch/photobook/PhotobookGenerationJobConfig.java`

**실행 일정:** `0 0 10 1 * *` — 매월 1일 오전 10시 (monthlyBillingJob 완료 1시간 후)

**처리 흐름:**
1. DB 폴링 — `status=ACTIVE AND payment_history에 이번 달 SUCCESS 있음 AND photobook에 이번 달 없는 구독` 조회
2. AI 분석 서비스 호출 → 사진 분석 요청
3. `photobook` 레코드 `GENERATING` 상태로 생성
4. 사진 없을 경우 건너뜀 + ApplicationEventPublisher로 알림 이벤트 발행
5. 분석 완료 후 (비동기 @Async) `READY` 상태로 업데이트

- [ ] **Step 1: 테스트 작성**

```java
@Test
void generation_job_skips_if_photobook_exists_this_month() throws Exception {
    // 이번 달 포토북이 이미 있는 구독 세팅
    // 배치 실행
    // AI 분석 요청이 호출되지 않았음을 확인
    verify(photoAnalysisClient, never()).analyzePhotos(any(), any());
}
```

- [ ] **Step 2: Processor 구현**

```java
@Component
@StepScope
@RequiredArgsConstructor
public class PhotobookGenerationProcessor implements ItemProcessor<Subscription, PhotobookGenerationResult> {

    private final PhotobookRepository photobookRepository;
    private final PhotoAnalysisClient photoAnalysisClient;

    @Override
    public PhotobookGenerationResult process(Subscription sub) {
        String thisMonth = YearMonth.now().toString(); // YYYY-MM

        // 이미 생성됐으면 SKIP
        boolean exists = photobookRepository.existsBySubscriptionIdAndTargetYearMonth(
            sub.getId(), thisMonth);
        if (exists) return null;

        // AI 분석 요청
        PhotoAnalysisResponse analysis = photoAnalysisClient.analyzePhotos(
            sub.getMemberId(), thisMonth);

        if (analysis.photoCount() == 0) {
            // 사진 없음 이벤트 발행 (알림)
            return new PhotobookGenerationResult(sub, null, false);
        }

        // 포토북 레코드 GENERATING 상태로 생성
        Photobook pb = Photobook.builder()
            .subscriptionId(sub.getId())
            .memberId(sub.getMemberId())
            .status(PhotobookStatus.GENERATING)
            .targetYearMonth(thisMonth)
            .coverDate(YearMonth.now().getMonth().getDisplayName(TextStyle.FULL, Locale.KOREAN) + " " + YearMonth.now().getYear())
            .build();
        photobookRepository.save(pb);

        return new PhotobookGenerationResult(sub, pb, true);
    }
}
```

- [ ] **Step 3: Job Config 및 Scheduler 등록 후 커밋**

```bash
git commit -m "feat: 포토북 자동 생성 트리거 배치 구현"
```

---

### Task 16: 공유 알림 발송 배치

**파일:**
- Create: `src/main/java/com/snaps/subscription/batch/share/ShareNotificationJobConfig.java`

**실행 일정:** `0 0 11 1 * *` — 매월 1일 오전 11시 (photobookGenerationJob 완료 1시간 후)

**DB 폴링 조건:** `photobook.status = 'READY' AND share_history에 해당 photobook_id 발송 이력 없음`

**처리 흐름:**
1. 포토북 ID로 구독 조회
2. 해당 구독의 공유 대상자 목록 조회 (`notification_enabled=true`)
3. 카카오 알림톡 발송 시도
4. 카카오 실패 시 SMS fallback
5. `share_history` 저장

- [ ] **Step 1: 테스트 작성**

```java
@Test
void send_kakao_success_saves_sent_history() {
    // given: photobookId, 수신자 설정
    // given: kakaoClient.send 성공
    // when: job 실행
    // then: share_history.status = SENT, channel = KAKAO
}

@Test
void kakao_failure_falls_back_to_sms() {
    // given: kakaoClient.send throws exception
    // given: smsClient.send 성공
    // then: share_history.status = SENT, channel = SMS
}
```

- [ ] **Step 2: Processor 구현**

```java
@Component
@StepScope
@RequiredArgsConstructor
public class ShareNotificationProcessor implements ItemProcessor<ShareRecipient, ShareNotificationResult> {

    private final KakaoAlimtalkClient kakaoClient;
    private final SmsClient smsClient;
    private final PhotobookRepository photobookRepository;

    // photobookId는 JobParameter로 주입
    @Value("#{jobParameters['photobookId']}")
    private Long photobookId;

    @Override
    public ShareNotificationResult process(ShareRecipient recipient) {
        Photobook pb = photobookRepository.findById(photobookId)
            .orElseThrow(() -> new SubscriptionException("E_NOT_FOUND", "포토북 없음"));

        String viewerUrl = "https://snaps.com/view/" + pb.getShareToken();
        String channel;
        boolean success;

        if ("KAKAO".equals(recipient.getContactType())) {
            try {
                kakaoClient.sendAlimtalk(recipient.getContactValue(), pb.getTitle(), viewerUrl);
                channel = "KAKAO";
                success = true;
            } catch (Exception e) {
                // SMS fallback
                smsClient.send(recipient.getContactValue(), "[스냅스] 포토북이 도착했어요! " + viewerUrl);
                channel = "SMS";
                success = true;
            }
        } else {
            smsClient.send(recipient.getContactValue(), "[스냅스] 포토북이 도착했어요! " + viewerUrl);
            channel = "SMS";
            success = true;
        }

        return new ShareNotificationResult(photobookId, recipient.getId(), channel, success);
    }
}
```

- [ ] **Step 3: Writer — share_history 저장 후 커밋**

```bash
git commit -m "feat: 공유 알림 발송 배치 구현 (카카오 알림톡 + SMS fallback)"
```

---

### Task 17: 실물 배송 자동 발주 배치

**파일:**
- Create: `src/main/java/com/snaps/subscription/batch/delivery/DeliveryOrderJobConfig.java`

**실행 일정:**
- 라이트 요금제: 포토북 PUBLISHED 이벤트 직후 즉시 (`매월`)
- 시즌 요금제: 매 분기 말 (`0 0 9 1 3,6,9,12 *`)

**처리 흐름:**
1. 자동 발주 대상 조회 (요금제별 주기 체크)
2. 이번 주기에 이미 발주됐으면 SKIP
3. POD 파이프라인에 생산 주문 생성
4. `delivery_order` 레코드 생성
5. 배송 상태 추적 시작 (POD webhook 또는 polling)

- [ ] **Step 1: Processor 구현**

```java
@Component
@StepScope
@RequiredArgsConstructor
public class DeliveryOrderProcessor implements ItemProcessor<Subscription, DeliveryOrderResult> {

    private final DeliveryRepository deliveryRepository;
    private final PhotobookRepository photobookRepository;
    private final PodClient podClient;

    @Override
    public DeliveryOrderResult process(Subscription sub) {
        String thisMonth = YearMonth.now().toString();

        // 이미 이번 달 발주됐으면 SKIP
        boolean alreadyOrdered = deliveryRepository
            .existsBySubscriptionIdAndOrderTypeAndOrderedAtYearMonth(
                sub.getId(), "AUTO", thisMonth);
        if (alreadyOrdered) return null;

        // 시즌 요금제는 분기 체크
        if ("SEASON".equals(sub.getPlanCode())) {
            int month = LocalDate.now().getMonthValue();
            if (!List.of(3, 6, 9, 12).contains(month)) return null;
        }

        // 최근 포토북 조회
        Photobook pb = photobookRepository
            .findFirstBySubscriptionIdOrderByTargetYearMonthDesc(sub.getId())
            .orElse(null);
        if (pb == null) return null;

        // POD 발주
        String podOrderId = podClient.createOrder(pb.getId(), sub.getMemberId());

        DeliveryOrder order = DeliveryOrder.builder()
            .subscriptionId(sub.getId())
            .memberId(sub.getMemberId())
            .photobookId(pb.getId())
            .orderType("AUTO")
            .status(DeliveryStatus.MANUFACTURING)
            .podOrderId(podOrderId)
            .build();
        deliveryRepository.save(order);

        return new DeliveryOrderResult(sub, order);
    }
}
```

- [ ] **Step 2: Job Config + Scheduler 설정**

```java
// 라이트: 매월 1일 오후 12시 DB 폴링
@Scheduled(cron = "0 0 12 1 * *")
public void runMonthlyLightDeliveryJob() throws Exception {
    JobParameters params = new JobParametersBuilder()
        .addString("planCode", "LIGHT")
        .addLocalDate("runDate", LocalDate.now())
        .toJobParameters();
    jobLauncher.run(deliveryOrderJob, params);
}

// 시즌: 분기 말(3/6/9/12월) 1일 오전 9시 DB 폴링
@Scheduled(cron = "0 0 9 1 3,6,9,12 *")
public void runQuarterlySeasonDeliveryJob() throws Exception {
    JobParameters params = new JobParametersBuilder()
        .addString("planCode", "SEASON")
        .addLocalDate("runDate", LocalDate.now())
        .toJobParameters();
    jobLauncher.run(deliveryOrderJob, params);
}
```

- [ ] **Step 3: POD 배송상태 polling 배치**

```java
// 매일 오전 6시 배송 상태 업데이트
@Scheduled(cron = "0 0 6 * * *")
public void syncDeliveryStatus() {
    List<DeliveryOrder> inProgress = deliveryRepository
        .findByStatusIn(List.of(DeliveryStatus.MANUFACTURING, DeliveryStatus.SHIPPING));

    inProgress.forEach(order -> {
        PodStatusResponse status = podClient.getOrderStatus(order.getPodOrderId());
        DeliveryStatus newStatus = mapPodStatus(status.code());
        if (newStatus != order.getStatus()) {
            order.updateStatus(newStatus);
            deliveryRepository.save(order);
            // 상태 변경 푸시 알림 발송
            eventPublisher.publishEvent(new DeliveryStatusChangedEvent(
                order.getMemberId(), order.getId(), newStatus));
        }
    });
}
```

- [ ] **Step 4: 커밋**

```bash
git commit -m "feat: 실물 배송 자동 발주 및 배송 상태 동기화 배치 구현"
```

---

### Task 18: 구독 상태 관리 배치

**처리 목록:**
1. 결제 실패 구독 재시도 (D+1, D+3, D+5 재시도)
2. 일시정지 자동 해제 (pause_resume_at <= today)
3. 만료 포토북 EXPIRED 상태 전환 (12개월 이상 경과)

**실행 일정:** 매일 오전 7시 `0 0 7 * * *`

- [ ] **Step 1: Scheduler 구현**

```java
@Component
@RequiredArgsConstructor
public class SubscriptionStateJobScheduler {

    private final SubscriptionRepository subscriptionRepository;
    private final PaymentHistoryRepository paymentHistoryRepository;
    private final PhotobookRepository photobookRepository;
    private final PgClient pgClient;
    private final PaymentMethodRepository paymentMethodRepository;

    @Scheduled(cron = "0 0 7 * * *")
    public void manageSubscriptionStates() {
        retryFailedPayments();
        autoResumeFromPause();
        expireOldPhotobooks();
    }

    // 결제 실패 재시도
    private void retryFailedPayments() {
        List<LocalDate> retryDates = List.of(
            LocalDate.now().minusDays(1),
            LocalDate.now().minusDays(3),
            LocalDate.now().minusDays(5));

        subscriptionRepository.findByStatusAndNextBillingDateLessThanEqual(
            SubscriptionStatus.PAYMENT_FAILED, LocalDate.now())
            .forEach(sub -> {
                paymentMethodRepository.findByMemberIdAndIsDefaultTrue(sub.getMemberId())
                    .ifPresent(method -> {
                        try {
                            int amount = getPlanPrice(sub.getPlanCode());
                            PgResponse res = pgClient.chargeImmediately(method.getPgBillingKey(), amount);
                            // 성공: ACTIVE 복구
                            sub.resume();
                            sub.advanceBillingDate(1);
                            subscriptionRepository.save(sub);
                        } catch (Exception e) {
                            // 재시도도 실패 — 알림만 발송
                        }
                    });
            });
    }

    // 일시정지 자동 해제
    private void autoResumeFromPause() {
        subscriptionRepository.findByStatusAndPauseResumeAtLessThanEqual(
            SubscriptionStatus.PAUSED, LocalDate.now())
            .forEach(sub -> {
                sub.resume();
                subscriptionRepository.save(sub);
            });
    }

    // 만료 포토북 처리
    private void expireOldPhotobooks() {
        String expiryMonth = YearMonth.now().minusMonths(12).toString();
        photobookRepository.findByStatusAndTargetYearMonthLessThan(
            PhotobookStatus.PUBLISHED, expiryMonth)
            .forEach(pb -> {
                // status = EXPIRED 업데이트 (별도 메서드 추가 필요)
            });
    }

    private int getPlanPrice(String planCode) {
        return switch (planCode) {
            case "DIGITAL" -> 2900;
            case "SEASON"  -> 3900;
            case "LIGHT"   -> 5900;
            default -> 0;
        };
    }
}
```

- [ ] **Step 2: 커밋**

```bash
git commit -m "feat: 구독 상태 관리 배치 (결제재시도·일시정지해제·포토북만료) 구현"
```

---

## 📋 API 전체 목록 요약

### 구독 API
| Method | Path | 설명 |
|--------|------|------|
| POST | `/api/v1/subscriptions` | 구독 가입 |
| GET | `/api/v1/subscriptions/me` | 내 구독 현황 |
| GET | `/api/v1/subscriptions/me/dashboard` | 대시보드 종합 |
| PATCH | `/api/v1/subscriptions/me/plan` | 요금제 변경 |
| POST | `/api/v1/subscriptions/me/cancel` | 해지 |
| POST | `/api/v1/subscriptions/me/pause` | 일시정지 |
| POST | `/api/v1/subscriptions/me/resume` | 재개 |

### 결제 API
| Method | Path | 설명 |
|--------|------|------|
| GET | `/api/v1/payments/methods` | 결제수단 조회 |
| PUT | `/api/v1/payments/methods` | 결제수단 변경 |
| GET | `/api/v1/payments/history` | 결제내역 (페이징) |

### 포토북 API
| Method | Path | 설명 |
|--------|------|------|
| GET | `/api/v1/photobooks` | 포토북 목록 |
| GET | `/api/v1/photobooks/{id}` | 상세 |
| PATCH | `/api/v1/photobooks/{id}/cover` | 커버 커스텀 |
| GET | `/api/v1/photobooks/viewer/{shareToken}` | 비회원 뷰어 |
| POST | `/api/v1/photobooks/{id}/share` | 추가 공유 발송 |

### 공유·수신자 API
| Method | Path | 설명 |
|--------|------|------|
| GET | `/api/v1/share/recipients` | 공유 대상자 목록 |
| POST | `/api/v1/share/recipients` | 대상자 추가 |
| PATCH | `/api/v1/share/recipients/{id}` | 대상자 수정 |
| DELETE | `/api/v1/share/recipients/{id}` | 대상자 삭제 |
| GET | `/api/v1/share/history` | 발송 이력 |
| POST | `/api/v1/share/history/{id}/read` | 열람 확인 |

### 배송 API
| Method | Path | 설명 |
|--------|------|------|
| GET | `/api/v1/deliveries` | 배송 목록 |
| GET | `/api/v1/deliveries/{id}` | 배송 상세 + 스텝퍼 |
| POST | `/api/v1/deliveries/recipient-order` | 수신자 실물 주문 |

### 어드민 API
| Method | Path | 설명 |
|--------|------|------|
| GET | `/api/v1/admin/subscriptions` | 구독자 목록 (검색·페이징) |
| GET | `/api/v1/admin/subscriptions/{id}` | 상세 |
| PATCH | `/api/v1/admin/subscriptions/{id}` | 상태 수동 변경 |
| GET | `/api/v1/admin/deliveries` | 전체 배송 목록 |
| PATCH | `/api/v1/admin/deliveries/{id}/tracking` | 배송 추적 수동 입력 |

---

## 📦 배치 잡 전체 목록 요약

| 배치명 | 스케줄 | 폴링 조건 | 설명 |
|--------|--------|-----------|------|
| `monthlyBillingJob` | `0 0 9 1 * *` | status=ACTIVE AND next_billing_date <= today | 월정액 정기결제 |
| `photobookGenerationJob` | `0 0 10 1 * *` | 결제 성공했는데 이번 달 포토북 없는 구독 | 포토북 자동 생성 트리거 |
| `shareNotificationJob` | `0 0 11 1 * *` | status=READY AND 발송이력 없는 포토북 | 공유 알림 발송 (카카오/SMS) |
| `deliveryOrderJob (라이트)` | `0 0 12 1 * *` | LIGHT 요금제 + 이번 달 AUTO 발주 없는 구독 | 라이트 월별 실물 발주 |
| `deliveryOrderJob (시즌)` | `0 0 9 1 3,6,9,12 *` | SEASON 요금제 + 이번 분기 AUTO 발주 없는 구독 | 시즌 분기 실물 발주 |
| `deliveryStatusSyncJob` | `0 0 6 * * *` | status IN (MANUFACTURING, SHIPPING) | POD 배송 상태 동기화 |
| `subscriptionStateJob` | `0 0 7 * * *` | PAYMENT_FAILED / PAUSED 만료 조건 | 결제재시도·일시정지해제·만료처리 |

---

## ❓ 협의 필요 항목 (개발 전 확인)

1. **정기결제 기준일**: 가입일 기준 vs 매월 1일 고정
2. **포토북 생성 트리거**: 결제 성공 직후 vs 매월 고정일 vs 사진 N장 누적
3. **배송 발주 방식**: 자동 발주 vs 고객 최종 확인 후 발주
4. **수신자 실물 주문**: 스냅스 계정 없이 주문 가능 여부
5. **결제 실패 재시도**: 재시도 횟수 및 간격 (제안: D+1, D+3, D+5, 3회 초과 시 정지)
6. **무료체험 조건**: 기간·전환 정책 미정
7. **PG사 선택**: TOSS / KCP / INICIS 중 1차 연동 대상
