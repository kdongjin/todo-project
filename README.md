# Todo List 풀스택 프로젝트

이메일/소셜 로그인 기반 Todo List 웹앱. 사용자가 Tiptap 리치 에디터로 할 일을 작성·관리한다.

> 상세 요구사항은 [docs/PRD.md](./docs/PRD.md), API 계약은 [docs/API_SPEC.md](./docs/API_SPEC.md), 진행 순서는 [docs/ROADMAP.md](./docs/ROADMAP.md)를 참고한다. 작업 시 지켜야 할 규칙은 [CLAUDE.md](./CLAUDE.md)에 정리되어 있다.

## 모노레포 구조

이 프로젝트는 디렉토리는 한곳에 모여 있지만 **Git 저장소는 3개로 분리**되어 있다.

```
todo-project/
├── docs/              # 문서 (본 저장소, main 단일 브랜치)
├── CLAUDE.md          # 프로젝트 지침
├── todo-backend/      # 백엔드 (독립 저장소, main + develop)
└── todo-frontend/     # 프론트엔드 (독립 저장소, main + develop)
```

| 디렉토리 | 담당 | 브랜치 |
|---|---|---|
| 루트 (`docs/`, `CLAUDE.md`) | 문서 | `main` |
| `todo-backend/` | 백엔드 코드 | `main` + `develop` |
| `todo-frontend/` | 프론트엔드 코드 | `main` + `develop` |

변경한 파일이 속한 저장소에서 커밋한다. 자세한 내용은 [CLAUDE.md 3장](./CLAUDE.md#3-반드시-지킬-핵심-규칙-non-negotiable)을 참고한다.

## 사전 요구사항

- **JDK 21** — `java -version`이 21을 보고해야 한다 (17이면 `release version 21 not supported`로 빌드 실패)
- **Node.js** — Next.js 16 실행에 필요한 최신 LTS 권장
- **PostgreSQL** — 로컬 인스턴스, 스키마 `todolistdb`(개발) / `todolistdb_test`(테스트) 필요

## 기술 스택

- **Backend**: Spring Boot 4.x · JDK 21 · Maven · Spring Data JPA/Hibernate · Spring Security · JWT
- **Frontend**: Next.js 16(App Router) · React 19 · TypeScript · Tailwind CSS 4 · shadcn/ui
- **DB**: PostgreSQL

## 실행 방법

### Backend (`todo-backend/`)

```bash
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev   # 개발 서버 실행
./mvnw clean package                                     # 빌드
./mvnw test                                               # 테스트 (todolistdb_test 스키마 사용)
```

### Frontend (`todo-frontend/`)

```bash
npm run dev            # 개발 서버
npm run build           # 프로덕션 빌드
npm run lint            # 린트
```

## 환경변수

민감정보는 Git에 커밋하지 않고 환경변수로만 주입한다 (CLAUDE.md 불변 규칙 9). 값은 각자 로컬 환경에서 설정한다.

| 변수명 | 용도 |
|---|---|
| `DB_URL` | PostgreSQL 접속 URL |
| `DB_USERNAME` | DB 사용자명 |
| `DB_PASSWORD` | DB 비밀번호 |
| `JWT_SECRET` | JWT 서명 키 |
| `APP_FRONTEND_URL` | OAuth2 로그인 성공 후 리다이렉트할 프론트엔드 주소 |
| `OAUTH_GOOGLE_CLIENT_ID` / `OAUTH_GOOGLE_CLIENT_SECRET` | Google OAuth2 클라이언트 정보 |
| `OAUTH_KAKAO_CLIENT_ID` / `OAUTH_KAKAO_CLIENT_SECRET` | Kakao OAuth2 클라이언트 정보 |
| `NEXT_PUBLIC_API_BASE_URL` | 프론트엔드에서 사용할 백엔드 API 주소 (`todo-frontend/.env.local`) |

값 목록·상세 설명은 [docs/PRD.md 12장](./docs/PRD.md)을 참고한다.

## 문서 안내

- [docs/PRD.md](./docs/PRD.md) — 상세 요구사항, DB 스키마, 디자인 방향
- [docs/API_SPEC.md](./docs/API_SPEC.md) — API 요청/응답 계약
- [docs/ROADMAP.md](./docs/ROADMAP.md) — 마일스톤·Task·완료 기준
- [docs/PRD_VALIDATION.md](./docs/PRD_VALIDATION.md) — PRD 기술 검증 결과
- [docs/guides/](./docs/guides/) — 프레임워크별 개발 가이드

> ⚠️ 이 README는 초안이다. M8(Task 035)에서 실제 실행 절차를 검증한 뒤 최종본으로 다듬는다.
