# Tiptap 이미지 첨부 (S3 연동) — 지시서 검토 결과 및 실행 계획

> **문서 목적**: 「Tiptap 이미지 첨부 (S3 연동)」 작업 지시서를 실제 코드베이스·문서와 대조해 **정정이 필요한 항목**을 정리하고, 이를 반영한 실행 계획을 확정한다.
> 관련 문서: [PRD.md](./PRD.md) · [API_SPEC.md](./API_SPEC.md) · [ROADMAP.md](./ROADMAP.md) · [PRD_VALIDATION.md](./PRD_VALIDATION.md) · [../CLAUDE.md](../CLAUDE.md)

---

## 1. 배경

할 일 상세 설명(Tiptap 에디터)에 이미지를 첨부할 수 있도록 `todo-backend`, `todo-frontend`, PostgreSQL 스키마를 확장한다. 로컬(dev)은 디스크, 운영(prod)은 AWS S3에 저장한다.

작성된 작업 지시서를 실제 코드·문서와 대조한 결과, **그대로 실행하면 실패하거나 프로젝트 규약을 위반하는 항목이 13건** 확인되었다. 2장에 정정 내역을, 3장에 정정을 반영한 실행 계획을 둔다.

### 1.1 확인된 현황 (실측)

| 항목 | 실제 값 |
|------|---------|
| Spring Boot | **4.1.0** / JDK 21 / Maven |
| Jackson | **3.x** (`tools.jackson.*`) — `@JsonRawValue`만 2.x 좌표 |
| AWS SDK | **미설치** |
| 테스트 스타터 | Boot 4의 세분화 스타터 (`spring-boot-starter-test` 없음) |
| Next.js / React | 16.3.1 / 19.2.8 |
| Tiptap | `@tiptap/react` · `@tiptap/starter-kit` **3.30.2** (`extension-image` 미설치) |
| 기존 테이블 | `users`, `todos` 2개 |
| `todos.content` | **JSONB** (자바 필드는 `String` + `@JdbcTypeCode(SqlTypes.JSON)`) |
| 인증 주체 식별 | `Authentication#getName()` = **email** → `UserRepository.findByEmail` |
| `WebMvcConfigurer` | **구현체 없음** (CORS도 `CorsConfigurationSource` 빈으로 처리) |
| 토스트 라이브러리 | **미설치** (인라인 `role="alert"` 패턴 사용) |

### 1.2 사전 결정 사항

| 논점 | 결정 |
|------|------|
| 소유권 위반 응답 | **404로 통일** (불변 규칙 11 준수) |
| dev 이미지 접근 | **`/uploads/**` permitAll** (UUID 난수 파일명이 unguessable URL 역할) |
| 운영 S3 URL | **퍼블릭 읽기 URL** (본문 JSONB에 박힌 URL이 만료되지 않음) |
| `todo_id` 연결 | **Todo 저장 시 본문 파싱**으로 자동 연결 |

---

## 2. 지시서 정정 사항

### 2.1 규약 위반 — 반드시 수정

#### ❌ A-1. 소유권 위반 응답 코드 (지시서 3.3 · 6단계)

지시서: *"타인 첨부 삭제 시도 403"* → **불변 규칙 11 위반**. 403은 타인 리소스의 존재 여부를 노출한다.

→ 기존 `TODO_001`(404, 타인 소유 포함)과 동일하게 **404**로 통일한다.

#### ❌ A-2. 에러코드 명명 (지시서 3.3)

지시서의 `INVALID_INPUT` / `FORBIDDEN`은 이 프로젝트 체계가 아니다. `ErrorCode` enum은 `{도메인}_{일련번호}` 형식이므로 다음을 신규 추가한다.

| 코드 | HTTP | 의미 |
|------|------|------|
| `FILE_001` | 404 | 첨부를 찾을 수 없습니다. (타인 소유 포함) |
| `FILE_002` | 400 | 허용되지 않는 이미지 형식입니다. |
| `FILE_003` | 400 | 파일 크기가 제한을 초과했습니다. |
| `FILE_004` | 500 | 파일 저장에 실패했습니다. |

---

### 2.2 그대로 실행하면 깨지는 것

#### ⚠️ A-3. `@Profile`로 저장소 빈을 나누면 **기존 테스트가 전부 실패** (지시서 3.1)

`src/test/resources/application.properties`는 프로파일을 지정하지 않는다. 따라서 `@Profile("dev")` / `@Profile("prod")` 두 빈 모두 등록되지 않아 `FileStorageService` 주입 실패 → **컨텍스트 로드 실패 → 전체 테스트 붉어짐**.

