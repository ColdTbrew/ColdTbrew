# HACKATHON — Codex CLI용 3시간 웹앱 제작 지침

> **암기할 한 문장**  
> **"ColdTbrew/ColdTbrew의 HACKATHON.md 읽고, 3000포트에 로그인·SQLite 있는 웹앱 만들어. 과제: [문제]"**

**[상세 설계/설치 가이드](docs/skh-hackathon-3000-local-webapp.md)** — 이 문서부터 전부 읽고 구현한다. 브라우저로 GitHub를 열지 말고 Codex CLI/터미널에서 읽는다.

## 환경 제약 (절대 어기지 말 것)

1. 외부로 노출되는 포트는 **3000 하나**. React UI, 로그인, API를 동일한 TanStack Start 서버에서 제공. 서버는 `0.0.0.0:3000` 바인딩.
2. 브라우저는 **시험장 데모 URL만 접속 가능**. 문서·서비스 설정·로그인용으로 다른 사이트를 열라고 요구하지 말 것. GitHub는 Codex CLI를 통한 터미널 조회 허용.
3. **외부 서비스 의존 금지**: Clerk, Neon, Supabase, Firebase, Google SSO/OAuth, 원격 DB, AI 챗봇/LLM, CDN 금지.
4. **반드시 동작:** 이메일/비밀번호 가입·로그인·로그아웃, SQLite에 실제 DB 저장, 사용자별 CRUD 및 권한 분리, 대시보드.
5. 기술 스택: **TanStack Start + React + TypeScript + shadcn/ui + Tailwind + Better Auth (이메일/비밀번호만) + SQLite + Drizzle + Zod**.
6. npm 접속은 보장하지 않음. `npm view` 등으로 검증하고, 막혀 있으면 캐시·로컬 설치 패키지·허용된 GitHub 소스 등을 활용. 설치 가능하다고 가정하지 말 것.
7. **3시간 제한:** 과제 핵심 시연을 1개 먼저 완성. 로그인 → DB CRUD → 실제 과제 기능 → UI 다듬기 → 검증. 불필요한 모듈/테스트 아키텍처/마이크로서비스 금지.
8. 인증·DB 데이터는 서버에만 두고, 비밀번호 해싱·안전한 세션 쿠키·서버 소유권 검증 사용. 비밀값이나 SQLite 파일 커밋 금지.
9. 외부 폰트, 온라인 이미지, CDN 없이 화면이 표시되어야 함. 데모 URL이 `localhost`가 아닐 수 있으므로 인증 URL/origin은 실제 접근 주소에 맞춰 설정.
10. 작업 완료 전 **3000 포트 접속, 실제 가입·로그인, 데이터 생성·조회·수정·삭제, 다른 유저 데이터 접근 차단, 서버 재시작 후 데이터 유지**를 검증하고 성공/실패를 보고.

## 프로젝트 시작 명령 (에이전트가 직접 실행)

아래는 기본 예시다. 이미 코드가 있다면 재초기화하지 말고 재사용한다. `npx` 접속은 보장되지 않으므로 실패 시 원인과 대안을 판단한다.

```bash
npx shadcn@latest init -t start
# 생성된 프로젝트 디렉터리로 이동 후
npm install better-auth drizzle-orm better-sqlite3 zod
npm install -D drizzle-kit @types/better-sqlite3
# package.json을 확인한 다음 3000 고정 실행
npm run dev -- --host 0.0.0.0 --port 3000 --strictPort
```

Better Auth 라우트 `/api/auth/*`, 모든 CRUD 서버 함수, SQLite `./data/app.sqlite`를 동일 프로세스에서 제공. 필요시 공식 GitHub 저장소의 최신 스킬/문서를 CLI로 조회하되 설치 시간을 낭비하지 말 것.

## 실전용 한 문장 프롬프트

```text
ColdTbrew/ColdTbrew의 HACKATHON.md와 링크된 상세 설계서를 GitHub CLI로 읽고 구현해.
브라우저는 데모 URL만, 포트는 3000 하나. 로그인+SQLite+CRUD 필수, 외부 서비스 금지.
지금 바로 설치부터 실행·테스트까지 해. 문제: [여기에 과제를 붙여넣기]
```

**GitHub 직접 읽기** (Codex가 인터넷/GitHub 접근 가능할 때):

```bash
curl -fsSL https://raw.githubusercontent.com/ColdTbrew/ColdTbrew/main/HACKATHON.md
curl -fsSL https://raw.githubusercontent.com/ColdTbrew/ColdTbrew/main/docs/skh-hackathon-3000-local-webapp.md
```

시험 규정에서 외부 AI나 사전 작성 코드 사용이 허용되는지 반드시 확인한다.
