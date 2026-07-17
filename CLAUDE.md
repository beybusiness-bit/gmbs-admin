## GMBS 상품 등록·바코드 관리 웹앱 — CLAUDE.md

### 앱 기본 정보

```javascript
const AUTH = {
  ALLOWED_EMAILS: ['itsbeybusiness@gmail.com', 'baekeun0@gmail.com'],
  FIREBASE_CONFIG: {
    apiKey: 'TBD',
    authDomain: 'TBD',
    projectId: 'TBD',
    storageBucket: 'TBD',
    messagingSenderId: 'TBD',
    appId: 'TBD',
  },
};
const REPO = {
  GITHUB_URL: 'https://github.com/beybusiness-bit/gmbs-admin',
  LOCAL_PATH: '~/projects/gmbs-admin',
  DEPLOY_METHOD: 'vercel',
  LIVE_URL: 'TBD', // Vercel 대시보드에서 확인
  // 브랜치 push → Vercel 미리보기 URL 자동 생성 / main 머지 → 프로덕션 배포
};
```

### 앱 아키텍처 요약
- **앱 성격**: GMBS 소품샵의 일반상품·PB상품·위탁상품·수공예품을 하나의 체계로 등록하고 Code128 바코드를 자동 생성하는 내부 관리자 전용 도구
- **UI 구조**: 사이드바 메뉴 5개 — 대시보드 / 상품 관리(상품 목록·새 상품 등록·상품 수정·개체 관리) / 바코드(검색·선택 출력·최근 생성·라벨 설정) / 브랜드 관리(목록·새 브랜드 등록) / 설정
- **로그인**: 필수 (Google OAuth, ALLOWED_EMAILS 2개 계정만 허용, 그 외 계정은 로그인 즉시 로그아웃 처리)
- **사용자 역할**: 단일 (허가된 두 계정 모두 동일 권한의 관리자, 역할 구분 없음)
- **외부 연동**: 없음
- **PWA**: 적용 (홈 화면 설치, 오프라인 기본 캐싱)
- **FCM 알림**: 미적용 (1차 버전 제외. 향후 재고 관련 알림 등이 필요해지면 추가 검토)
- **기존 도구 마이그레이션**: 없음 — 처음부터 신규 구축

---

### ⚠️ 비전문가 사용자 안내 원칙

이 앱의 주 사용자는 개발·코딩 배경이 없는 비전문가다. Claude Code는 아래 원칙을 항상 지킨다:

1. **모든 작업에 자세한 설명 동반**: 코드를 수정했으면 "무엇을 왜 바꿨는지"를 평이한 말로 함께 설명한다. 전문 용어는 괄호 안에 간단한 풀이를 덧붙인다.
2. **단계별 안내**: 사용자가 직접 해야 할 일(파일 복사, 설정 입력 등)은 번호를 매긴 단계로 안내한다.
3. **오류 발생 시**: 에러 메시지를 그대로 던지지 말고 "무슨 문제인지, 어떻게 해결하면 되는지"를 풀어서 설명한다.
4. **확인 요청**: 사용자가 직접 조작해야 하는 단계가 있으면, 완료 여부를 확인 후 다음으로 넘어간다.

---

### 🔁 세션 과부하 감지 및 전환 권유

아래 상황 중 하나라도 해당되면 사용자에게 **세션 변경을 먼저 권유**한다:

- 현재 세션에서 주고받은 메시지가 많아져 맥락을 정확히 추적하기 어려울 때
- 같은 오류가 3회 이상 반복되어 해결이 안 될 때
- 여러 기능을 동시에 수정하다가 흐름이 얽혔을 때
- 세션 응답이 느려지거나 이전 내용을 잘못 참조하는 패턴이 보일 때

권유 멘트 예시:
> "지금 세션이 꽤 길어져서 맥락이 뒤섞일 수 있어요. 세션을 새로 시작하는 게 더 빠르고 정확할 것 같습니다.
> 아래 내용을 복사해서 새 세션에 붙여넣으면 바로 이어서 작업할 수 있어요:
> ---
> [다음 세션 시작 프롬프트 출력]
> ---"

---

### 세션 운영 원칙

- **기본 단위**: 개발 단계 하나 = Claude Code 세션 하나
- **세션 전환 기준**: 코드 300줄 초과로 수정이 복잡해질 때 / 새로운 기능 영역 진입 시 / 오류 반복으로 맥락이 꼬였을 때 → `/exit` 후 같은 폴더에서 `claude` 재실행
- **CLAUDE.md 갱신**: 매 세션 마무리 시 Claude Code가 이 파일을 직접 수정한다. 별도 복붙 불필요.

---

### 🟢 세션 시작 시 자동 수행

첫 메시지를 받으면 사용자 요청 처리 전에 **자동으로** 아래를 수행한다.

#### Step 1. 실행 환경 판별 → 적절한 분기

```
현재 위치가 /home/user 같은 임시 클라우드 경로인가? (Remote 세션 특징: 매번 깨끗한 VM에서 시작, 레포는 이미 clone된 상태)
  ├─ 예 (Remote 세션) → Step 1-R
  └─ 아니오 (Local 세션: 내 컴퓨터) → Step 1-L
```

#### Step 1-R. Remote 세션 (데스크톱 앱의 ☁️ 환경) — 이 프로젝트의 주 사용 환경

- VM이 자동으로 최신 GitHub 상태를 clone한 직후이므로 pull 불필요
- **🔑 PAT 확인 (필수):**
  ```bash
  git remote -v
  ```
  출력된 origin URL에 `ghp_` 또는 `github_pat_`로 시작하는 토큰이 포함되어 있으면 OK.
  토큰이 없다면 → 아래 "🔑 PAT 설정 프로토콜" 섹션을 먼저 수행한다.
  이 설정을 건너뛰면 세션 종료 시 push가 403으로 실패해서 작업이 전부 증발한다.
- PAT 확인(또는 설정)이 끝나면 Step 2(현황 요약)로 이동
- ⚠️ 이 세션 안에서 생긴 변경은 **push 전까지 VM 안에만 존재**. 세션 종료 시 VM이 사라지면 증발. 중간 커밋과 종료 시 push를 엄격히 지킨다.

#### Step 1-L. Local 세션 (내 컴퓨터의 터미널 또는 앱 Local 환경)

```
LOCAL_PATH 폴더가 존재하는가?
  ├─ 없음 (이 기기에서 처음) → 클론
  │     cd ~/projects
  │     git clone https://github.com/beybusiness-bit/gmbs-admin
  │     cd gmbs-admin
  │     → "이 기기에 처음 세팅했습니다. 클론 완료." 안내
  │
  └─ 있음 (기존 폴더) → pull
        cd ~/projects/gmbs-admin
        git pull origin main
        충돌 발생 시: 사용자가 직접 마커(<<<<<<) 손대지 않게 안내하고
        Claude가 직접 분석·해결. 원인 1순위(다른 기기·다른 세션에서 push 안 한 작업) 먼저 의심.
```

#### Step 2. 현황 요약 보고

```
📋 현재 상황 요약
- 환경: [Remote ☁️ / Local 💻]
- 완료: [완료된 단계 목록 ✅]
- 진행중: [현재 단계 🔄] — [어디까지 됐는지]
- 남은 것: [예정 단계 목록 🔲]
- 이번 세션 시작점: [다음 할 작업]
```

단, 세션이 이미 진행 중이고 사용자가 단순 작업 요청만 하는 경우엔 매번 pull·보고 반복하지 않음.

#### Step 3. 배포 방식: Vercel

**이 저장소 전용 세션 (gmbs-admin 직접 세션):** 별도 브랜치 없이 main에서 직접 작업한다. 수정 → 커밋 → push → Vercel 프로덕션 자동 배포. PR 과정 불필요. push 배포 횟수 제한 없음.

