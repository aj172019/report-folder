# /port-adapter - Port/Adapter 설계 가이드

## 설명
도메인 Port 인터페이스와 Infrastructure Adapter 구현체를 설계한다.
Hexagonal Architecture의 핵심: 도메인이 인프라에 의존하지 않고, 인프라가 도메인에 의존한다.

## 사용법
```
/port-adapter {모듈명}
/port-adapter targeting-engine
```

## 생성 순서
1. Port 인터페이스 (domain/port/) — 도메인이 필요로 하는 계약
2. Adapter 구현 (infrastructure/adaptor/) — jOOQ 기반 실제 구현

---

## 1. Repository Port 설계 (`domain/port/repository/`)

### Command Repository (쓰기)

```java
public interface {Entity}CommandRepository {
    void insert({Entity} entity);
    void update({Entity} entity);
    void updateDescriptionByKey(long {entity}Key, String description);
    void deleteByKey(long {entity}Key);
    void deleteBy{Type}Type(CommonCode {type}Type);
}
```

### Query Repository (읽기)

```java
public interface {Feature}QueryRepository {
    List<{Entity}Query> findAll();
    List<{Entity}Query> findBy{Parent}Keys(List<Long> {parent}Keys);
}
```

### 설계 원칙
- **Command/Query 분리 필수**: 하나의 인터페이스에 읽기+쓰기 혼합 금지
- **Fine-grained**: 메서드 1개 = 유스케이스 1개
- **generic CRUD 금지**: `save()`, `findAll()`, `findById()` 같은 범용 메서드 사용 안 함
- **파라미터**: 도메인 Entity 또는 도메인 타입 (CommonCode, long key)
- **Command 반환**: `void` (insert/update/delete)
- **Query 반환**: `Query DTO` (Entity 아님) 또는 `도메인 Entity`
- **네이밍**: `{Entity}CommandRepository`, `{Entity}QueryRepository`
- 소규모 엔티티는 `{Entity}Repository` (Command + Query 통합) 가능

---

## 2. Assembler Port 설계 (`domain/port/assembler/`)

### 용도
복합 조회 — 부모-자식 관계의 계층적 데이터를 조립하는 인터페이스.
QueryService의 복잡성을 Assembler로 위임한다.

```java
public interface {Entity}Assembler {
    {Entity} findBy{Type}Type(CommonCode {type}Type);
}

public interface {Feature}QueryAssembler {
    List<{Feature}Response> findAll();
}
```

### 설계 원칙
- 반환: 도메인 Entity (자식 assigned) 또는 Query DTO (조립 완료)
- QueryService가 직접 조립하지 않고 Assembler에 위임
- 하나의 Assembler = 하나의 조립 시나리오
- **네이밍**: `{Entity}Assembler`, `{Feature}QueryAssembler`

---

## 3. Resolver Port 설계 (`domain/port/resolver/`)

### 용도
비즈니스 규칙 해석/실행. 전략 패턴 적용.
도메인 로직이지만 인프라 의존이 필요한 경우.

```java
public interface {Domain}RuleSetResolver {
    {Result} resolve({Input} input);
}
```

### 설계 원칙
- 복잡한 비즈니스 규칙의 해석/실행을 담당
- 단순 CRUD가 아닌 규칙 기반 처리
- 필요할 때만 생성 (모든 모듈에 필수는 아님)

---

## 4. Repository Adapter 구현 (`infrastructure/adaptor/repository/`)

### Command Repository Impl

