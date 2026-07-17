# gmbs-vendor 개발 명세 — 고객 CS (매장 문의)

> admin에서 오프라인 매장 손님의 CS를 등록하면, 해당 브랜드의 담당자가 포털(gmbs-vendor)에서 확인하고 응답할 수 있는 기능입니다.
> 이 문서를 gmbs-vendor 개발에 전달하세요.

---

## 개요

- admin이 손님 CS를 `customer_inquiries` 컬렉션에 등록
- 브랜드 담당자(vendor 포털 로그인 사용자)가 자기 브랜드 문의를 조회 · 응답
- 응답은 `customer_inquiries/{id}/responses` 서브컬렉션에 저장
- 상태 흐름: `답변대기` → `답변완료` → `처리완료` / `재응답 필요` → `답변완료` (반복)

---

## Firestore 스키마

### `customer_inquiries/{inquiryId}`

| 필드 | 타입 | 설명 |
|---|---|---|
| `type` | string | 문의 종류: `상품` \| `재입고` \| `불량/고장` \| `기타` |
| `title` | string | 문의 제목 |
| `content` | string | 문의 내용 |
| `brand_id` | string | 관련 브랜드 ID (없으면 빈 문자열) |
| `brand_name` | string | 브랜드명 (표시용 캐시) |
| `product_id` | string | 관련 상품 ID (없으면 빈 문자열) |
| `product_name` | string | 상품명 (표시용 캐시) |
| `sku_id` | string | 관련 SKU ID (없으면 빈 문자열) |
| `unit_id` | string | 관련 개체 ID (없으면 빈 문자열) |
| `status` | string | `답변대기` \| `답변완료` \| `재응답 필요` \| `처리완료` |
| `re_response_note` | string | 재응답 요청 시 관리자가 입력한 추가 문의/사유 |
| `created_at` | string | 등록일 (YYYY-MM-DD) |
| `created_by` | string | 등록한 관리자 이메일 |
| `updated_at` | string | 최종 수정일 |

### `customer_inquiries/{inquiryId}/responses/{responseId}`

| 필드 | 타입 | 설명 |
|---|---|---|
| `content` | string | 응답 내용 |
| `author_type` | string | `admin` \| `vendor` |
| `author_email` | string | 응답 작성자 이메일 |
| `author_name` | string | 응답 작성자 이름 |
| `created_at` | string | 작성일 (YYYY-MM-DD) |

---

## Firestore 보안 규칙 (추가 필요)

```
// customer_inquiries — admin 전체 쓰기, vendor는 자기 brand_id 문의만 읽기
match /customer_inquiries/{docId} {
  allow read, write: if request.auth != null &&
    request.auth.token.email in ['itsbeybusiness@gmail.com', 'baekeun0@gmail.com'];
  // vendor 읽기: 자기 brandId와 일치하는 문의만
  allow read: if request.auth != null &&
    resource.data.brand_id == request.auth.token.brand_id;
}

match /customer_inquiries/{docId}/responses/{respId} {
  allow read, write: if request.auth != null &&
    request.auth.token.email in ['itsbeybusiness@gmail.com', 'baekeun0@gmail.com'];
  // vendor 읽기/쓰기: 부모 문의가 자기 브랜드인 경우
  allow read, write: if request.auth != null &&
    get(/databases/$(database)/documents/customer_inquiries/$(docId)).data.brand_id
      == request.auth.token.brand_id;
}
```

> ⚠️ 주의: `request.auth.token.brand_id`는 vendor 로그인 시 custom claim으로 발급되어야 합니다. gmbs-functions에서 로그인 시 해당 vendor의 brand_id를 custom claim으로 설정하는 로직이 필요합니다.

---

## vendor 포털 구현 명세

### 1. 사이드바 메뉴 추가

```
고객 지원
  └─ 고객 문의       (신규 추가)
```

배지: `답변대기` + `재응답 필요` 건수 합계

---

### 2. 고객 문의 목록 화면

**URL/라우트:** `/customer-inquiries` (또는 기존 라우팅 방식에 맞게)