**다른 저장소 세션에서 gmbs-admin을 add_repo로 추가해 작업하는 경우 (예: gmbs-vendor 세션):** main 직접 push 금지. 반드시 `claude/기능명-랜덤` 형태의 별도 브랜치에서 작업하고 PR을 올린다. 이유: 동시에 진행 중인 gmbs-admin 직접 세션과 충돌 방지.

---

### 📌 세션 시작 방법 (사용자 참고용)

**방법 A. Claude 데스크톱 앱 Code 탭 — Remote(☁️) 환경 (권장, 이 프로젝트의 주 사용 방식)** ⭐
1. 앱 좌측 사이드바에서 Code 탭 열기
2. `+ 새 세션` 클릭
3. 환경 드롭다운에서 **Remote(☁️)** 선택
4. `+ 레포 선택` 클릭 → `beybusiness-bit/gmbs-admin` 저장소 선택
5. 작업 내용 입력하고 시작

장점: 로컬에 폴더 세팅이 없어도 됨. 어느 컴퓨터·어느 기기에서든 로그인만 돼 있으면 동일하게 시작.
주의: 작업 내용이 VM 안에만 있으므로 **반드시 커밋·push하고 종료해야 함**.

**방법 B. Claude 데스크톱 앱 Code 탭 — Local(💻) 환경**
1. Code 탭 → `+ 새 세션`
2. 환경: **Local(💻)** 선택
3. 프로젝트 폴더 선택 (`~/projects/gmbs-admin`) — 없으면 터미널에서 `cd ~/projects && git clone https://github.com/beybusiness-bit/gmbs-admin` 먼저 실행

**방법 C. 터미널 (전통 방식)**
```bash
cd ~/projects/gmbs-admin
claude
```

---

### 🔴 세션 종료 시 자동 수행

사용자가 "끝났어" / "마무리할게" / "다른 기기 갈게" / "세션 끝내자" 등을 말하면:

1. `git status` — 변경된 파일 목록 확인
2. 변경 목록 + 제안 커밋 메시지를 사용자에게 보여주고 승인 받기
3. 승인 후:
   ```bash
   git add [변경 파일 명시] && git commit -m "..." && git push origin main
   ```
   - `git add .` / `git add -A` 금지 — 변경 파일을 명시적으로 지정
   - push 성공 여부 반드시 확인. "아마 됐을 거예요" 식으로 얼버무리지 말 것
   - push 완료 후: "✅ 푸시 완료. Vercel 배포까지 ~1분. Cmd+Shift+R 하드 리프레시 해주세요."
   - **403 오류가 나면** → "🔑 PAT 설정 프로토콜" 섹션으로 이동
4. CLAUDE.md 직접 갱신:
   - 완료된 단계 ✅ 표시
   - 다음 세션 시작점 업데이트
   - DB 구조 변경 사항 반영 (있을 경우)
   - **기획·구현 내용 누적 보존**: 이번 세션에서 사용자와 합의한 기획 내용, 구현 결정 사항을 "개발 기획 내용 누적" 섹션에 추가. 기존 내용은 절대 삭제·축약하지 않음. 변경이 생긴 경우 해당 항목 옆에 변경 이유를 주석으로 남김.
   - 가이드 문서 참고 내용 누적에 이번 세션 내용 추가
5. CLAUDE.md 갱신분도 함께 커밋·push (별도 커밋 권장: `docs: CLAUDE.md 진행 상황 갱신`)
6. **🧪 테스트 체크리스트 출력** — 이번 세션에서 변경한 기능별로 사용자가 직접 확인할 항목을 번호 목록으로 출력한다. 절대 빠뜨리지 않는다.
   - 새로 추가한 기능: 핵심 동작 + 엣지 케이스 (비로그인, 빈 데이터 등)
   - 수정한 버그: 버그가 실제로 고쳐졌는지 재현 방법
7. **GitHub PAT 출력**: `git remote -v` 출력에서 `ghp_...` 토큰 부분을 그대로 복사해서 아래처럼 안내한다:
   > "🔑 현재 PAT: `ghp_XXXXXXXXXXXXXXXX`
   >  다음 세션 시작 시 이 토큰이 없으면 붙여넣으세요."
   (PAT가 설정되어 있지 않은 경우에는 이 단계를 건너뜀)
8. 다음 세션 시작 프롬프트 출력:

```
환경 확인: Remote(☁️)인지 Local(💻)인지 먼저 판별해줘.
배포 방식: vercel (브랜치 push → 미리보기 URL / main 머지 → 프로덕션)

PAT: [현재 PAT, 없으면 생략]

이번 세션 작업: [N단계] [기능명] — [어디서부터 시작할지 구체적으로]
이전 세션에서 [완료 내용]까지 완성했고, 이번엔 [다음 작업]을 구현하면 돼.
```

---

### 🔑 PAT(Personal Access Token) 설정 프로토콜

**왜 필요한가:**
Claude Code Remote(☁️) 환경에서는 로컬 프록시가 git 요청을 중계하는데, 이 프록시는 **쓰기 권한이 없어 push를 차단**한다. 그래서 토큰 없이 `git push`를 시도하면 항상 403 오류가 난다.

**해결책 — `.git/config`의 remote URL에 PAT 직접 포함:**
작업 디렉토리(`.git/config`)는 세션 간 유지되므로, 여기에 토큰을 박아 두면 이후 세션에서도 push가 된다.

**실패 시점(push 403) 또는 세션 시작 시점(Step 1-R)에 PAT이 설정돼 있지 않으면 아래를 수행한다:**

1. 사용자에게 토큰 요청:
   > "Remote 환경에서 GitHub push를 하려면 Personal Access Token이 필요합니다.
   >  GitHub → Settings → Developer settings → Personal access tokens (classic)
   >  → Generate new token → Scope: **`repo` 하나만** 체크 → 토큰 생성
   >  → `ghp_…`로 시작하는 토큰을 붙여넣어 주세요."

2. 토큰을 받으면 remote URL 재설정:
   ```bash
   git remote set-url origin https://ghp_TOKEN@github.com/beybusiness-bit/gmbs-admin.git
   ```

3. 동작 확인:
   ```bash
   git remote -v
   git push -u origin main
   ```

4. 성공 시 안내:
   > "✅ PAT 설정 완료. 이후 모든 세션에서 별도 설정 없이 push 가능합니다."

**⚠️ PAT 관련 보안 수칙:**
- 토큰은 **절대 CLAUDE.md, 코드, 커밋 메시지**에 남기지 않는다.
- `.git/config`에 들어가는 것은 허용(해당 파일은 `.git/` 안에 있어 .gitignore로 제외됨).
- 세션 종료 시 PAT 출력은 **사용자 편의용**으로만 사용. 코드나 커밋에는 절대 포함 금지.

---

### 🔵 수정 후 배포

수정 요청 → 코드 수정 → 문법 검증 통과 → 자동으로 commit + push → 아래 안내 출력:

```
✅ 푸시 완료. Vercel 배포까지 ~1분 소요.
브라우저에서 Ctrl+Shift+R (Mac: Cmd+Shift+R) 하드 리프레시 해주세요.
```

- commit 메시지 형식: `fix: [수정 내용 한 줄 요약]`
- `git add .` 금지 — 수정된 파일 명시적으로 지정
- 문법 검증 실패 시 push 중단, 오류 먼저 해결

**push 보류 판단 기준 — 아래 상황이면 Claude가 먼저 제안:**
- 미완성 상태 (UI 절반만 수정, 기능 흐름 끊김 등)
- 인증·데이터 쓰기 등 검증 안 된 큰 변경
- 사용자가 "일단 해봐", "테스트해볼게" 등 탐색적 요청을 한 경우

---

### 🟡 작업 중간에 기기 이동 또는 환경 전환할 때

사용자가 "덜 끝났는데 다른 기기 가야 돼", "세션 잠깐 끊고 다른 데서 이어서" 등을 말하면:

1. 현재 변경사항 확인 → WIP 커밋 생성 후 push: `git commit -am "WIP: [간단 설명]"` → `git push origin main`
2. "⚠️ WIP 상태 push 완료. 다음 세션에서 이어서 작업 가능합니다" 안내

---

### Phase 2: 개발 진행 프로토콜

#### 코드 작업 원칙

```javascript
// ① today() — 반드시 로컬 날짜 기준 (toISOString() UTC 방식 금지)
const today = () => {
  const d = new Date();
  return d.getFullYear() + '-'
    + String(d.getMonth() + 1).padStart(2, '0') + '-'
    + String(d.getDate()).padStart(2, '0');
};
// ② AUTH 상수 — 매 작업마다 위 실제값 유지
// ③ 모든 번호 필드(product_number, option_number, unit_number)는 반드시 문자열로 저장 (앞자리 0 유지)
```

**절대 금지:** script 안 백틱 중첩 / script 안 `&` `<` `>` `"` 직접 삽입 / `innerHTML` null 체크 생략 / `toISOString()` UTC 날짜

**매 작업 후 필수:** 문법 오류 확인 → 파일 저장

**수정 방식:** 300줄 미만 전체 재작성 / 이상은 부분 수정

**코드 패턴 참고** (필요 시 claude.ai에서 로드):
- CSS·레이아웃·컴포넌트: `webapp-builder` 스킬 `references/app-structure.md`
- Firebase 헬퍼: `webapp-builder` 스킬 `references/firebase-integration.md`
- PWA 설정: `webapp-builder` 스킬 `references/pwa-setup.md`

**이 프로젝트 전용 라이브러리 (CDN):**
- 바코드 생성: JsBarcode (Code128, 브라우저에서 SVG/Canvas로 직접 생성 — 서버 불필요)
- PDF 생성(라벨 인쇄용): jsPDF
- ZIP 일괄 다운로드: JSZip
- 엑셀 내보내기(설정 > 데이터 관리): SheetJS(xlsx)

#### 단계별 진행 원칙
- 각 단계 완료 후 사용자 확인 받고 다음 단계 시작
- 단계 완료 또는 세션 마무리 시 → 위 "🔴 세션 종료 시 자동 수행" 절차 따름

#### 기획 내용 유실 방지 원칙
- CLAUDE.md를 갱신할 때 **기존 기획 내용, 구현 결정 사항은 절대 삭제·축약·생략하지 않는다.**
- 세션이 여러 번 바뀌어 CLAUDE.md 갱신이 반복되어도, Phase 1에서 합의한 모든 단계·기능·요구사항은 원문 그대로 유지된다.
- 사용자가 의사를 바꿔 기획 내용이 변경된 경우에는, 변경된 항목 옆에 `[변경: 이유 요약]` 형태로 주석을 달고 이전 내용도 취소선 등으로 남긴다.
- "간략화", "요약", "정리" 등을 이유로 기존 기획 내용을 줄이지 않는다.

---

### Phase 3: 배포 프로토콜

1. `index.html` 및 `service-worker.js`(PWA) GitHub 저장소 업로드
2. Settings → Pages → Branch: main → Save
3. Firebase 콘솔 → Authentication → 승인된 도메인에 Vercel 배포 URL 추가
4. Firestore 보안 규칙 확인 (ALLOWED_EMAILS에 해당하는 로그인 사용자만 읽기/쓰기 허용 — 아래 "DB 구조" 참고)
5. manifest.json, 아이콘 파일들 배포 확인 (PWA)
6. Google Cloud OAuth 동의 화면에 테스트 사용자로 itsbeybusiness@gmail.com, baekeun0@gmail.com 등록 확인

Firebase 첫 설정이라면: `webapp-builder` 스킬 `references/firebase-setup.md` 참고

---

### Phase 4: 가이드 문서 작성 프로토콜

배포 완료 후 요청 시, 아래 프롬프트를 완성해서 출력한다.
(이 시점에서 "가이드 문서 참고 내용 누적" 항목의 내용을 프롬프트 안에 포함시킨다)

```
지금까지 이 프로젝트에서 개발한 앱의 사용 가이드를 노션에 작성해줘.
노션 MCP로 바로 작성. 작성할 노션 페이지 URL: [URL]

앱 이름: GMBS 상품 등록·바코드 관리
접속 URL: [Vercel 배포 URL]
로그인: Google OAuth (허가된 이메일만)
허가 이메일: itsbeybusiness@gmail.com, baekeun0@gmail.com
데이터 저장: Firebase Firestore (Project: [projectId])

주요 기능 (사이드바 순서): [구현된 메뉴 전체]
주요 워크플로우: [반복 업무 흐름]

개발 과정에서 수집된 가이드 참고 내용:
[아래 "가이드 문서 참고 내용 누적" 항목 전체 삽입]

문서 요건:
- 대상: 기술 배경 없는 처음 사용자
- 구성: 시작하기 → 메뉴별 기능 → 주요 워크플로우 → 주의사항·FAQ
- 각 섹션에 "📸 이미지 추가 위치: [캡처할 화면]" 안내 포함
- 관리자용 섹션 포함
```

---

### 개발 단계 현황