또한 지시서는 `@Profile`과 `app.upload.storage-type` 프로퍼티를 **동시에** 요구해 스위치가 이중화되어 있다.

→ **`@ConditionalOnProperty` 단일 방식**으로 통일한다.

```java
@ConditionalOnProperty(name = "app.upload.storage-type", havingValue = "LOCAL", matchIfMissing = true)
@ConditionalOnProperty(name = "app.upload.storage-type", havingValue = "S3")
```

프로퍼티를 단일 진실 공급원으로 삼고 `@Profile`은 쓰지 않는다. 테스트 프로퍼티에도 `LOCAL`을 명시한다.

#### ⚠️ A-4. `/uploads/**` 가 401이 된다 (지시서 3.4)

`SecurityConfig`는 `anyRequest().authenticated()`다. 정적 리소스 핸들러만 등록해도 브라우저 `<img src>`는 `Authorization` 헤더를 붙일 수 없어 **이미지가 전부 401**이 된다.

→ `SecurityConfig`의 `permitAll` 목록에 **`/uploads/**` 추가**. UUID 난수 파일명이 unguessable URL 역할을 하며, 운영 S3 퍼블릭 읽기와 동작이 일치한다.

#### ⚠️ A-5. 프론트 `apiFetch`로는 파일 업로드가 불가능하다 (지시서 4.3)

`lib/api/client.ts`가 `"Content-Type": "application/json"`을 **항상** 주입한다. `FormData` 전송 시 boundary가 누락되어 **업로드 자체가 실패**한다.

→ `apiFetch`에서 **body가 `FormData`면 `Content-Type`을 넣지 않는다**(브라우저가 자동 설정). 기존 호출부는 영향 없음.

#### ⚠️ A-6. 5MB 초과 응답이 공통 포맷을 벗어난다 (지시서 3.3)

`spring.servlet.multipart.max-file-size=5MB`를 넘으면 `MaxUploadSizeExceededException`이 발생하는데, `GlobalExceptionHandler`에 해당 핸들러가 없어 **500 + 비표준 바디**가 나간다.

→ `@ExceptionHandler(MaxUploadSizeExceededException.class)` 추가 → `FILE_003`(400).

---

### 2.3 지시서에 빠진 것

#### 📌 A-7. `attachments.todo_id`를 채우는 경로가 없다 (지시서 2단계)

지시서에는 업로드 API만 있고 Todo와 연결하는 지점이 없다. 결과적으로 **모든 첨부가 `todo_id = NULL`로 남아** `(todo_id, deleted_at)` 인덱스가 무의미해진다.

→ **Todo 생성/수정 시 `content` JSONB를 파싱**해 `image` 노드의 `src` URL을 추출하고, 해당 사용자 소유 첨부의 `todo_id`를 채운다. 이전 저장본에는 있었으나 본문에서 사라진 첨부는 Soft Delete 한다. `TodoService`에 후처리로 넣으며 **API 스펙 변경은 없다**.

#### 📌 A-8. `WebMvcConfigurer` 구현체가 프로젝트에 하나도 없다 (지시서 3.4)

CORS조차 `WebMvcConfigurer`가 아니라 `CorsConfigurationSource` 빈으로 처리 중이다.

→ `common/config/WebMvcConfig.java`를 **신규 생성**하고 `addResourceHandlers`로 `/uploads/**` → `file:${app.upload.local-path}/` 매핑. LOCAL일 때만 등록되도록 `@ConditionalOnProperty` 적용.

#### 📌 A-9. AWS SDK v2 의존성이 없다 (지시서 3.1)

→ `pom.xml`에 `software.amazon.awssdk:s3`를 BOM으로 추가한다. 자격증명은 **`DefaultCredentialsProvider`만** 사용하고 코드에서 키를 직접 읽지 않는다.

#### 📌 A-10. MIME 검증이 위조 가능하다 (지시서 3.3)

`MultipartFile#getContentType()`은 **클라이언트가 보낸 값**이다. 지시서의 "확장자·MIME 불일치 시 거부"만으로는 `.png`로 위장한 실행 파일을 막지 못한다.

→ 선언 MIME 화이트리스트 + 확장자 일치 검사에 더해 **매직 바이트(파일 시그니처) 검증**을 추가한다.
- JPEG/PNG/GIF: `javax.imageio.ImageIO.read()` 결과가 `null`이 아닌지로 판정 (신규 의존성 불필요)
- **WebP**: JDK 21 ImageIO가 읽지 못하므로 `RIFF....WEBP` 시그니처 바이트를 직접 검사

