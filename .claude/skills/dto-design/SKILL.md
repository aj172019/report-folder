# /dto-design - DTO 3계층 설계 가이드

## 설명
Request → Query → Response 3계층 DTO 구조를 설계한다.
각 계층은 서로 다른 관심사를 담당하며, 단일 DTO가 여러 역할을 겸하지 않는다.

## 사용법
```
/dto-design {모듈명} {기능명}
/dto-design targeting-engine context CRUD
```

## 3계층 구조 개요

```
Client → [Request DTO] → Controller → Service
                                         ↓
                    Assembler → [Query DTO] → Service
                                                ↓
                              [Response DTO] → Controller → Client
```

| 계층 | 형태 | 위치 | 용도 |
|------|------|------|------|
| Request | Java record | `application/dto/request/` | API 입력 + 검증 |
| Query | Lombok class | `application/dto/query/` | 내부 조립 중간체 |
| Response | Lombok class | `application/dto/response/` | API 최종 응답 |

---

## 1. Request DTO (`application/dto/request/`)

### 구조

```java
public record {Entity}Request(
    @CommonCodeCreator(path = CommonCodePaths.{Domain}.{Type}.PATH)
    CommonCode {type}Type,

    @NotBlank
    String description
) {}
```

### Update 변형

```java
public record {Entity}UpdateRequest(
    @NotBlank
    String description
) {}
```

### 규칙
- **Java record** — 불변, 간결, 자동 equals/hashCode
- Jakarta validation: `@NotNull`, `@NotBlank`, `@Min`, `@Max`
- CommonCode 필드: `@CommonCodeCreator(path = "...")` 로 유효성 자동 검증
- 하나의 API 엔드포인트 = 하나의 Request DTO
- Create/Update 별도 DTO (공유 금지)
- 중첩 가능: `List<{Child}Request>` 필드

### 네이밍
- 생성: `{Entity}Request`
- 수정: `{Entity}UpdateRequest`
- 중첩: `{Parent}Request` 내부에 `List<{Child}Request>`

### 참조
- `member/common/targeting-engine/.../application/dto/request/TargetingContextRequest.java`
- `member/common/targeting-engine/.../application/dto/request/TargetingContextUpdateRequest.java`

---

## 2. Query DTO (`application/dto/query/`)

### 구조

```java
@Getter
@ToString(onlyExplicitlyIncluded = true)
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class {Entity}Query {

    @ToString.Include
    @EqualsAndHashCode.Include
    private long {entity}Key;

    private CommonCode {type}Type;
    private String description;

    // 자식 리스트 (Assembler가 주입)
    private List<{Child}Query> {child}Queries;

    private {Entity}Query(long {entity}Key, CommonCode {type}Type, String description) {
        this.{entity}Key = {entity}Key;
        this.{type}Type = {type}Type;
        this.description = description;
    }

    // ========== 팩토리 ==========
    public static {Entity}Query of(long {entity}Key, CommonCode {type}Type, String description) {
        return new {Entity}Query({entity}Key, {type}Type, description);
    }

    // jOOQ Records.mapping()에서 직접 참조:
    // .fetch(Records.mapping({Entity}Query::of))

    // ========== 자식 할당 ==========
    public void assign{Child}Queries(List<{Child}Query> queries) {
        this.{child}Queries = queries;
    }
}
```

### 규칙
- **Lombok class** (record 아님 — `assign*()` 메서드로 자식 할당 필요)
- `@Getter`, `@ToString(onlyExplicitlyIncluded = true)`, `@EqualsAndHashCode(onlyExplicitlyIncluded = true)`
- **private 생성자** + `static of(...)` 팩토리
- `of()` 파라미터 순서 = jOOQ select 컬럼 순서 (Records.mapping 호환)
- `assign*()` 메서드로 자식 리스트 주입 (Assembler가 호출)
- API에 직접 노출되지 않음 — 내부 전용

### 네이밍
- `{Entity}Query` — 단일 엔티티 조회 결과
- `{Feature}{Entity}Query` — 특정 기능의 조회 결과 (예: `TargetingMetaContextQuery`)

### 참조
- `member/common/targeting-engine/.../application/dto/query/TargetingContextQuery.java`
- `member/common/targeting-engine/.../application/dto/query/TargetingMetaContextQuery.java`

---

## 3. Response DTO (`application/dto/response/`)

### 구조

```java
@Getter
@ToString(onlyExplicitlyIncluded = true)
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class {Entity}Response {

    @ToString.Include
    @EqualsAndHashCode.Include
    private long {entity}Key;

    private CommonCode {type}Type;
    private String description;

    // 자식 리스트
    private List<{Child}Response> {child}Responses;

    private {Entity}Response(long {entity}Key, CommonCode {type}Type, String description) {
        this.{entity}Key = {entity}Key;
        this.{type}Type = {type}Type;
        this.description = description;
    }

    // ========== Query → Response 변환 ==========
    public static {Entity}Response of({Entity}Query query) {
        return new {Entity}Response(
            query.get{Entity}Key(),
            query.get{Type}Type(),
            query.getDescription()
        );
    }

    // ========== 자식 할당 (재귀 변환) ==========
    public void assign{Child}Responses(List<{Child}Query> queries) {
        this.{child}Responses = queries.stream()
            .map({Child}Response::of)
            .toList();
    }
}
```

### 규칙
- **Lombok class** — Query DTO와 유사한 구조
- private 생성자 + `static of(Query query)` 팩토리
- `of()`에서 Query → Response 변환
- 자식 변환: `assign*()` 메서드에서 재귀적으로 `{Child}Response::of` 호출
- API 클라이언트에게 노출되는 최종 형태

### 네이밍
- `{Entity}Response`
- `{Feature}{Entity}Response` (예: `TargetingMetaContextResponse`)

### 참조
- `member/common/targeting-engine/.../application/dto/response/TargetingContextResponse.java`
- `member/common/targeting-engine/.../application/dto/response/TargetingMetaContextResponse.java`

---

## 의사결정 트리

```
Q: 이 DTO는 어느 계층에 속하는가?

├─ API 요청 입력을 받는가?
│   └─ YES → Request (record)
│
├─ DB 조회 결과를 조립하는 중간 단계인가?
│   └─ YES → Query (class + assign*)
│
├─ API 응답으로 클라이언트에 노출되는가?
│   └─ YES → Response (class + of(Query))
│
└─ 단건 flat 조회 결과인가? (자식 없음)
    ├─ 내부 전용 → Query (Records.mapping 호환)
    └─ API 응답 → Response (of(Query) 변환)
```

## 생성 순서

1. **Request DTO** — API 경계 정의 (Controller가 먼저 이해해야 할 것)
2. **Query DTO** — 내부 조립 구조 (Repository/Assembler가 사용)
3. **Response DTO** — Query → Response 변환 (최종 API 형태)

## 검증 기준

1. Request: record 타입, validation 어노테이션 존재
2. Query: class 타입, private ctor, of() 팩토리, assign*() 메서드
3. Response: class 타입, of(Query) 팩토리로 Query → Response 변환
4. MapStruct 미사용 확인
5. `./gradlew build` 성공
