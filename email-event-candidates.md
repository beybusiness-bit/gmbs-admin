# 이메일 트리거 후보 목록

관리자가 검토 후 원하는 것을 골라 이메일 설정 페이지에 추가하면 됩니다.
새 후보가 생길 때마다 이 파일에 계속 추가합니다.

---

## 현재 구현된 트리거 (이미 이메일 설정 페이지에 있음)

| 트리거 ID | 설명 | 발송 시점 |
|---|---|---|
| `application_received` | 신규 브랜드 등록 신청 접수 | vendor가 신규 입점 신청 제출 시 |
| `join_received` | 기존 브랜드 합류 신청 접수 | vendor가 합류 신청 제출 시 |
| `brand_approved` | 입점 승인 | admin이 브랜드 입점 상태를 '승인'으로 변경 시 |
| `brand_rejected` | 입점 거절 | admin이 브랜드 입점 상태를 '거절'로 변경 시 |
| `product_registered` | 상품 등록 승인 | admin이 상품 승인 처리 시 |
| `product_rejected` | 상품 등록 거절 | admin이 상품 거절 처리 시 |
| `notice_published` | 공지사항 게시 | admin이 공지사항을 '공개' 상태로 게시 시 |
| `inquiry_answered` | 1:1 문의 답변 완료 | admin이 문의에 답변 작성 시 문의 담당자에게 발송 |

---

## 추가 가능한 트리거 후보 (미구현)

### 📦 재고·상품 관련

| 트리거 ID | 설명 | 발송 시점 | 필요 기능 |
|---|---|---|---|
| `product_stop` | 상품 판매중지 알림 | admin이 상품을 판매중지 처리 시 | 현재 구현 가능 |
| `inventory_low` | 재고 부족 알림 | 재고가 설정된 최소 수량 이하로 떨어질 때 | 재고관리 기능 구현 후 |
| `inventory_out` | 재고 소진 알림 | 재고가 0이 될 때 | 재고관리 기능 구현 후 |
| `inventory_received` | 입고 확인 알림 | 입고 처리 완료 시 vendor에게 발송 | 재고관리 기능 구현 후 |

### 💰 정산 관련

| 트리거 ID | 설명 | 발송 시점 | 필요 기능 |
|---|---|---|---|
| `settlement_ready` | 정산 내역 생성 알림 | 정산 초안 생성 시 vendor에게 발송 | gmbs-functions 정산 기능 구현 후 |
| `settlement_confirmed` | 정산 확정 알림 | admin이 정산을 '확정' 상태로 변경 시 | 정산 기능 구현 후 |
| `settlement_paid` | 정산 지급 완료 알림 | admin이 정산 상태를 '지급완료'로 변경 시 | 정산 기능 구현 후 |

### 📝 계약 관련

| 트리거 ID | 설명 | 발송 시점 | 필요 기능 |
|---|---|---|---|
| `contract_sent` | 전자계약 발송 알림 | 유캔사인 전자계약 발송 시 | 18단계(유캔사인 연동) 구현 후 |
| `contract_completed` | 전자계약 체결 완료 알림 | 서명 완료 시 admin에게 알림 | 18단계 구현 후 |
| `contract_expiring` | 계약 만료 임박 알림 | 계약 종료 30일/7일 전 | gmbs-functions 스케줄러 구현 후 |
| `contract_expired` | 계약 만료 알림 | 계약 종료일 당일 | gmbs-functions 스케줄러 구현 후 |

### 🏪 입점·브랜드 관련

| 트리거 ID | 설명 | 발송 시점 | 필요 기능 |
|---|---|---|---|
| `brand_terminated` | 입점 종료 알림 | admin이 입점 상태를 '종료'로 변경 시 | 현재 구현 가능 |
| `brand_welcome` | 입점 환영 안내 | 브랜드가 처음 승인될 때 (brand_approved와 별도 또는 통합) | 현재 구현 가능 |

### 💬 문의·공지 관련

| 트리거 ID | 설명 | 발송 시점 | 필요 기능 |
|---|---|---|---|
| `inquiry_received_admin` | 새 문의 접수 알림 (관리자용) | vendor가 새 문의를 등록했을 때 admin에게 발송 | 현재 구현 가능 |

---

## 추가 방법

1. `email-event-candidates.md`에서 원하는 트리거 후보 선택
2. Claude에게 "이메일 트리거 추가해줘: `{트리거_ID}` — {설명}" 요청
3. Claude가 `index.html`의 `EMAIL_TRIGGERS` 배열에 항목 추가 후 push

※ vendor 앱(`gmbs-vendor`)의 `emailjs-config.js`는 수정 불필요 — 공통 `sendEmail` 함수가 trigger_event로 동적 처리함.
