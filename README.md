# Snaps 구독 서비스 Java 백엔드 Implementation Plan

> 📂 **이 저장소는 설계·문서 전용입니다.** 실제 구현 코드는 별도 위치에 있습니다.

---

## 📄 문서 위치

| 파일 | 설명 |
|------|------|
| [`docs/superpowers/plans/2026-04-10-snaps-subscription-backend.md`](./docs/superpowers/plans/2026-04-10-snaps-subscription-backend.md) | Implementation Plan (2,945줄) — 18개 Task + 구현 현황 |
| [`docs/superpowers/plans/2026-04-10-snaps-subscription-backend.html`](./docs/superpowers/plans/2026-04-10-snaps-subscription-backend.html) | API 명세 HTML (1,107줄) — 전체 API 요청/응답 JSON 포함 |

---

## 🔗 실제 구현 코드 위치

> **코드는 이 저장소가 아닌, 기존 `snaps-service-api-v2` 프로젝트에 통합되었습니다.**

**Base Path**: `/Users/s-a-039/project/snaps-service-api-v2/`

### 통합 배경
- **선택지 B**: 기존 프로젝트 내부 통합 (Java 8 / Spring MVC / WAR)
- **선택지 A (미채택)**: 별도 Spring Boot 3.x / Java 17 독립 프로젝트
- **사유**: 기존 회원·세션·DB·배포 인프라 재활용

### 구현 파일 트리

```
snaps-service-api-v2/
└── src/main/java/com/snaps/backend/api/
    ├── exception/
    │   └── SubscriptionException.java                                   # 1
    └── server/
        ├── RtCode.java                                                   # +12개 에러코드 추가
        ├── adapter/
        │   ├── subscription/                                              # 인터페이스 3개
        │   │   ├── PgAdapter.java
        │   │   ├── PodAdapter.java
        │   │   └── SubscriptionNotificationAdapter.java
        │   └── kor/web/subscription/                                      # Stub 구현 3개
        │       ├── PgAdapterStub.java
        │       ├── PodAdapterStub.java
        │       └── SubscriptionNotificationAdapterStub.java
        ├── controller/web/subscription/                                   # REST 컨트롤러 6개
        │   ├── SubscriptionController.java
        │   ├── SubscriptionPaymentController.java
        │   ├── SubscriptionPhotobookController.java
        │   ├── SubscriptionShareController.java
        │   ├── SubscriptionDeliveryController.java
        │   └── AdminSubscriptionController.java
        ├── data/
        │   ├── entity/common/subscription/                                # Entity 9 + Enum 5
        │   ├── dto/subscription/                                          # 응답 DTO 13개
        │   └── param/subscription/                                        # 요청 Param 11개
        ├── facade/subscription/                                           # 인터페이스 5 + Impl 5
        │   └── kor/web/
        ├── service/subscription/                                          # 인터페이스 5 + Impl 5
        │   └── kor/web/
        ├── repository/common/subscription/                                # Repository 8 + MyBatis Impl 3
        └── handler/
            └── SubscriptionControllerAdvice.java                          # @ControllerAdvice

snaps-service-api-v2/src/main/resources/META-INF/
├── mybatis/datasource/common/mappers/
│   ├── SubscriptionRepository.xml
│   ├── PaymentHistoryRepository.xml
│   └── DeliveryRepository.xml
└── sql/subscription/
    └── V1__subscription_schema.sql                                        # DDL 10개 테이블
```

**총 87개 Java 파일** (Entity 9 + Enum 5 + Repository 11 + DTO 13 + Param 11 + Service 10 + Facade 10 + Controller 6 + Adapter 6 + etc) + MyBatis XML 3개 + SQL DDL 1개.

---

## 📊 구현 현황 (2026-04-13 기준)

| 구분 | 진행률 | 비고 |
|------|--------|------|
| **API** | **23/27 (85%)** | 구독 7, 결제 3, 포토북 4, 공유 5, 배송 3, 어드민 2 (누락: 어드민 구독 3 + 공유 전체이력 1) |
| **배치** | **0/7 (제외)** | Spring Batch — 별도 프로젝트 권장 |
| **외부 연동** | **0/3 (Stub)** | PG / POD / Notification — 실제 연동 미구현 |
| **DB** | 스키마만 | SQL DDL 작성 완료, 수동 실행 필요 |
| **빌드** | ✅ 검증됨 | 87 .java → 111 .class, 0 에러 (Java 8) |
| **빈 이름 충돌** | ✅ 해결 | 20개 클래스에 `Subscription` prefix 적용 |

---

## 🎯 다음 단계 (간략)

**P1 — 즉시**
1. `V1__subscription_schema.sql` 수동 실행
2. Tomcat 재배포 → 정상 기동 확인

**P2 — API 완성도 (1~2시간)**
3. 어드민 구독 API 3개 추가
4. 공유 전체 발송이력 API 1개 추가

**P3 — 프로덕션 준비**
5. PG 어댑터 실제 구현 (TOSS/KCP/INICIS 중 1차 선택)
6. POD 파이프라인 어댑터 실제 구현
7. 카카오 알림톡 + SMS 어댑터 실제 구현
8. 어드민 권한 체크 AOP 추가

**P4 — 배치 (별도 프로젝트)**
9. `snaps-batch` 프로젝트에 7개 배치 잡 추가

상세는 [플랜 문서](./docs/superpowers/plans/2026-04-10-snaps-subscription-backend.md#-다음-단계-우선순위) 참조.

---

## ❓ 미결정 협의 항목 (7건)

1. 정기결제 기준일 (가입일 vs 매월 1일)
2. 포토북 생성 트리거 방식
3. 배송 발주 방식 (자동 vs 고객 확인)
4. 수신자 실물 주문 비회원 허용 여부
5. 결제 실패 재시도 정책
6. 무료체험 전환 정책
7. 1차 연동 PG사 선택

---

## 📝 문서 작성·수정 흐름

이 저장소에서 **문서만** 관리하고, 코드 변경은 `snaps-service-api-v2`에서 작업합니다.

```
 설계 변경 필요 시         →  superPowers/docs/...plans/*.md 수정
 코드 변경 필요 시         →  snaps-service-api-v2/src/... 수정
 API 명세 변경 시          →  superPowers/docs/...plans/*.html 업데이트
```

문서와 코드의 연결고리는 이 README와 플랜 문서 상단의 "🚧 구현 현황" 섹션에서 유지합니다.