**데이터 조회:**
```javascript
// 자기 브랜드의 문의만 조회 (status가 처리완료가 아닌 것 우선)
const snap = await getDocs(query(
  collection(db, 'customer_inquiries'),
  where('brand_id', '==', currentBrandId)
));
```

**목록 표시 컬럼:**
- 상태 배지 (답변대기=노랑, 답변완료=파랑, 재응답 필요=빨강, 처리완료=초록)
- 종류 (상품/재입고/불량고장/기타)
- 제목
- 관련 상품명
- 등록일

**필터:** 전체 / 답변대기 / 재응답 필요 / 답변완료 / 처리완료

**정렬:** 최신순

---

### 3. 고객 문의 상세 화면

클릭 시 상세 표시:

**기본 정보 섹션:**
- 상태 배지
- 종류, 등록일
- 브랜드명 (표시 생략 가능 — vendor는 자기 브랜드만 봄)
- 관련 상품명 / SKU
- 제목 (크게)
- 내용 (전체)

**재응답 요청 알림 박스 (status === '재응답 필요' 일 때):**
```
⚠️ 관리자가 추가 응답을 요청했습니다.
[re_response_note 내용]
```
노란색 배경 강조 박스로 표시

**응답 이력 섹션:**
- 각 응답마다: 작성자 이름/유형 + 작성 시각 + 내용
- admin 응답: 회색 배경
- vendor 응답: 브랜드 컬러(파랑 계열) 배경

**응답 작성 폼:**
- textarea (응답 내용)
- "응답 저장" 버튼

**응답 저장 로직:**
```javascript
// 1. responses 서브컬렉션에 응답 추가
await addDoc(collection(db, 'customer_inquiries', inquiryId, 'responses'), {
  content: responseText,
  author_type: 'vendor',
  author_email: currentUser.email,
  author_name: currentUserName,  // vendor 포털의 로그인된 담당자 이름
  created_at: today(),  // YYYY-MM-DD
});

// 2. 문의 상태를 '답변완료'로 업데이트
await updateDoc(doc(db, 'customer_inquiries', inquiryId), {
  status: '답변완료',
  updated_at: today(),
});
```

---

### 4. 상태 표시 규칙

| status | vendor 화면 표시 | 응답 폼 표시 여부 |
|---|---|---|
| `답변대기` | 🟡 답변 필요 | ✅ 표시 |
| `재응답 필요` | 🔴 재응답 요청됨 | ✅ 표시 (노란 박스도 함께) |
| `답변완료` | 🔵 답변 완료 | ✅ 표시 (추가 응답 가능) |
| `처리완료` | 🟢 처리 완료 | ❌ 숨김 (관리자가 종결 처리함) |

---

### 5. 사이드바 배지 업데이트 로직

```javascript
const snap = await getDocs(query(
  collection(db, 'customer_inquiries'),
  where('brand_id', '==', currentBrandId)
));
const urgent = snap.docs.filter(d =>
  d.data().status === '답변대기' || d.data().status === '재응답 필요'
).length;
// 배지에 urgent 숫자 표시
```

---

### 6. 알림 (선택 구현 — 우선순위 낮음)

새 문의 등록 시 또는 재응답 요청 시, 해당 브랜드 담당자에게 이메일 알림을 보낼 수 있습니다.
admin의 이메일 설정 페이지에 트리거 추가 예정:
- `customer_inquiry_received` — 고객 CS 문의 등록 시 브랜드 담당자에게 발송
- `customer_inquiry_re_requested` — 재응답 요청 시 브랜드 담당자에게 발송

이 트리거는 admin 기능 완성 후 순서에 따라 추가합니다.

---

## 구현 순서 (권장)

1. Firestore 보안 규칙 업데이트
2. 사이드바에 "고객 문의" 메뉴 추가 + 배지
3. 목록 화면 구현
4. 상세 화면 + 응답 폼 구현
5. 테스트: admin에서 문의 등록 → vendor에서 조회/응답 → admin에서 응답 확인 및 처리

---

*작성일: 2026-07-17*
*admin 개발자: gmbs-admin 프로젝트 (beybusiness-bit/gmbs-admin)*
