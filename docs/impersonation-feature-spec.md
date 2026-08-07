# 벤더 포털 담당자 계정 직접 접속 기능 (Impersonation)

> 저장 일자: 2026-08-07  
> 상태: **보류 중** — 나중에 구현 요청 시 이 문서를 기준으로 작업

---

## 개요

어드민에서 특정 담당자의 Firebase 계정으로 벤더 포털을 직접 열어, 그 담당자가 보는 화면을 그대로 확인하는 기능.  
Firebase Custom Token(`auth.createCustomToken`)을 사용하는 완전한 계정 전환 방식이라 보안 규칙, 권한 필터링, 화면 조건이 모두 그 계정 기준으로 동작한다.

---

## 어드민 쪽 (이미 구현 완료 — commit `aadc4ca`)

- 브랜드 상세 팝업 → 담당자 항목 → **"👁 벤더로 접속"** 버튼
- 클릭 시 확인 다이얼로그 → `generateImpersonationToken` callable 호출 → 반환된 토큰을 URL 파라미터로 담아 `https://vendor.gmbs.kr?impersonate_token=TOKEN` 새 탭으로 열기
- 해당 계정이 벤더 포털에 한 번도 로그인한 적 없으면 "아직 로그인한 적 없습니다" 안내

---

## gmbs-functions 구현 요구사항

### 함수명: `generateImpersonationToken`
- 타입: `onCall` (callable)
- 리전: `asia-northeast3`

**요청 파라미터**
```js
{
  brandId: string,      // 브랜드 ID (확인용)
  loginEmail: string    // 담당자의 벤더 포털 로그인 구글 계정 이메일
}
```

**처리 로직**
```js
// 1. 어드민 계정에서만 호출 가능
const ALLOWED_EMAILS = ['itsbeybusiness@gmail.com', 'baekeun0@gmail.com'];
if (!ALLOWED_EMAILS.includes(context.auth.token.email)) {
  throw new functions.https.HttpsError('permission-denied', '어드민 전용');
}

// 2. 해당 이메일의 Firebase Auth UID 조회
const user = await admin.auth().getUserByEmail(loginEmail);
// → 존재하지 않으면 auth/user-not-found → 어드민 앱이 "아직 로그인한 적 없음"으로 안내

// 3. Custom Token 발급 (1시간 유효)
const customToken = await admin.auth().createCustomToken(user.uid, {
  brandId,
  isImpersonation: true   // 벤더 포털에서 impersonation 세션 구별용
});

// 4. 토큰 반환
return { token: customToken };
```

**보안 포인트**
- `ALLOWED_EMAILS` 체크로 어드민 계정에서만 호출 가능
- Custom Token은 `signInWithCustomToken()`으로만 사용 가능하고 만료됨
- `isImpersonation: true` 클레임으로 벤더 포털에서 "어드민 미리보기 모드" 배너 표시 가능

---

## gmbs-vendor 구현 요구사항

앱 진입 시 URL에 `?impersonate_token=...`이 있으면 해당 계정으로 자동 로그인:

```js
import { signInWithCustomToken } from 'firebase/auth';

const params = new URLSearchParams(window.location.search);
const token = params.get('impersonate_token');
if (token) {
  await signInWithCustomToken(auth, token);
  // URL 파라미터 제거 (히스토리 오염 방지)
  window.history.replaceState({}, '', window.location.pathname);
  // (선택) 상단에 "어드민 미리보기 모드" 배너 표시
  // → auth.currentUser.getIdTokenResult()로 isImpersonation: true 확인 후 표시
}
```

---

## 전제 조건

- 담당자가 벤더 포털에 **한 번 이상 로그인한 적이 있어야** Firebase Auth UID가 존재함
- gmbs-functions가 먼저 구축되어 있어야 함