1단계 프로젝트 셋업(Firebase 연결, Google 로그인, 사이드바 레이아웃, PWA manifest) — ✅ 완료
2단계 브랜드 관리(등록/수정/사용중지, 브랜드코드 중복확인) — ✅ 완료
3단계 상품 등록 ①(기본정보+옵션 설정, 상품번호 자동 미리보기, 옵션 최대 3축, 조합 자동생성, 순서 드래그) — ✅ 완료
4단계 상품 등록 ②(트랜잭션 기반 번호 발급·중복검증 저장, 개체관리 상품 개체 생성, 최종 확인 화면) — ✅ 완료
5단계 상품 목록·검색·수정(상품 단위 묶음 표시, 브랜드/상품명/바코드 검색, 판매중지 처리) — ✅ 완료
6단계 개체 관리(SKU별 개체 목록, 상태 변경: 판매중/판매완료/분실/폐기) — ✅ 완료
7단계 바코드 생성·출력(Code128, 라벨 미리보기, 동일SKU 반복/개체 연속 출력, PNG/ZIP 다운로드, 브라우저 인쇄·PDF) — ✅ 완료
8단계 설정 페이지(번호 자릿수 규칙 및 변경 경고, 라벨 크기, 데이터 관리 엑셀 내보내기, Firebase 연결상태) — ✅ 완료 (가격구조·번호자릿수·라벨크기 설정 구현. 엑셀 내보내기·Firebase 연결상태 표시는 미구현)
9단계 PWA 마무리(아이콘, service worker, 홈 화면 설치 안내) — ✅ 완료 (service worker 구현. 전용 아이콘·홈화면 설치 안내 배너는 미구현)
추가 브랜드 정보 확장(사업자구분/과세정보/주민등록번호/연락처/SNS/소개/로고 사진) — ✅ 완료
추가 상품·SKU·개체 사진 기능(등록·수정·목록 썸네일·바코드 페이지 썸네일·라벨 미리보기) — ✅ 완료
추가 UI 개선(브랜드 상세 팝업, 안내문구 스타일 통일, 썸네일 2배 크기) — ✅ 완료
10단계 담당자(Person) 관리 — 브랜드별 복수 담당자 등록/수정/삭제(상태값 처리), vendor와 공유되는 필드 정의 — ✅ 완료 (전화번호 자동 하이픈 포맷, 브랜드 상세 팝업 내 섹션으로 구현)
11단계 계약(Contract) 관리 — 계약 정보 입력, 계약 PDF 업로드(Firebase Storage), 계약 버전·상태 관리, 다운로드 링크 제공 — ✅ 완료
12단계 입점 심사 워크플로우 — 브랜드 입점 상태(심사중/승인/거절/종료) 관리, 상태 변경 시 사유 기록 — ✅ 완료 (브랜드 목록 필터 + 상태 변경 모달 + activity_log 기록)
13단계 상품 승인 워크플로우 — 상품 등록/수정 요청 대기열, 승인/거절 처리, 공급가격·판매수수료율 입력, 정가 자동 산정 로직 — ✅ 완료 (retail_price_auto = ceil(supply/(1-rate/100)/100)*100)
14단계 정산(Settlement) 스키마 및 관리 화면 — 정산 데이터 모델 확정, 지급 상태 관리, 세무/증빙 상태 관리 (실제 자동 생성은 gmbs-functions 완성 후 연결) — ✅ 완료
15단계 Activity Log 화면 — 브랜드별 변경 이력 조회 화면 (자동 기록 로직은 1차로 admin 클라이언트에서 직접 기록, 추후 gmbs-functions의 Firestore 트리거로 이관 예정) — ✅ 완료 (writeActivityLog() + loadBrandActivity() 구현, 브랜드 상세 팝업 내 이력 섹션)
~~16단계 공지사항(Notice) 관리~~ → **[변경: vendor 온보딩·FAQ와 통합 관리를 위해 "안내 관리" 메뉴로 확장]**
16단계 안내 관리 메뉴 — 공지사항(작성/수정/게시상태), 안내 페이지 작성(가입 전 안내·가입 후 안내 카드 CRUD+순서변경+이미지업로드), 자주하는 질문 정리(카테고리별 FAQ CRUD+순서변경) — ✅ 완료
17단계 문의(Inquiry) 관리 — vendor가 등록한 문의 목록 조회 및 답변 작성 — ✅ 완료 (사이드바 메뉴 추가, 목록/상세/답변 작성 기능)
추가 이메일 설정 페이지(EmailJS 연동 설정, 트리거별 템플릿 연결·활성화 관리) — ✅ 완료 (vendor.gmbs.kr과 공유 Firebase 프로젝트의 email_configs 컬렉션 사용)
추가 UI/UX 개선 — ✅ 완료 (탭 타이틀 "GMBS 관리자", 키컬러 #8c52ff/#9dff20, 사이드바 햄버거 버튼, 대시보드 처리할 업무 0건 표시, 문의 담당자 표시+타임스탬프 정상화, inquiry_answered 이메일 트리거 추가)
18단계 전자계약 발송·상태 표시(유캔사인) — 브랜드 상세 화면에 "전자계약 발송" 버튼 추가(gmbs-functions 콜러블 호출), 계약 상태 배지 실시간 표시 — 🔲 진행 예정

---

### 이메일 트리거 현황 및 추가 예정 목록

**현재 구현된 트리거** (email_configs 컬렉션에서 관리):
- `application_received` — 신규 브랜드 등록 신청 접수 시
- `join_received` — 기존 브랜드 합류 신청 접수 시
- `brand_approved` — 입점 승인 시
- `brand_rejected` — 입점 거절 시
- `product_registered` — 상품 등록 승인 시
- `product_rejected` — 상품 등록 거절 시
- `notice_published` — 공지사항 게시 시
- `inquiry_answered` — 1:1 문의 답변 완료 시 (문의자 담당자에게 발송)

**향후 기능 확장 시 추가 예정인 트리거** (기능 구현 후 이메일 설정 페이지에 함께 추가):
- `inventory_low` — 재고 부족 알림 (재고관리 기능 구현 후)
- `inventory_out` — 재고 소진 알림 (재고관리 기능 구현 후)
- `settlement_ready` — 정산 내역 생성 알림 (정산 기능 구현 후)
- `settlement_paid` — 정산 지급 완료 알림 (정산 기능 구현 후)
- `contract_expiring` — 계약 만료 임박 알림 (계약관리 기능 구현 후)
- `product_stop` — 상품 판매중지 알림 (추후 필요 시)

⚠️ **새 트리거 추가 방법**: admin/index.html의 `EMAIL_TRIGGERS` 배열에 항목 추가 + vendor/emailjs-config.js는 수정 불필요 (공통 sendEmail 함수가 trigger_event로 동적 처리).

---

### DB 구조 (Firestore 컬렉션 구성)

상품(Product) → SKU → 개체(Unit)의 3단계 구조를 Firestore 서브컬렉션으로 그대로 표현한다.

```
brands/{brandId}
  brand_name, brand_code, brand_type(PB|위탁|매입 등), active,
  last_product_number,   // 이 브랜드에서 마지막으로 발급한 상품번호 (트랜잭션으로 다음 번호 계산)
  created_at, updated_at

products/{productId}     // productId 예: "ONG-014"
  brand_id, brand_code, product_number, product_name, base_price,
  tracking_type(quantity|individual),   // quantity=수량관리(일반상품), individual=개체관리(수공예품 등)
  status(판매중|판매중지), memo,
  last_option_number,    // 이 상품에서 마지막으로 발급한 옵션번호
  created_at, updated_at

  products/{productId}/skus/{skuId}    // skuId 예: "ONG-014-01"
    option_number,          // 바코드에 들어가는 영구 식별번호 (2자리, "00"=옵션없음, "01~99"=옵션조합)
    option_sort_order,      // 화면 표시 순서 (옵션번호와 분리, 자유롭게 변경 가능)
    option_1_name, option_1_value,
    option_2_name, option_2_value,
    option_3_name, option_3_value,
    price, barcode_value,   // "브랜드코드-상품번호-옵션번호" (예: ONG-014-01)
    sku_status(판매중|판매중지|품절),
    barcode_printed_at,
    last_unit_number,       // 이 SKU에서 마지막으로 발급한 개체번호
    created_at, updated_at

    products/{productId}/skus/{skuId}/units/{unitId}   // tracking_type=individual 인 SKU만 사용
      unit_number,            // 3자리, SKU 내에서 순차 발급 ("001~999")
      barcode_value,          // "브랜드코드-상품번호-옵션번호-개체번호" (예: DRM-027-01-003)
      unit_price,             // 비워두면 SKU 기본가(price) 사용
      unit_status(판매중|판매완료|분실|폐기),
      memo, barcode_printed_at,
      created_at, updated_at

settings/config
  product_number_digits: 3, option_number_digits: 2, unit_number_digits: 3,
  no_option_code: "00", barcode_format: "CODE128",
  default_label_width, default_label_height
  // 자릿수 변경 시 기존 번호·바코드와 형식이 달라지므로 반드시 관리자 경고 표시

brands/{brandId}
  // 기존 필드에 추가:
  onboarding_status(심사중|승인|거절|종료), onboarding_status_reason, onboarding_status_changed_at

  brands/{brandId}/persons/{personId}
    name, phone, email, role, active, created_at, updated_at
    // admin·vendor 양쪽 다 쓰기 가능 (vendor는 자기 brandId 범위만, 보안 규칙으로 제한)

  brands/{brandId}/contracts/{contractId}
    version, signed_at, end_at, auto_renew(bool), status(진행중|만료|해지),
    file_url(Storage 경로), uploaded_at, uploaded_by
    // admin만 쓰기 가능. vendor는 읽기 전용 (다운로드만)

products/{productId}
  // 기존 필드에 추가:
  approval_status(승인대기|승인|거절), rejection_reason,
  supply_price(공급가격), commission_rate(수수료율),
  retail_price_auto(정가 자동산정 값), pending_changes(수정요청 시 변경 예정 값 draft, 승인 전까지 실제 필드에 반영 안 함),
  requested_by(vendor uid, 신규 등록/수정 요청한 주체 기록)
  // 신규 등록·수정요청은 vendor가 씀(approval_status=승인대기 상태로), 승인/거절과 supply_price·commission_rate는 admin만 씀

settlements/{settlementId}
  brand_id, period(YYYY-MM), sales_ids(정산 대상 판매건 참조 배열),
  supply_total, commission_total, settlement_amount, pay_date,
  pay_status(예정|검토중|확정|지급완료),
  biz_type(비사업자|간이|일반), tax_type,
  receipt_type(세금계산서|지출증빙현금영수증|원천징수), receipt_status(미처리|완료), receipt_processed_at,
  withholding_status, withholding_processed_at,
  created_at, updated_at
  // 초안 생성은 gmbs-functions(추후), 검토·확정·증빙 처리는 admin, 열람은 vendor도 가능(읽기 전용)

activity_log/{logId}
  brand_id, entity_type(contract|product|price|sales|settlement|admin_action),
  entity_id, action(created|updated|status_changed|deleted),
  actor_email, actor_role(admin|vendor|system),
  before_value, after_value, created_at
  // admin 전용 열람. 1차는 admin 클라이언트 코드에서 주요 동작 시 직접 기록,
  // 추후 gmbs-functions의 Firestore onWrite 트리거로 자동화 예정 (그때는 이 컬렉션 스키마 그대로 재사용)

notices/{noticeId}
  title, content, pinned(bool), status(공개|비공개), created_by, created_at, updated_at
  // admin만 씀, vendor는 읽기 전용

onboarding_cards/{cardId}              // 신규 — vendor 안내 위저드용
  audience('public'|'member'),         // public=가입 전 안내, member=가입 후 안내
  order,                               // audience 그룹 내 정렬 순서
  title, body,
  image_url, image_alt,               // Firebase Storage 업로드 (권장 1200×675px, 최대 2MB)
  cta_type(none|apply|faq_link|continue),
  active, created_at, updated_at
  // admin만 write. vendor(gmbs-vendor)에서 read (로그인 여부 무관)

faq_items/{faqId}                      // 신규 — vendor FAQ 화면용
  category(입점신청|상품등록입고|진열판매|정산세무|교환환불회수|시스템이용|정책계약),
  question, answer, order, active, created_at, updated_at
  // admin만 write. vendor에서 read (로그인 여부 무관)

inquiries/{inquiryId}
  brand_id, person_id, title, content, status(답변대기|답변완료),
  answer, answered_by, answered_at, created_at
  // vendor가 title/content 작성, admin이 answer 작성. 양쪽 다 자기 관련 문서만 읽기 가능(보안 규칙)

brands/{brandId}/contracts/{contractId}
  // 기존 필드에 추가 (유캔사인 전자계약 연동):
  ucansign_document_id,
  status('발송대기'|'발송됨'|'서명진행중'|'체결완료'|'취소됨'),
  sent_at, completed_at, canceled_at
  // 위 필드들은 functions(Admin SDK)만 write. admin 클라이언트는 읽기 전용 + 발송 트리거(콜러블 함수 호출)만 수행
```

**보안 규칙 방향 정리** (실제 규칙 작성은 gmbs-admin 세션에서):
- `contracts`, `notices`: admin만 write, vendor는 read만
- `persons`, `products`(등록/수정요청 시): admin·vendor 둘 다 write 가능하되 vendor는 자기 brandId 범위만
- `products`의 `approval_status`/`supply_price`/`commission_rate` 필드: vendor는 이 필드를 직접 못 바꾸게 규칙에서 막고, admin만 수정 가능하도록 분리 (Firestore 규칙에서 필드 단위 제어는 `request.resource.data` 비교로 구현)
- `settlements`, `activity_log`: 시스템(추후 functions)·admin만 write, vendor는 read만 (activity_log는 vendor 아예 read도 불가)
- `inquiries`: vendor는 자기 브랜드 문서 생성 가능, `answer` 필드는 admin만 수정 가능하도록 분리

**Firestore 보안 규칙 원칙**: 로그인 && `request.auth.token.email in ['itsbeybusiness@gmail.com', 'baekeun0@gmail.com']` 인 경우에만 모든 컬렉션 읽기/쓰기 허용. 그 외는 전면 차단.

**번호 발급은 반드시 Firestore `runTransaction`으로 처리** (아래 "개발 기획 내용 누적" > 번호 발급·중복 방지 참고).

---

### 개발 기획 내용 누적

**⚠️ 이 섹션은 절대 삭제·축약하지 않는다. 기획이 바뀌면 변경 주석을 추가할 뿐이다.**

#### 1. 프로젝트 목적
GMBS에서 판매하는 일반 상품, PB상품, 위탁상품, 수공예품을 하나의 체계로 등록하고, 실제 판매 및 재고관리 단위에 맞는 바코드를 자동으로 생성하는 내부 관리용 웹앱.

**1차 버전 범위**: 브랜드 등록·브랜드코드 관리 / 상품 등록 / 옵션 등록·옵션 조합 생성 / 상품번호·옵션번호·개체번호 자동 발급 / Code128 바코드 생성 / 바코드 라벨 미리보기·출력 / Firestore 데이터 저장 / 상품 및 바코드 검색

**향후 확장 (1차 제외)**: 입고·출고·판매·반품·폐기·분실 관리 / 현재 재고 계산 / 수공예품 개체별 상태 추적(1차에서는 상태 필드만 있고 재고 거래 이력은 없음) / 위탁작가별 판매 정산 / POS 연동 / 온라인 GMBS 상품 데이터 연동 / 주문 및 판매 이력 조회 / 다중 관리자 권한(역할 구분) / 전문 데이터베이스(Supabase 등) 이전

#### 2. 핵심 데이터 개념 — 상품 / SKU / 개체
```
상품
└─ SKU
   └─ 개체
```
- **상품**: 같은 이름과 성격을 가진 상품 전체 (예: "체크 파우치", 상품ID: ONG-014). 색상·크기가 여러 개여도 모두 같은 상품에 속한다.
- **SKU**: 상품 안에서 실제로 구분해서 판매하고 재고를 관리해야 하는 옵션 조합 (예: ONG-014-01 = 빨강/소형, ONG-014-02 = 빨강/대형...). 사용자 화면에서는 상품 하나로 묶어 보여주지만, 데이터상으로는 옵션 조합별로 별도 저장한다.
- **개체**: 같은 SKU 안에서도 실물 하나하나를 구분해야 하는 경우에만 쓰는 단위. 대상: 수공예품, 빈티지 상품, 중고 상품, 실물마다 가격·상태가 다른 상품, 개별 판매 이력을 추적해야 하는 상품. 일반 상품(수량관리)은 SKU까지만 관리하고, 개체관리 상품만 SKU 아래에 개체를 생성한다.

#### 3. 바코드 식별 체계
```
브랜드코드-상품번호-옵션번호[-개체번호]
```
- 일반 상품(수량관리): `브랜드코드-상품번호-옵션번호` (예: ONG-014-02)
- 개체관리 상품: `브랜드코드-상품번호-옵션번호-개체번호` (예: DRM-027-02-003)
- Code128은 가변 길이 문자열을 지원하므로 일반 상품과 개체관리 상품의 바코드 길이가 달라도 문제없다. 바코드 리더기는 문자열 길이가 아니라 저장된 전체 값을 읽는다. **일반 상품에는 의미 없는 개체번호 "000"을 붙이지 않는다.**
- 번호별 역할: 브랜드코드=브랜드/입점작가 식별(예: ONG) / 상품번호=해당 브랜드 안의 상품 식별(예: 014) / 옵션번호=해당 상품 안의 옵션 조합 식별(예: 02) / 개체번호=해당 SKU 안의 개별 실물 식별(예: 003)
- **모든 번호는 앞자리 0을 유지하기 위해 숫자형이 아니라 문자열로 저장한다** ("001", "01", "003").

#### 4. 브랜드코드 관리
- 웹앱에 별도 "브랜드 관리" 메뉴 (브랜드 목록 / 새 브랜드 등록)
- 등록 항목: 브랜드명, 브랜드코드, 거래유형(PB/위탁/매입 등), 사용 상태
- 브랜드코드는 사용자가 직접 지정. 권장 규칙: 영문 대문자 3자리 기본, 영문+숫자만 허용, 중복 불가, 등록 후 가급적 변경 금지(브랜드명이 바뀌어도 기존 코드는 유지), 이미 상품에 사용된 브랜드코드는 삭제하지 않고 사용중지 처리
- 예: 온기→ONG, 두루미상점→DRM, 소소리잡화점→SSR, GMBS PB→GMB
- 상품 등록 화면에서는 브랜드코드를 직접 입력하지 않고 등록된 브랜드를 드롭다운에서 선택 → 시스템이 해당 brand_code를 자동 사용

#### 5. 상품번호 발급 규칙
- 상품번호는 **브랜드별로 독립적으로** 순차 발급 (예: ONG-001, ONG-002... / DRM-001, DRM-002...). 브랜드가 다르면 같은 상품번호를 사용할 수 있다.
- 처리 흐름: ① 브랜드 선택 → ② 브랜드코드 확인 → ③ 해당 브랜드에 이미 등록된 가장 큰 상품번호 조회(브랜드 문서의 `last_product_number` 필드) → ④ 다음 번호 계산 → ⑤ 예정 상품ID를 화면에 미리 표시 → ⑥ **저장 시 Firestore 트랜잭션으로 중복 여부 재검증 후 확정**
- 상품번호는 기본적으로 자동 발급하되, 관리자에게는 기존 데이터 이전 등 예외 상황을 위한 "직접 지정" 옵션 제공 가능 (체크박스로 자동/직접 전환)
- **한 번 사용한 상품번호는 상품이 판매중지·삭제 처리되어도 재사용하지 않는다.**

#### 6. 옵션번호 운영 규칙
- 모든 상품은 반드시 **두 자리** 옵션번호를 가진다 (00~99).
- 옵션이 없는 상품: 옵션번호를 `00`으로 고정 (예: ONG-014-00)
- 옵션이 있는 상품: 각 옵션 조합에 `01~99`를 순차 부여. `00`은 옵션 없음 전용으로 예약.
- 상품 하나당 99개의 옵션 조합 사용 가능. 시스템 내부적으로는 향후 세 자리 확장을 고려해 필드 길이를 과도하게 제한하지 않는다 (문자열 필드로 저장, 자릿수 검증만 UI 단에서).
- **옵션 설정 화면**: 옵션 최대 3개 축 입력 (예: 옵션1=색상[빨강,파랑], 옵션2=크기[소형,대형]) → 시스템이 가능한 조합 자동 생성 → 옵션 조합 확인 화면(순서/옵션번호/조합/판매가 테이블)에서 **최초 저장 전에만** 드래그 앤 드롭으로 순서 변경 가능 → **최초 등록 시점의 정렬 순서대로 옵션번호를 부여**

#### 7. 옵션번호와 표시 순서 분리 (중요)
- `option_number`(바코드에 들어가는 영구 식별번호)와 `option_sort_order`(화면에 보여주는 순서)는 별도 필드로 관리한다.
- 상품 저장 후 화면 표시 순서는 언제든 바꿀 수 있지만, **옵션번호 자체는 절대 변경하지 않는다** (이미 출력된 바코드·판매 기록·재고 기록과의 연결 유지 때문).
- 저장 후 새 옵션을 추가할 때는 기존 옵션번호를 그대로 유지하고, 사용하지 않은 다음 번호를 부여한다 (예: 01=빨강, 02=파랑 있는 상태에서 새로 03=초록 추가).
- 삭제하거나 사용중지한 옵션번호는 재사용하지 않는다.

#### 8. 개체번호 운영 규칙
- 개체번호는 개체관리 상품(tracking_type=individual)에만 사용. **세 자리 문자열** (001~999).
- 같은 SKU 안에서 순차 발급. 옵션번호가 달라지면 개체번호는 다시 001부터 시작할 수 있다 (SKU마다 독립).
- 판매완료·분실·폐기된 개체번호는 재사용하지 않는다.
- 개체가 999개를 넘는 예외 상황에 대비해 데이터 필드는 더 긴 문자열도 저장 가능하게 설계하되, 초기 표시·자동 발급 규칙은 세 자리로 한다.

#### 9. 등록 예시 (참고용)
- **일반 상품** (온기 / 체크 파우치 / 수량관리): 상품ID ONG-014, 옵션(색상: 빨강,파랑 / 크기: 소형,대형) → ONG-014-01=빨강/소형, -02=빨강/대형, -03=파랑/소형, -04=파랑/대형. 각 SKU의 동일 재고에는 같은 바코드를 반복 출력 (예: ONG-014-01 재고 10개 → 라벨 10장 출력).
- **옵션 없는 수공예품** (두루미상점 / 손뜨개 고양이 인형 / 개체관리): 상품ID DRM-027, SKU DRM-027-00 (옵션 없음이므로 00), 개체 3개 입력 시 DRM-027-00-001~003 생성.
- **옵션 있는 수공예품**: 옵션(크기: 소형,대형) → SKU DRM-027-01=소형, DRM-027-02=대형 → 소형 3개(DRM-027-01-001~003), 대형 2개(DRM-027-02-001~002) 생성.

#### 10. 상품 등록 화면 단계 구성
1. **기본 정보**: 브랜드, 상품명, 기본 판매가, 관리방식(수량관리/개체관리), 메모. 브랜드 선택 시 예정 상품ID 자동 표시.
2. **옵션 설정**: 옵션 없음(자동 00 SKU 생성) / 옵션 있음(최대 3개 축, 각 옵션값 입력)
3. **옵션 조합 확인**: 조합 자동 생성, 순서 드래그 변경, 옵션별 판매가 변경, 사용하지 않을 조합 제외, 옵션 표시명 수정, 옵션번호 미리보기
4. **개체 생성** (관리방식=개체관리인 경우만 노출): SKU별 생성 개수 입력 → 개체번호·최종 바코드값 미리보기
5. **최종 확인 및 저장**: 브랜드/상품명/상품ID/SKU 및 옵션번호/개체 생성 목록/판매가/생성될 바코드/저장 예정 문서 수 확인 → **저장 시 서버(클라이언트 Firestore 트랜잭션)가 번호 중복을 다시 검사한 뒤 확정**

#### 11. 상품 목록 화면
- SKU별 행을 그대로 노출하지 않고 **상품 단위로 묶어서** 표시 (상품ID / 브랜드 / 상품명 / 옵션 수 / 관리방식 / 상태)
- 상품 클릭 시 해당 상품의 SKU 목록 표시 (옵션번호 + 옵션 조합)
- 개체관리 상품은 각 SKU 아래의 개체도 확인 가능

#### 12. 바코드 생성 및 출력
- 바코드 형식: **Code128** (JsBarcode로 브라우저에서 직접 생성)
- 라벨 표시 정보(선택적 on/off): 브랜드명, 상품명, 옵션명, 개체번호, 판매가, Code128 바코드 이미지, 사람이 읽을 수 있는 바코드 문자열
- 일반 상품 라벨 예: "온기 / 체크 파우치·빨강·소형 / 15,000원 / [바코드] / ONG-014-01"
- 개체관리 상품 라벨 예: "두루미상점 / 손뜨개 고양이 인형·소형 / 28,000원 / [바코드] / DRM-027-01-003"
- 출력 기능: 개별 미리보기, 선택 바코드 일괄 출력, 동일 SKU 여러 장 출력, 연속 개체 바코드 출력, 출력 수량 지정, 라벨 크기·여백 조정, 상품명·가격 표시 여부 선택, 브라우저 인쇄, PNG 개별 다운로드, ZIP 일괄 다운로드(JSZip), 인쇄용 PDF 생성(jsPDF), 최근 출력일 기록(barcode_printed_at)
- 라벨프린터 기종·실제 라벨지 크기는 설정에서 등록 가능

#### 13. 번호 발급·중복 방지 (Firestore 트랜잭션 기반)
Google Sheets API 대신 Firestore를 쓰므로 서버(Next.js) 없이 **클라이언트에서 Firestore `runTransaction`을 직접 호출**해 아래를 처리한다. Firestore 트랜잭션은 원자적 읽기·계산·쓰기를 보장하므로, 여러 요청이 동시에 들어와도 서버 없이 번호 중복이 발생하지 않는다.

- **상품번호 발급**: 트랜잭션 안에서 ① 해당 브랜드 문서의 `last_product_number` 조회 → ② 다음 번호 계산(3자리, zero-pad) → ③ 브랜드 문서의 `last_product_number` 갱신 + 새 상품 문서 생성을 한 트랜잭션으로 커밋
- **옵션번호 발급**: 트랜잭션 안에서 상품 문서의 `last_option_number` 조회 → 다음 번호(들) 계산(2자리) → 상품 문서 갱신 + 새 SKU 문서(들) 생성을 한 트랜잭션으로 커밋
- **개체번호 발급**: 트랜잭션 안에서 SKU 문서의 `last_unit_number` 조회 → 필요한 수량만큼 연속 번호 계산(3자리) → SKU 문서 갱신 + 새 개체 문서(들) 생성을 한 트랜잭션으로 커밋
- Google Sheets는 동시성 제어가 약해 서버 레이어가 필요했지만, Firestore 트랜잭션은 이 문제를 자체적으로 해결하므로 별도 서버가 필요 없다. (Firebase Firestore + Firebase Auth 구조를 선택한 핵심 이유. 상세 판단 근거는 대화 기록 참고.)
- Firestore 보안 규칙으로 `request.auth.token.email in ALLOWED_EMAILS`만 읽기/쓰기 허용해 인증정보를 보호한다 (Google Sheets API처럼 별도 서비스 계정 키를 숨길 필요가 없는 구조).

#### 14. 수정 및 삭제 규칙
- **브랜드**: 상품에 한 번이라도 사용된 브랜드코드는 직접 변경하지 않는 것이 기본. 필요하면 브랜드명만 변경. 더 이상 사용하지 않으면 삭제하지 않고 사용중지(active=false) 처리.
- **상품**: 상품명과 가격은 수정 가능. 상품ID와 상품번호는 최초 저장 후 변경하지 않는다. 더 이상 판매하지 않으면 삭제하지 않고 판매중지 상태로 변경.
- **옵션(SKU)**: 옵션명과 표시 순서(option_sort_order)는 수정 가능. 발급된 옵션번호(option_number)는 변경하지 않는다. 더 이상 사용하지 않는 옵션은 삭제 대신 사용중지. 새 옵션은 사용하지 않은 다음 옵션번호로 추가.
- **개체**: 개체번호는 변경하지 않는다. 판매완료·분실·폐기 등의 상태만 변경. 종료된 개체번호는 재사용하지 않는다.

#### 15. 향후 재고관리 확장 구조 (1차 미구현, 구조만 대비)
- 향후 `inventory_transactions` 컬렉션 추가 예정. 재고 수량을 직접 덮어쓰지 않고 모든 변동(입고/판매/반품/폐기/분실)을 누적 기록하는 방식으로 관리.
- 필드 예상: transaction_id, sku_id, unit_id(개체관리 상품만, 일반 상품은 빈칸), transaction_type(입고/판매/반품/폐기/분실), quantity(일반 상품용 수량 변화), transaction_date, reference_id, memo
- 일반 상품 현재 재고 = 모든 거래 수량의 합계. 개체관리 상품은 수량 증감 대신 개체 상태 변화(판매중→판매완료→분실→반품 등)로 추적.
- **1차 버전에서는 재고 거래 기능을 구현하지 않지만, 추후 연결할 수 있도록 product_id, sku_id, unit_id 구조를 처음부터 유지한다** (위 "DB 구조" 참고).

#### 16. 핵심 운영 원칙 (요약, 전체 원문 유지)
1. 모든 상품은 브랜드코드, 상품번호, 옵션번호를 가진다.
2. 옵션이 없는 상품은 옵션번호 "00"을 사용한다.
3. 옵션이 있는 상품은 "01~99"를 사용한다.
4. 일반 상품은 SKU 단위로 동일한 바코드를 반복 사용한다.
5. 수공예품 등 개체관리 상품은 실제 실물마다 개체번호를 추가한다.
6. 개체관리 상품도 옵션이 있으면 옵션번호와 개체번호를 모두 사용한다.
7. 상품은 동일한 상품 기획 전체를 의미한다.
8. SKU는 실제 판매 및 재고관리 대상인 옵션 조합을 의미한다.
9. 개체는 같은 SKU 안에서도 따로 추적해야 하는 실제 실물을 의미한다.
10. 상품과 SKU는 사용자 화면에서 하나의 상품으로 묶어서 보여준다.
11. (원본: "Google Sheets의 Products 시트에서는 SKU별로 한 행씩 저장한다" → Firestore 적용: products/{productId}/skus/{skuId} 서브컬렉션으로 SKU별 문서를 저장한다.)
12. 동일 상품의 SKU는 같은 productId로 연결한다 (서브컬렉션 구조상 자동으로 보장됨).
13. 브랜드코드는 브랜드 관리 화면에서 사용자가 지정한다.
14. 상품번호는 브랜드별로 웹앱이 자동 발급한다.
15. 옵션번호는 최초 옵션 조합 순서를 바탕으로 웹앱이 자동 발급한다.
16. 옵션번호와 화면 표시 순서는 별도로 관리한다.
17. 표시 순서는 바꿀 수 있지만 발급된 옵션번호는 변경하지 않는다.
18. 한 번 사용한 상품번호·옵션번호·개체번호는 재사용하지 않는다.
19. 일반 상품과 개체관리 상품의 바코드 길이가 달라도 된다.
20. 모든 번호 필드는 문자열로 저장한다.
21. (원본: "최종 번호 발급과 중복 검사는 서버에서 처리한다" → Firestore 적용: 최종 번호 발급과 중복 검사는 클라이언트에서 호출하는 Firestore 트랜잭션으로 처리한다.)
22. (원본: "상품과 옵션 데이터는 초기에는 하나의 Products 시트에서 통합 관리한다" → Firestore 적용: products 컬렉션과 그 서브컬렉션 skus로 통합 관리한다.)
23. 카테고리는 입력하거나 관리하지 않는다.
24. (원본: "Google Sheets는 초기 데이터 저장소로 사용하되 향후 전문 데이터베이스로 이전할 수 있게 설계한다" → Firestore 적용: Firestore를 초기 데이터 저장소로 사용하며, 문서 구조 자체가 이미 관계형 DB로도 옮기기 쉬운 정규화된 형태다.)

#### 18. 항목별 작성/열람 주체 정리 (admin/vendor/functions 간 데이터 흐름)

이 프로젝트는 admin, vendor, (추후) functions가 하나의 Firebase 프로젝트를 공유하며, 항목마다 작성 주체와 열람 주체가 다르다. 아래 표를 기준으로 삼는다.

| 항목 | 작성 주체 | 열람 주체 |
|---|---|---|
| Brand 기본정보 | admin | admin, vendor(자기 브랜드만) |
| Brand 입점 상태 | admin | admin, vendor(읽기전용) |
| Person(담당자) | admin, vendor(자기 브랜드만) | admin, vendor |
| Contract | admin | admin, vendor(다운로드만) |
| Product 신규등록/수정요청 | vendor | admin(승인대기 목록), vendor(자기 요청) |
| Product 승인/거절/공급가격/수수료율 | admin | admin, vendor(승인 결과만) |
| Inventory | (추후) functions(Toss 동기화), admin(회수/폐기 등 수동 조정) | admin, vendor(읽기전용) |
| Sales | (추후) functions(Toss 웹훅) | admin, vendor(읽기전용) |
| Settlement | (추후) functions(초안 자동생성), admin(검토·확정·증빙처리) | admin, vendor(읽기전용) |
| Activity Log | admin 클라이언트(1차), (추후) functions 트리거 | admin만 |
| Notice | admin | admin, vendor |
| Inquiry 질문 | vendor | admin, vendor(자기 것만) |
| Inquiry 답변 | admin | admin, vendor(자기 것만) |

#### 19. Product 승인 워크플로우
- vendor가 신규 상품을 등록하면 `approval_status=승인대기`로 저장. 이 상태에서는 vendor 자신에게만 보이고, 실제 판매 가능 상태(`sku_status=판매중`)로 전환되지 않는다.
- admin이 승인하면 `approval_status=승인`으로 바꾸고 이때 `supply_price`, `commission_rate`, `retail_price_auto`를 함께 입력/계산한다.
- admin이 거절하면 `approval_status=거절` + `rejection_reason` 기록. vendor 화면에는 거절 사유가 노출된다.
- 이미 승인된 상품을 vendor가 수정 요청하는 경우, 기존 필드를 바로 덮어쓰지 않고 `pending_changes`에 변경 예정 값을 저장해둔다. admin이 승인하면 `pending_changes`의 값을 실제 필드로 반영하고 `pending_changes`를 비운다. 거절하면 `pending_changes`만 비우고 기존 값 유지.

#### 20. Activity Log 1차 구현 방식
- gmbs-functions가 아직 없는 지금 단계에서는, admin 클라이언트 코드가 주요 동작(계약 업로드, 상품 승인/거절, 정산 상태 변경, 브랜드 입점 상태 변경 등) 직후 `activity_log`에 직접 문서를 추가하는 방식으로 1차 구현한다.
- 이후 gmbs-functions 프로젝트가 만들어지면, 각 컬렉션에 Firestore onWrite 트리거를 걸어 자동 기록으로 전환한다. 이때 `activity_log`의 문서 스키마는 그대로 재사용하고, admin 클라이언트의 수동 기록 코드는 제거한다. (스키마를 미리 맞춰두는 이유: 나중에 트리거 방식으로 바꿔도 기존 로그 데이터와 형식이 섞이지 않도록.)

#### 21. Notice·Inquiry는 원본 명세서에 admin 쪽 기능으로 명시되어 있지 않았지만, vendor 쪽에 대응 기능이 있어 admin에도 반드시 필요하다고 판단해 추가함. (원본 명세서 누락 보완)

#### 22. 이 확장 범위에서 제외한 것
- Inventory/Sales의 실제 데이터 반영, Settlement 자동 생성 로직은 gmbs-functions 완성 후 연결한다. 이번 admin 확장 단계에서는 스키마와 (있다면) 수동 조정 화면만 만든다.
- Toss POS 연동, 웹훅 수신은 gmbs-functions 프로젝트에서 진행한다.

#### 23. 전자계약 발송·상태 연동 (유캔사인) — 18단계 예정
이전에는 "외부에서 전자계약을 진행하고 완료된 문서를 admin에 수동 업로드"하는 방식으로 계획했으나, 실제 이용 중인 유캔사인의 서명요청 생성 API와 웹훅을 활용해 발송부터 체결 완료까지 자동화하기로 확정.

**admin의 역할**: 발송 트리거(버튼) + 상태 표시만. 실제 서명·완료 문서 발급은 유캔사인이 벤더/GMBS 이메일로 직접 처리.
**구현 방식**: "전자계약 발송" 버튼 → gmbs-functions 콜러블 함수 호출 → functions이 유캔사인 API 호출 → 웹훅으로 상태 업데이트.
**admin 클라이언트 역할 제한**: `ucansign_document_id`, `status`, `sent_at`, `completed_at`, `canceled_at` 필드는 읽기 전용 (functions만 write).
**계약 상태값**: 발송대기(회색) → 발송됨(파랑) → 서명진행중(노랑) → 체결완료(초록) / 취소됨(빨강, 다시 발송 가능).
**onboarding_status 자동 전환**: 체결완료 시 functions가 브랜드 입점 상태를 자동으로 다음 단계로 전환 (admin 수동 처리 불필요).
자세한 기술 스펙은 gmbs-functions_확장_전자계약연동.md 참고.

#### 17. 원본 제안서 vs 실제 구현의 차이 (변경 이력)
- **[변경: 서버 불필요 판단]** 원본 제안서는 "Next.js 서버 + Google Sheets API + Vercel 배포"를 명시했으나, Firestore 트랜잭션이 번호 중복 방지를, Firestore 보안 규칙이 인증정보 보호를 대체할 수 있다고 판단해 **단일 HTML + Firebase(Firestore/Auth) + Vercel**로 변경. 데이터 모델·번호 체계·바코드 규칙·화면 흐름은 원본 그대로 유지.
- **[변경: 저장 형태]** 원본의 Google Sheets 4개 시트(Brands/Products/Units/Settings)를 Firestore 컬렉션 구조(brands, products→skus→units 서브컬렉션, settings)로 재구성. 필드 의미는 100% 동일하게 유지.
- **[변경: 구글 시트 직접 열람 불가]** Google Sheets를 안 쓰므로 시트에서 직접 데이터를 열람하는 방식은 없음. 대신 설정 > 데이터 관리 메뉴에서 엑셀(.xlsx) 내보내기 기능으로 대체.
- **[확정]** FCM 백그라운드 알림은 1차 버전에서 제외 (사용자 확인, 재고 알림 등 필요 시 향후 단계에서 추가 검토).
- **[확정]** PWA는 적용 (모바일 홈 화면 설치 필요, 사용자 확인).

---

### 가이드 문서 참고 내용 누적

(없음 — 첫 세션 완료 후부터 이 항목에 내용이 누적됨)

---

### 다음 세션 시작점

1~17단계 + 추가 UI 개선 완료. 계약서 템플릿 기능 보완(변수 패널, UCansign 필드 자동감지, 대량등록 가이드, 이미지 행번호 방식) 완료.

**18단계(전자계약 발송·상태 표시)** 구현 예정. gmbs-functions 콜러블 함수 연결 포함.

**이번 세션에서 구현한 내용:**
- 계약서 템플릿 "🔍 필드 가져오기": UCansign API 응답에서 fieldName + 레이블(label/description 등)을 함께 추출해 설명 칸 자동입력
- 계약서 변수 패널: 섹션 상단 배치, 변수 칩 클릭 시 클립보드 복사(`window._cpVar`), 변수 치환 미구현 안내 경고
- 계약서 템플릿 "항목 추가" 버튼 제거 (UCansign 필드 자동감지로 대체)
- 대량 등록 가이드: 엑셀 단독 / ZIP 포함 두 섹션 분리
- 상품 이미지 파일명: 복잡한 네이밍 → 엑셀 행번호(`4.jpg`, `5.jpg` 등) 방식으로 단순화

**현재 브랜치:** `main` (push 완료, Vercel 자동 배포됨)

**다음 세션 할 일:**
1. 이번 세션 구현 피드백 반영 (테스트 후 나온 수정 사항)
2. 18단계: 전자계약 발송·상태 표시(유캔사인) 실제 구현
   - 브랜드 상세 화면에 "전자계약 발송" 버튼 추가
   - gmbs-functions 콜러블 함수 연결 (functions 준비 상태 확인 필요)
   - 계약 상태 배지 실시간 표시 (발송대기→발송됨→서명진행중→체결완료/취소됨)

세션 시작 방법: Claude 데스크톱 앱 → Code 탭 → 새 세션 → Remote(☁️) → 레포 선택

---

### ⚠️ 세션 종료 규칙 요약

위 "🔴 세션 종료 시 자동 수행" 절차를 따른다. 사용자가 명시적으로 종료 의사를 밝혀야 CLAUDE.md를 갱신한다. 작업 중간에 임의로 갱신하지 말 것.