```java
@Repository
@RequiredArgsConstructor
public class {Entity}CommandRepositoryImpl implements {Entity}CommandRepository {

    private static final String SYSTEM_USER_ID = "system";

    private final DSLContext dslContext;

    @Override
    public void insert({Entity} entity) {
        dslContext.insertInto(TARGETING_{TABLE})
            .columns(
                TARGETING_{TABLE}.{TYPE}_TYPE,
                TARGETING_{TABLE}.DESCRIPTION,
                TARGETING_{TABLE}.REG_ID,
                TARGETING_{TABLE}.REG_DTTM,
                TARGETING_{TABLE}.MOD_ID,
                TARGETING_{TABLE}.MOD_DTTM
            )
            .values(
                entity.get{Type}Type().codeKey(),
                entity.getDescription(),
                SYSTEM_USER_ID,
                LocalDateTime.now(),
                SYSTEM_USER_ID,
                LocalDateTime.now()
            )
            .execute();
    }

    @Override
    public void update({Entity} entity) {
        dslContext.update(TARGETING_{TABLE})
            .set(TARGETING_{TABLE}.DESCRIPTION, entity.getDescription())
            .set(TARGETING_{TABLE}.MOD_ID, SYSTEM_USER_ID)
            .set(TARGETING_{TABLE}.MOD_DTTM, LocalDateTime.now())
            .where(TARGETING_{TABLE}.{ENTITY}_KEY.eq(entity.get{Entity}Key()))
            .execute();
    }

    @Override
    public void deleteByKey(long {entity}Key) {
        dslContext.deleteFrom(TARGETING_{TABLE})
            .where(TARGETING_{TABLE}.{ENTITY}_KEY.eq({entity}Key))
            .execute();
    }
}
```

### Query Repository Impl

```java
@Repository
@RequiredArgsConstructor
public class {Feature}QueryRepositoryImpl implements {Feature}QueryRepository {

    private final DSLContext dslContext;

    @Override
    public List<{Entity}Query> findAll() {
        return dslContext
            .select(
                TARGETING_{TABLE}.{ENTITY}_KEY,
                TARGETING_{TABLE}.{TYPE}_TYPE,
                TARGETING_{TABLE}.DESCRIPTION
            )
            .from(TARGETING_{TABLE})
            .fetch(Records.mapping({Entity}Query::of));
    }

    @Override
    public List<{Entity}Query> findBy{Parent}Keys(List<Long> {parent}Keys) {
        return dslContext
            .select(
                TARGETING_{TABLE}.{ENTITY}_KEY,
                TARGETING_{TABLE}.{PARENT}_KEY,
                TARGETING_{TABLE}.{TYPE}_TYPE
            )
            .from(TARGETING_{TABLE})
            .where(TARGETING_{TABLE}.{PARENT}_KEY.in({parent}Keys))
            .fetch(Records.mapping({Entity}Query::of));
    }
}
```

### jOOQ 패턴 규칙
- `@Repository` + `@RequiredArgsConstructor`
- `private final DSLContext dslContext`
- **static import**: `import static com.fnbplus.common.jooq.generated.{schema}.Tables.*;`
- **감사 필드**: `REG_ID`/`MOD_ID` = `SYSTEM_USER_ID`, `REG_DTTM`/`MOD_DTTM` = `LocalDateTime.now()`
- **Insert**: `.insertInto(TABLE).columns(...).values(...).execute()`
- **Update**: `.update(TABLE).set(...).where(...).execute()`
- **Delete**: `.deleteFrom(TABLE).where(...).execute()`
- **Select**: `.select(...).from(TABLE).where(...).fetch(Records.mapping(...))`
- **결과 매핑**: `Records.mapping(QueryDTO::of)` — DTO의 `of()` 팩토리를 직접 참조
- **selectFrom() 금지**: 필요 컬럼만 명시적으로 select

---

## 5. Assembler Adapter 구현 (`infrastructure/adaptor/assembler/`)