---

### 2.4 사실관계 오기

| # | 지시서 기술 | 실제 |
|---|-------------|------|
| A-11 | "PRD.md 3장(DB 스키마)" | DB 스키마는 **8장**(`## 8. 데이터베이스 설계`, 8.2 테이블 정의). 3장은 「사용자 여정」. 기능 요구사항이 4장인 것은 맞음 |
| A-12 | "`.env.example` 갱신" | **파일이 존재하지 않음**. 갱신이 아니라 **신규 생성**. 백엔드는 `.env`를 쓰지 않고 **시스템 환경변수 + `${ENV}` 플레이스홀더** 방식(PRD 12.1, 변수 9개)이므로 이 규약을 바꾸지 않도록 `.env.example`은 "주입할 변수 이름 목록" 문서 용도로만 만든다 |
| A-13 | "M7 하위 Task로 편입" | Task 번호는 **연속·재사용 금지**이고 최신이 `Task 039` → 신규는 **040부터**. M4·M7은 이미 ✅ 완료 표기이므로 하위에 끼워 넣지 말고 **확장 Task**로 추가 |

#### 📌 A-14. 커밋 분리 지시 누락

이 프로젝트는 **Git 저장소가 3개**다. 지시서에 언급이 없다.

| 변경 대상 | 커밋 위치 |
|-----------|-----------|
| `docs/` (이 문서 포함) | 루트 `todo-project/` |
| 백엔드 코드 | `todo-backend/` |
| 프론트 코드 | `todo-frontend/` |

---

## 3. 실행 계획

각 단계 완료 후 중단하고 진행 상황을 보고한다.

### 3.1 단계 1 — 백엔드: 엔티티 + 저장소 추상화

기존 패키지 규약(도메인은 `domain/`, 컨트롤러·DTO는 기능 패키지)을 따른다.

- `domain/attachment/Attachment.java`
  - `BaseEntity` 상속, `@SQLDelete` + `@SQLRestriction("deleted_at IS NULL")`
  - `@ManyToOne(fetch = FetchType.LAZY)` **명시** (불변 규칙 14)
  - `@Table(name = "attachments", indexes = {@Index(columnList = "todo_id, deleted_at"), @Index(columnList = "user_id, deleted_at")})`
  - `todo`는 nullable, `user`는 `nullable = false`
  - `Todo`/`User`와 동일한 `protected` 기본 생성자 + getter-only 불변 스타일
- `domain/attachment/StorageType.java` — `LOCAL, S3`
- `domain/attachment/AttachmentRepository.java`
  - `findByIdAndUser_IdAndDeletedAtIsNull`, `findByTodo_IdAndDeletedAtIsNull`, `findByUser_IdAndUrlInAndDeletedAtIsNull`
- `domain/attachment/storage/FileStorageService.java` (interface) + `StoredFile` record
  - `LocalFileStorageService` / `S3FileStorageService` (A-3의 `@ConditionalOnProperty`)
  - S3 키 규칙: `todo-images/{userId}/{uuid}.{ext}`
- `domain/attachment/AttachmentService.java` — 업로드/삭제/본문연동. 검증은 `ImageFileValidator`로 분리 (A-10)
- `pom.xml` — `software.amazon.awssdk:s3` 추가 (A-9)

#### `attachments` 테이블 정의

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | BIGINT | PK, AUTO | 첨부 ID |
| todo_id | BIGINT | FK → todos.id, NULL 허용 | 소속 Todo (작성 중에는 NULL) |
| user_id | BIGINT | FK → users.id, NOT NULL | 업로더 (소유권 검증용) |
| original_name | VARCHAR(255) | NOT NULL | 원본 파일명 |
| storage_key | VARCHAR(500) | NOT NULL | 저장 키 (S3 object key 또는 로컬 경로) |
| url | VARCHAR(1000) | NOT NULL | 접근 URL |
| content_type | VARCHAR(100) | NOT NULL | MIME 타입 |
| file_size | BIGINT | NOT NULL | 바이트 크기 |
| storage_type | VARCHAR(20) | NOT NULL | `LOCAL` / `S3` |
| created_at | TIMESTAMP | NOT NULL | 생성일 |
| updated_at | TIMESTAMP | NOT NULL | 수정일 |
| deleted_at | TIMESTAMP | NULL | Soft Delete 시각 |

**인덱스**: `(todo_id, deleted_at)`, `(user_id, deleted_at)`

