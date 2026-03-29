# /analyze - 기존 코드 패턴 분석

## 설명
특정 작업/영역의 기존 코드 패턴을 분석하여 패키지 구조, 네이밍 규칙, 의존성 패턴, 공통 모듈 활용 가능 여부를 파악한다.
DDD/Clean Architecture 패턴을 포함하여 심층 분석한다.

## 사용법
```
/analyze member 타겟팅 메타 API
/analyze web-bo 회원관리 페이지
/analyze member/web-bo 타겟팅 메타 전체
```

## 에이전트
- member 대상: analyzer-be (`.claude/agents/analyzer-be.md`)
- web-bo 대상: analyzer-fe (`.claude/agents/analyzer-fe.md`)
- 양쪽 대상: analyzer-be + analyzer-fe 둘 다

## 실행 절차

### Step 1: 대상 레포 식별
인자에서 대상 레포를 판별한다:
- `member` → 백엔드 (analyzer-be)
- `web-bo` → 프론트엔드 (analyzer-fe)
- `member/web-bo` 또는 양쪽 키워드 → BE + FE 둘 다
- 명시하지 않은 경우 → 작업 내용에서 추론

### Step 2: 유사 기존 코드 탐색
지시된 작업과 유사한 기존 코드를 Grep/Glob으로 검색한다.
- BE: Controller, Service, Repository, DTO, Entity, VO, Collection VO, Exception, Assembler
- FE: Vue 컴포넌트, Composable, Service, Store, Router

### Step 3: 기본 패턴 파악
- 패키지/디렉토리 구조
- 네이밍 규칙
- 어노테이션/컴포넌트 사용법
- 의존성 주입 / API 호출 방식

### Step 3b: 도메인 모델 분석 (BE 전용)

#### Entity 패턴
- 생성자 접근 제한자 (private/protected/public)
- 팩토리 메서드 유무 (`create()`, `load()`, `ofDefault()`, `of()`)
- Inner Validator 클래스 유무
- AuditingEntity 구현 방식 (implements vs extends)
- Lombok 어노테이션 조합

#### Value Object 패턴
- `domain/vo/` 디렉토리 존재 여부
- VO 종류: Generic VO, Enum-like VO, Scalar VO
- 도메인 행위 포함 여부 (resolve, canResolve, matches 등)
- `from(CommonCode)` 팩토리 패턴

#### Collection VO 패턴
- `domain/entity/collection/` 디렉토리 존재 여부
- record 사용 여부
- `List.copyOf()` 불변 보장
- `empty()`, `from()`, `find()`, `get()` 메서드 패턴

#### Port 구조
- `domain/port/repository/` — Command/Query 분리 여부
- `domain/port/assembler/` — 복합 조회 인터페이스 유무
- `domain/port/resolver/` — 전략 패턴 인터페이스 유무
- 인터페이스 메서드: generic CRUD vs use-case 기반

#### DTO 3계층
- Request: record vs class, validation 어노테이션 종류
- Query: class + `of()` + `assign*()` 패턴 유무
- Response: `of(Query)` 변환 패턴 유무

#### Exception 계층
- `domain/exception/` 디렉토리 구조
- 모듈별 추상 베이스 유무
- DomainException 상속 여부

#### Infrastructure 패턴
- `infrastructure/adaptor/` 하위 구조 (repository, assembler, resolver)
- jOOQ DSL 패턴 (static table import, Records.mapping 등)
- 감사 필드 관리 방식

### Step 4: 공통 모듈 확인
- BE: `common/` 하위 모듈 중 활용 가능한 것 식별
  - CommonResponse / ResponseUtils (common/common)
  - AuditingEntity / AuditModifier (common/auditing)
  - CommonCode / CommonCodeCaches (common/commoncode)
  - DomainException (common/common)
- FE: `src/common/`, `components/uikit/` 활용 가능한 것 식별

### Step 5: 분석 보고서 출력
파일 경로 + 핵심 패턴 요약으로 정리하여 출력한다.
파일 전체 내용은 포함하지 않는다.

## 산출물 형식

```
## 분석 결과

### 참조 파일
- `member/.../SomeController.java` — @RequiredArgsConstructor → @RequestMapping → @RestController, CommonResponse 반환
- `member/.../SomeCommandService.java` — concrete @Service, @Transactional per method

### 패키지 구조
{새 코드가 위치해야 할 패키지 경로}

### 네이밍 규칙
{기존 코드에서 확인된 네이밍 패턴}

### 핵심 패턴
{어노테이션, DI, 쿼리 등 핵심 패턴 요약}

### 도메인 모델 패턴 (BE)
- Entity: {생성자/팩토리/Validator/AuditingEntity 구조}
- VO: {종류, 도메인 행위}
- Collection VO: {record 패턴, 불변성}
- Exception: {계층 구조}

### DTO 3계층 패턴 (BE)
- Request: {record/class, validation 패턴}
- Query: {of() + assign*() 패턴}
- Response: {of(Query) 변환 패턴}

### Infrastructure 패턴 (BE)
- Repository: {jOOQ 패턴, 감사 필드}
- Assembler: {batch load 패턴}

### 활용할 공통 모듈
{common/ 하위에서 활용해야 할 모듈 목록}
```

## 우선순위
1. `AGENTS.md` (존재 시)
2. `docs/standards/` 규정
3. 기존 코드 패턴
