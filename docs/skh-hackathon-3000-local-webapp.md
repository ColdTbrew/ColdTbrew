# SK하이닉스 해커톤 — 3시간 풀스택 데모 설계서

**목표:** 대회 과제 공개 후 3시간 안에 로그인 + SQLite 영속 저장 + CRUD가 실제 작동하는 웹앱을 만들고, **외부 브라우저를 열지 않고도 3000번 포트에서 시연**한다.

> 2026-10-31 작성. 이 문서는 현장 제약을 우선한다. 대회 규정 및 실제 실행환경에서 외부 AI/외부 패키지 사용이 허용되는지 확인한다. **AI 챗봇, SSO, 외부 인증/DB 서비스는 만들지 않는다.**

## 0. 현장 제약 — 최우선

- **브라우저:** 데모 웹앱 주소만 열 수 있음. 설치법 검색, 콘솔 접근, 외부 서비스 설정을 위한 다른 웹사이트는 브라우저에서 열지 않는다.
- **CLI:** Codex CLI를 통해 GitHub에 접근할 수 있음. 프로젝트 생성·의존성 설치·문서 확인·실행·테스트는 터미널/에이전트가 담당.
- **외부 수신 포트:** **3000 하나만 허용.** 웹 UI, API, 로그인 엔드포인트 모두 같은 호스트/포트에서 제공. 다른 서비스를 외부 포트에 바인딩하지 않는다.
- **서비스 의존성:** Clerk, Neon, Supabase, Firebase, Google OAuth/SSO, 원격 PostgreSQL, 원격 LLM, CDN 의존 금지.
- **패키지 다운로드:** GitHub 접속 가능과 npm 레지스트리 접속 가능은 다르다. `npm view`, `npm install` 등을 먼저 시험한다. 실패하면 설치돼 있는 패키지/로컬 캐시를 재활용하거나 허용된 GitHub 소스로 확보 가능한지 검토한다. 불가능한 기능을 설치됐다고 주장하지 않는다.
- **외부 자산:** 온라인 폰트, 이미지, 맵, 분석 스크립트, CDN이 없더라도 UI가 깨지지 않도록 로컬 자산과 시스템 폰트를 사용한다.
- **계정/DB:** 데모용이어도 실제 비밀번호 해싱·세션 쿠키·사용자별 데이터 접근제어를 구현한다. 평문 비밀번호나 브라우저 localStorage 토큰 인증은 금지.
- **데이터 보존:** SQLite 파일은 쓰기 가능한 `./data/app.sqlite`에 저장. `data/`, `.env*` 등 민감/생성 데이터는 gitignore. 환경 재생성/컨테이너 삭제 후 영구 보존은 보장되지 않으므로, 필요하면 제출 전 데이터 내보내기도 고려한다.

## 1. 권장 구성

| 구성 요소 | 선택 | 이유 |
| --- | --- | --- |
| 웹 프레임워크 | TanStack Start + React + TypeScript | 프런트·SSR·Server Functions·API를 단일 앱에서 제공 |
| UI | shadcn/ui + Tailwind CSS | 폼·테이블·대시보드 빠르게 생성 |
| 인증 | Better Auth, email/password만 | 앱 내부 로그인/회원가입/로그아웃, OAuth/SSO 불필요 |
| 데이터베이스 | SQLite + better-sqlite3 | 별도 DB 프로세스나 네트워크 포트 불필요 |
| ORM/스키마 | Drizzle ORM | TypeScript 기반 CRUD, 간단한 스키마 변경 |
| 검증 | Zod | 서버 입력 데이터 검증 |
| 에이전트 참조 | TanStack 공식 docs, shadcn CLI 문서, 필요시 TanStack Intent | 설치된 버전에 맞는 API 사용 |

별도 API 서버, Redis, Postgres 컨테이너, AI 챗봇, 벡터 DB, 마이크로서비스는 도입하지 않는다. TanStack Query는 실제 캐싱 문제가 있을 때만 도입한다.

## 2. 초기 설치 및 구성 — 에이전트에게 위임

```bash
# 새 디렉터리에서 실행. 공식 TanStack Start 템플릿 기반으로 shadcn/ui까지 구성
npx shadcn@latest init -t start
```

생성된 프로젝트로 이동한 후:

```bash
npm install better-auth drizzle-orm better-sqlite3 zod
npm install -D drizzle-kit @types/better-sqlite3
```

이 명령은 **패키지 레지스트리 접속이 가능할 때**의 예시다. 실제 생성 경로, lockfile, 패키지 매니저를 확인하고 프로젝트에 맞게 조정한다. 이미 패키지가 있으면 중복 설치하지 않는다. CLI 사용법이 달라졌다면 CLI 내장 `--help`와 공식 GitHub 문서를 우선 확인한다.

공식 참조:
- TanStack Start: https://tanstack.com/start/latest/docs/framework/react/overview
- shadcn의 Start 설치: https://ui.shadcn.com/docs/installation/tanstack
- Better Auth의 TanStack Start 통합: https://www.better-auth.com/docs/integrations/tanstack
- Better Auth 설치/이메일 비밀번호: https://www.better-auth.com/docs/installation
- Drizzle SQLite: https://orm.drizzle.team/docs/get-started/sqlite-new

### 인증 구현 규칙