> `todos` 테이블은 변경하지 않는다. 본문 `content`(JSONB)에 이미지 URL이 포함되는 구조를 유지한다.
> Todo가 Soft Delete되어도 첨부는 곧바로 물리 삭제하지 않는다 (추후 정리 배치 대상).

### 3.2 단계 2 — 백엔드: API + 설정 + 예외

- `upload/UploadController.java` + `upload/dto/UploadResponse.java`

| Method | 경로 | 응답 | 인증 |
|--------|------|------|------|
| POST | `/api/uploads/image` | `201` · `ApiResponse<UploadResponse>` | ✅ |
| DELETE | `/api/uploads/{attachmentId}` | `200` · `ApiResponse<null>` | ✅ |

  - `UploadResponse{ attachmentId, url, originalName, fileSize }`
  - 사용자 식별은 기존 `TodoController`와 동일하게 `Authentication authentication` → `getName()`(email) → `UserRepository.findByEmail`
  - 소유권 위반은 **404 `FILE_001`** (A-1)
- `common/exception/ErrorCode.java` — `FILE_001~004` 추가 (A-2)
- `common/exception/GlobalExceptionHandler.java` — `MaxUploadSizeExceededException` 핸들러 추가 (A-6)
- `common/config/WebMvcConfig.java` 신규 (A-8)
- `common/config/SecurityConfig.java` — `/uploads/**` permitAll 추가 (A-4)
- 설정 파일

```properties
# application.properties (공통)
spring.servlet.multipart.max-file-size=5MB
spring.servlet.multipart.max-request-size=10MB
app.upload.max-size=5242880

# application-dev.properties
app.upload.storage-type=LOCAL
app.upload.local-path=./uploads
app.upload.base-url=http://localhost:8080/uploads

# application-prod.properties
app.upload.storage-type=S3
app.aws.s3-bucket=${AWS_S3_BUCKET}
app.aws.region=${AWS_REGION}
```

  - ⚠️ `src/test/resources/application.properties`는 main을 **shadow**하므로 `storage-type=LOCAL` + 임시 디렉토리 경로를 **자립적으로 명시**해야 한다
  - ❌ 어떤 프로파일 파일에도 평문 시크릿을 적지 않는다
- `todo-backend/.gitignore` — `uploads/`, `.env` 추가

### 3.3 단계 3 — 백엔드: Todo 본문 연동 + 테스트

- `TodoService` 생성/수정 경로에 A-7의 첨부 동기화 후처리 추가
- 테스트 — `extends IntegrationTestSupport`, 실제 signup→login 토큰 발급 패턴, 한글 스네이크 메서드명

| 케이스 | 기대 |
|--------|------|
| 허용되지 않은 MIME (선언 MIME 위조 / 매직바이트 위조 각각) | 400 `FILE_002` |
| 5MB 초과 파일 | 400 `FILE_003` |
| 미인증 요청 | 401 `AUTH_003` |
| 타인 첨부 삭제 시도 | **404 `FILE_001`** |
| 삭제 후 재조회 | Soft Delete로 조회에서 제외 |

  - 테스트용 저장 경로는 `@TempDir` 또는 `target/test-uploads`

### 3.4 단계 4 — 프론트엔드

- `npm i @tiptap/extension-image@3.30.2` — **버전 고정**(코어 3.30.2와 일치). Tiptap은 코어/확장 버전 불일치에 민감하므로 `latest` 설치 금지
- `lib/api/client.ts` — FormData 분기 (A-5)
- `lib/api/upload.ts` — `uploadImage(file: File)`, `deleteAttachment(id: number)`
- `types/attachment.ts` — `UploadResponse` (`any` 금지)
- `components/editor/TiptapEditor.tsx`
  - `Image` 확장 추가
  - 툴바에 `ImagePlus`(lucide-react) 버튼 + 숨김 `<input type="file">` — 기존 `ToolbarButton` 패턴 재사용
  - `editorProps.handlePaste` / `handleDrop`으로 붙여넣기·드래그앤드롭 처리
  - 업로드 중 버튼 `disabled` + `Loader2` 스피너
  - 실패 시 기존 관례인 **인라인 `role="alert"` · `text-sm text-destructive`** 메시지 (토스트 라이브러리 미설치이므로 도입하지 않음)
  - 업로드 실패 시 에디터 트랜잭션을 건드리지 않아 상태가 깨지지 않게 한다
- 클라이언트 선검증(타입·5MB)으로 불필요한 요청 차단

**업로드 흐름**

```
파일 선택 / 붙여넣기 / 드롭
  → 클라이언트 검증 (타입 · 5MB)
  → POST /api/uploads/image (FormData, JWT 자동 첨부)
  → editor.chain().focus().setImage({ src: url }).run()
```