```java
@Component
@RequiredArgsConstructor
public class {Feature}QueryAssemblerImpl implements {Feature}QueryAssembler {

    private final {Parent}QueryRepository {parent}QueryRepository;
    private final {Child}QueryRepository {child}QueryRepository;
    private final {GrandChild}QueryRepository {grandChild}QueryRepository;

    @Override
    public List<{Parent}Query> findAll() {
        // 1. 부모 조회
        var parents = {parent}QueryRepository.findAll();
        if (parents.isEmpty()) return parents;

        // 2. 부모 key 추출
        var parentKeys = parents.stream()
            .map({Parent}Query::get{Parent}Key)
            .toList();

        // 3. 자식 일괄 조회 (batch load — N+1 방지)
        var children = {child}QueryRepository.findBy{Parent}Keys(parentKeys);

        // 4. 손자 일괄 조회 (필요 시)
        var childKeys = children.stream()
            .map({Child}Query::get{Child}Key)
            .toList();
        var grandChildren = {grandChild}QueryRepository.findBy{Child}Keys(childKeys);

        // 5. 손자를 자식에 assign (grouping)
        var grandChildMap = grandChildren.stream()
            .collect(Collectors.groupingBy({GrandChild}Query::get{Child}Key));
        children.forEach(child ->
            child.assign{GrandChild}s(
                grandChildMap.getOrDefault(child.get{Child}Key(), List.of())
            )
        );

        // 6. 자식을 부모에 assign (grouping)
        var childMap = children.stream()
            .collect(Collectors.groupingBy({Child}Query::get{Parent}Key));
        parents.forEach(parent ->
            parent.assign{Child}s(
                childMap.getOrDefault(parent.get{Parent}Key(), List.of())
            )
        );

        return parents;
    }
}
```

### Assembler 규칙
- `@Component` + `@RequiredArgsConstructor`
- 다수 Query Repository 주입
- **N+1 방지 필수**: 루프 내 쿼리 호출 절대 금지
- **패턴**: batch load → extract keys → load children → group by parent key → assign*()
- `Collectors.groupingBy()` 로 parent key 기준 그룹핑
- `getOrDefault(key, List.of())` 로 null-safe 처리

---

## 6. Resolver Adapter 구현 (`infrastructure/adaptor/resolver/`)

```java
@Component
@RequiredArgsConstructor
public class {Domain}RuleSetResolverImpl implements {Domain}RuleSetResolver {

    private final DSLContext dslContext;

    @Override
    public {Result} resolve({Input} input) {
        // 비즈니스 규칙 기반 쿼리 구성
        // 전략 패턴으로 확장 가능
    }
}
```

- 필요 시 `spec/` 하위 패키지에 전략 구현체 분리

---

## 패키지 구조 정리

```
domain/port/
├─ repository/                  # CRUD 포트
│   ├─ {Entity}CommandRepository.java
│   └─ {Feature}QueryRepository.java
├─ assembler/                   # 복합 조회 조립 포트
│   └─ {Feature}QueryAssembler.java
└─ resolver/                    # 비즈니스 규칙 포트
    └─ {Domain}RuleSetResolver.java

infrastructure/adaptor/
├─ repository/                  # jOOQ CRUD 구현
│   ├─ {Entity}CommandRepositoryImpl.java
│   └─ {Feature}QueryRepositoryImpl.java
├─ assembler/                   # 조립 구현
│   └─ {Feature}QueryAssemblerImpl.java
└─ resolver/                    # 규칙 구현
    └─ {Domain}RuleSetResolverImpl.java
```

## 검증 기준

1. Port 인터페이스: 도메인 타입만 사용 (jOOQ, Spring 의존 없음)
2. Adapter 구현: `@Repository` 또는 `@Component` 어노테이션 존재
3. N+1 방지: Assembler에서 루프 내 쿼리 호출 없음
4. 감사 필드: insert/update 시 REG_ID/MOD_ID, REG_DTTM/MOD_DTTM 설정
5. `./gradlew build` 성공

## 참조 파일

| 유형 | 참조 파일 |
|------|----------|
| Command Repository Port | `member/common/targeting-engine/.../domain/port/repository/TargetingContextCommandRepository.java` |
| Query Repository Port | `member/common/targeting-engine/.../domain/port/repository/TargetingMetaContextQueryRepository.java` |
| Assembler Port | `member/common/targeting-engine/.../domain/port/assembler/TargetingContextAssembler.java` |
| Command Repository Impl | `member/common/targeting-engine/.../infrastructure/adaptor/repository/TargetingContextCommandRepositoryImpl.java` |
| Query Repository Impl | `member/common/targeting-engine/.../infrastructure/adaptor/repository/TargetingMetaContextQueryRepositoryImpl.java` |
| Assembler Impl | `member/common/targeting-engine/.../infrastructure/adaptor/assembler/TargetingContextAssemblerImpl.java` |