- Better Auth 공식 문서의 최신 TanStack Start 통합에 맞춰 동일 앱의 `/api/auth/*` 라우트에 인증 핸들러를 연결한다.
- **email/password 로그인만** 활성화한다. OAuth/SSO 버튼을 노출하지 않는다.
- Better Auth의 인증 테이블과 앱의 업무 테이블을 **동일한 로컬 SQLite 파일**에 저장하는 구조를 우선한다.
- 앱의 서버 함수/요청 핸들러에서 로그인 사용자를 검증한다. CRUD마다 `userId` 소유권 조건을 적용한다.
- `BETTER_AUTH_SECRET` 등 비밀값은 환경변수에서 읽고 저장소에 커밋하지 않는다. 비밀값 생성도 CLI에서 수행한다.
- `BETTER_AUTH_URL`·허용된 origin·쿠키 설정은 **실제 제공되는 데모 URL** 및 프록시 환경에 맞게 구성한다. 전달받은 URL을 무작정 `localhost`로 고정하지 않는다.
- 별도 외부 callback/redirect URL을 요구하지 않는다.

### SQLite 규칙

- `mkdir -p data` 이후 `./data/app.sqlite`을 사용하고, `.gitignore`에 `data/`, `.env`, `.env.local`, `*.db`, `*.sqlite`, `*.sqlite-*` 추가.
- 마이그레이션 또는 스키마 적용을 먼저 완료하고 서버를 구동한다. Better Auth가 사용하는 인증 테이블과 Drizzle 스키마가 일치하는지 확인한다.
- 서버 재시작 후 데이터가 남는지 테스트한다.
- 동시 쓰기 필요시 SQLite WAL과 busy timeout을 고려한다.

## 3. 단일 포트

개발 서버는 생성된 프로젝트의 `package.json` 스크립트를 확인한 뒤 아래 형태로 실행한다.

```bash
npm run dev -- --host 0.0.0.0 --port 3000 --strictPort
```

위 플래그는 Vite 기반 개발 서버에 대한 예시다. 스크립트가 다르면 **실제 명령을 조정해 반드시 3000만** 사용한다. 접속 가능한 URL이 `http://localhost:3000`가 아닐 수 있으므로 현장 안내된 데모 URL로 최종 확인한다.

- React 화면: `/`
- 로그인/회원가입: `/login`, `/signup` 또는 앱 내부 인증 UI
- 앱: `/dashboard`
- 인증 핸들러: `/api/auth/*` (같은 3000)
- CRUD: Server Functions 또는 동일 origin의 Server Routes

## 4. 3시간 구현 순서

| 경과 | 해야 할 일 | 최소 완료 조건 |
| --- | --- | --- |
| 0–15분 | 과제 이해·필수 시연 흐름 1개 선정·프로젝트 생성 | 화면이 3000에서 열림 |
| 15–45분 | Better Auth + SQLite 연동 | 가입/로그인/로그아웃 |
| 45–105분 | 과제의 핵심 데이터모델·입력 폼·목록·CRUD 구현 | DB에 실제 저장됨 |
| 105–145분 | 대시보드·차트·예외 처리·UI 정리 | 메인 시나리오 처음부터 끝까지 동작 |
| 145–165분 | 재시작·권한·입력 검증·프로덕션 빌드 | 재현 가능한 작동 증거 |
| 165–180분 | 제출물·스크린샷·데모 스크립트 | 공개되는 데모 URL에서 시연 가능 |

**우선순위:** 작동하는 로그인 → 실제 DB 저장/조회 → 1개 핵심 워크플로우 → 미려한 UI → 부가기능. 프로젝트 시작부터 3시간이 아니라 과제 공개 시점부터의 예산을 준수.

## 5. 완료 판정 (출력 증거 필요)

- [ ] 데모 URL 하나, 포트 3000으로 화면에 접속 가능
- [ ] 외부 서비스 호출 없이 로그인/회원가입/로그아웃 가능
- [ ] 사용자 A로 데이터 생성 → 조회 → 수정 → 삭제 동작
- [ ] 서버 재시작 후 남긴 데이터 유지
- [ ] 사용자 B로 로그인해 A 소유 데이터 조회/수정 불가
- [ ] 새로고침해도 인증 세션 유지, 로그아웃 후 보호 라우트 차단
- [ ] 잘못된 입력과 서버 오류에서 UI가 정상 복구
- [ ] `npm run build` 등 실제 빌드 성공, 실패 시 사유 보고
- [ ] 외부 폰트/CDN/DB/인증/LLM 접근 없어도 주요 기능 동작
- [ ] 비밀번호, `BETTER_AUTH_SECRET`, SQLite DB 파일을 GitHub에 커밋하지 않음

정확한 브라우저 E2E가 환경상 불가능하면 curl/CLI로 서버 응답을 확인하고, 허용된 **데모 URL의 브라우저**에서 핵심 시나리오를 직접 확인한다. 확인하지 못한 항목은 완료로 표시하지 않는다.

## 6. 실전에서 외울 한 문장

> **`ColdTbrew/ColdTbrew` GitHub의 `HACKATHON.md` 읽고 따라. 3000포트 하나로 로그인·SQLite·CRUD 되는 웹앱 만들어. 과제: [문제 붙여넣기].**

에이전트는 GitHub를 CLI로 읽고, 문서에 없는 과제별 화면·데이터모델만 추가 설계하면 된다. 브라우저로 GitHub/문서 사이트를 열 필요가 없다.

**빠른 읽기 (Codex CLI가 GitHub 접근할 수 있을 때):**

```bash
curl -fsSL https://raw.githubusercontent.com/ColdTbrew/ColdTbrew/main/HACKATHON.md
```

원본 문서와 도구 정보는 대회 직전에 재확인. npm 레지스트리 접근 실패 가능성 때문에 가능한 경우 의존성을 준비/캐시한 스타터를 사용한다.