> 📌 부가 발견: `TiptapEditor`는 `value` 변경 시 `editor.commands.setContent`로 동기화하는 effect가 없다. 이번 작업 범위는 아니지만 인지해 둘 것.

### 3.5 단계 5 — E2E (Playwright)

`e2e/todo-image-upload.spec.ts` 신규. 기존 패턴대로 `page.route("**/api/uploads/image", ...)` 모킹 + `page.addInitScript` 토큰 주입.

- [ ] 파일 선택 업로드 → 본문에 `<img>` 렌더
- [ ] 드래그앤드롭 / 붙여넣기 (`DataTransfer`를 `page.evaluate`로 합성)
- [ ] 업로드 실패 시 `role="alert"` 메시지 노출

### 3.6 단계 6 — 문서 동기화 (루트 저장소에서 커밋)

- `PRD.md` **8.2**에 `attachments` 표 추가 (공통 꼬리 3행 형식 준수) + 인덱스 볼드 한 줄
- `PRD.md` **4.2**에 기능 ID `TODO-07`(이미지 첨부) 행 추가
- `API_SPEC.md` — 에러코드 표에 `FILE_001~004` 추가, 아래 두 엔드포인트를 기존 표기 형식으로 작성
  - `### 5.1 이미지 업로드 — POST /api/uploads/image`
  - `### 5.2 첨부 삭제 — DELETE /api/uploads/{attachmentId}`
  - 형식: `> 기능 ID ... · **인증 필요**` → **Request** → 필드 표 → **Response** → **에러**
- `ROADMAP.md` — `Task 040`(M4 확장, BE) / `Task 041`(M7 확장, FE) 추가 + 각 마일스톤 DoD 보강
- `todo-backend/.env.example` — 변수 **이름만** (`AWS_S3_BUCKET`, `AWS_REGION`, 로컬 테스트 한정 `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`). **값은 절대 적지 않는다**
- `todo-frontend/.env.example`

---

## 4. 보안 전제

- ❌ AWS 키를 코드·문서·Git에 하드코딩하지 않는다. 모두 환경변수로만 주입한다
- ✅ 운영(EC2)은 **IAM Role** 부여를 기본으로 하고 `DefaultCredentialsProvider`를 사용한다. 액세스 키 방식은 로컬 테스트로 한정
- ✅ 업로드 API는 **JWT 인증 필수**, 첨부는 **해당 Todo 소유자만** 등록/삭제 가능
- ✅ 파일명은 **UUID로 난수화**한다 (원본 파일명 그대로 사용 금지)
- ✅ MIME은 화이트리스트 + **매직 바이트**로 이중 검증 (A-10)
- ❌ `.env`, `application-*.properties`의 실제 값은 커밋 금지

---

## 5. 검증 방법

```bash
# 백엔드 (JDK 21 활성 + DB_PASSWORD / JWT_SECRET 주입 상태)
cd todo-backend
./mvnw test
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev

# 프론트엔드
cd todo-frontend
npm run lint && npm run build
npx playwright test e2e/todo-image-upload.spec.ts
```

**수동 확인 절차**
1. dev 서버 기동 → `/todos/new` 진입
2. 툴바 이미지 버튼으로 PNG 업로드 → 본문에 표시 확인
3. 저장 → 상세 재진입 시 이미지 유지 확인
4. `./uploads/` 에 UUID 파일 생성 확인
5. DB `attachments` 행의 `todo_id`가 채워졌는지 확인

> ⚠️ S3 경로는 운영 자격증명이 없으면 로컬 검증 불가. `S3FileStorageService`는 **단위 테스트(모킹) 수준까지만** 검증하고, 실제 S3 연동 확인은 **M9 배포 시점**으로 남긴다.

---

## 6. 범위 제외 (명시)

- 첨부 정리 배치 (고아 파일 삭제 스케줄러) — 지시서에서도 "추후 정리 배치 대상"으로 유보
- 이미지 리사이즈 / 썸네일 / 최적화
- 토스트 UI 도입 (기존 인라인 `role="alert"` 패턴 유지)
- `TiptapEditor`의 `value` 외부 동기화 버그 수정 (별건)
- Presigned URL 방식 (퍼블릭 읽기 URL로 결정)

---

### 다음 단계

이 문서의 3장 실행 계획에 따라 **단계 1(백엔드 엔티티 + 저장소 추상화)** 부터 착수한다. 각 단계 완료 시 중단하고 진행 상황을 보고한다.
