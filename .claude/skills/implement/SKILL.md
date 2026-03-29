# /implement - 기존 패턴 준수 코드 구현

## 설명
분석 결과를 기반으로 기존 패턴을 100% 준수하여 코드를 구현한다. 필요 시 사전에 `/analyze`를 실행하여 패턴을 파악한다.

## 사용법
```
/implement member 신규 API 엔드포인트
/implement web-bo 검색 필터 컴포넌트
/implement member/web-bo 타겟팅 메타 CRUD
```

## 에이전트
- member 대상: developer-be (`.claude/agents/developer-be.md`)
- web-bo 대상: developer-fe (`.claude/agents/developer-fe.md`)
- 양쪽 대상: developer-be + developer-fe 둘 다

## 실행 절차

### Step 1: 기존 분석 결과 참조
이전 `/analyze` 결과가 있으면 참조한다.
없으면 대상 레포의 유사 코드를 직접 탐색하여 패턴을 파악한다.

### Step 2: 구현 계획 수립
`/write-plan` 스킬로 단계별 구현 계획을 수립한다.
- 신규 도메인 작업 시 domain-outward 순서 적용
- 기존 도메인 확장 시 영향받는 레이어만 선별

### Step 3: 레이어별 코드 생성

#### BE (domain-outward 순서)
**신규 도메인 모듈** 시:
1. **Domain Layer** — `/domain-model` 스킬 참조
   - Exception 계층 (`domain/exception/`)
   - Value Objects (`domain/vo/`)
   - Entities + Collection VOs (`domain/entity/`, `domain/entity/collection/`)
2. **Port 인터페이스** — `/port-adapter` 스킬 참조
   - Repository (Command/Query 분리) (`domain/port/repository/`)
   - Assembler (`domain/port/assembler/`)
   - Resolver (필요 시) (`domain/port/resolver/`)
3. **DTO** — `/dto-design` 스킬 참조
   - Request (Java record) (`application/dto/request/`)
   - Query (class + assign*) (`application/dto/query/`)
   - Response (class + of(Query)) (`application/dto/response/`)
4. **Service**
   - CommandService (concrete `@Service`, `@Transactional`)
   - QueryService (concrete `@Service`, `@Transactional(readOnly = true)`)
5. **Controller**
   - `CommonResponse<T>` 반환, `ResponseUtils` 사용
6. **Infrastructure** — `/port-adapter` 스킬 참조
   - Repository Impl (jOOQ)
   - Assembler Impl (batch load 패턴)
   - Resolver Impl (필요 시)

**기존 기능 수정/확장** 시:
- 영향받는 레이어만 수정
- 기존 패턴 100% 준수 (유사 코드 참조)
- 순서: 변경이 필요한 가장 안쪽 레이어부터 바깥으로

#### FE
- Vue 컴포넌트 → Composable → Service → Store → Router
- 기존 코드와 동일한 구조, 네이밍, 패턴

### Step 4: 빌드 검증
- **BE**: `cd member && ./gradlew build`
- **FE**: `cd web-bo && yarn lint && yarn type-check && yarn build`

### Step 5: 변경 파일 목록 보고
생성/수정한 파일 목록과 구현 요약을 출력한다.

## 안티패턴 (BE)

| 금지 | 올바른 방법 |
|------|------------|
| Entity에 `@Builder` | private ctor + `create()`/`load()` |
| Service interface + Impl | concrete `@Service` class |
| `ResponseEntity` 반환 | `CommonResponse<T>` via `ResponseUtils` |
| MapStruct | `of()` 정적 팩토리 |
| 루프 내 쿼리 (N+1) | Assembler batch load |
| generic CRUD (`save`, `findAll`) | use-case 기반 메서드명 |
| raw `List<Entity>` 필드 | Collection VO (record) |
