# /domain-model - DDD 도메인 모델 설계 가이드

## 설명
새 도메인 모듈의 Entity, Value Object, Collection VO, Exception을 설계한다.
targeting-engine의 검증된 패턴을 100% 준수한다.

## 사용법
```
/domain-model {모듈명} {엔티티 목록}
/domain-model targeting-engine TargetingContext, TargetingFieldDefinition
```

## 생성 순서
**반드시 이 순서를 따른다** (의존성 방향):
1. Exception 계층 (다른 모든 곳에서 참조)
2. Value Objects (Entity가 사용)
3. Entities (VO, Exception 사용)
4. Collection VOs (Entity를 감싸므로 Entity 이후)

---

## 1. Exception 계층 설계 (`domain/exception/`)

### 구조
```java
// 1-1. 모듈별 추상 베이스
public abstract class {Module}Exception extends DomainException {
    protected {Module}Exception(String message) {
        super(message);
    }
    protected {Module}Exception(String message, Throwable cause) {
        super(message, cause);
    }
}

// 1-2. Not Found
public class {Module}NotFoundException extends {Module}Exception {
    public {Module}NotFoundException(String message) {
        super(message);
    }
}

// 1-3. Invalid Value
public class {Module}InvalidValueException extends {Module}Exception {
    public {Module}InvalidValueException(String message) {
        super(message);
    }
}
```

### 규칙
- 모듈마다 독립된 예외 계층을 만든다
- `DomainException`을 상속한다 (`RuntimeException` 직접 상속 금지)
- 도메인 의미를 담은 이름: `Targeting`NotFoundException, `Targeting`InvalidValueException
- 필요 시 세분화: `{Module}{Entity}NotFoundException`

### 참조
- `member/common/targeting-engine/.../domain/exception/TargetingEngineException.java`
- `member/common/targeting-engine/.../domain/exception/TargetingNotFoundException.java`
- `member/common/targeting-engine/.../domain/exception/TargetingInvalidValueException.java`

---

## 2. Value Object 설계 (`domain/vo/`)

### 2a. Generic VO (타입별 행위가 다른 경우)

```java
@Getter
@ToString
@EqualsAndHashCode
public class {Domain}DataType<T> {

    private final CommonCode code;
    private final Function<String, T> resolver;

    // Static 인스턴스
    public static final {Domain}DataType<String> TEXT = new {Domain}DataType<>(..., Function.identity());
    public static final {Domain}DataType<BigDecimal> NUMBER = new {Domain}DataType<>(..., BigDecimal::new);
    public static final {Domain}DataType<LocalDate> DATE = new {Domain}DataType<>(..., LocalDate::parse);
    // ...

    public static final List<{Domain}DataType<?>> ALL = List.of(TEXT, NUMBER, DATE, ...);

    private {Domain}DataType(CommonCode code, Function<String, T> resolver) {
        this.code = code;
        this.resolver = resolver;
    }

    // 팩토리
    public static {Domain}DataType<?> from(CommonCode code) {
        return ALL.stream()
            .filter(type -> type.code.equals(code))
            .findFirst()
            .orElseThrow(() -> new {Module}InvalidValueException("..."));
    }

    // 도메인 행위
    public T resolve(String value) { return resolver.apply(value); }
    public Boolean canResolve(String value) {
        try { resolve(value); return true; }
        catch (Exception e) { return false; }
    }
}
```

### 2b. Enum-like VO (CommonCode 기반 타입 래퍼)

```java
@Getter
@ToString
@EqualsAndHashCode
public class {Domain}ArityType {

    private final CommonCode code;
    private final Predicate<{Domain}Values> matcher;

    public static final {Domain}ArityType ZERO = new {Domain}ArityType(..., values -> values.isEmpty());
    public static final {Domain}ArityType ONE = new {Domain}ArityType(..., values -> values.size() == 1);
    // ...

    public static final List<{Domain}ArityType> ALL = List.of(ZERO, ONE, ...);

    private {Domain}ArityType(CommonCode code, Predicate<{Domain}Values> matcher) {
        this.code = code;
        this.matcher = matcher;
    }

    public static {Domain}ArityType from(CommonCode code) { ... }
    public boolean isAllowed({Domain}Values values) { return matcher.test(values); }
}
```

### 2c. Scalar VO (원시 값 래핑)

```java
public record {Domain}Sequence(int value) {
    public {Domain}Sequence {
        if (value < 0) throw new {Module}InvalidValueException("...");
    }
    public static {Domain}Sequence of(int value) { return new {Domain}Sequence(value); }
}
```

### 규칙
- VO는 **행위를 포함**한다 (단순 데이터 홀더 아님)
- private 생성자 + static 팩토리 or static final 인스턴스
- `CommonCode`를 감싸서 타입 안전성 제공
- `equals`/`hashCode`는 값 기반

### 참조
- Generic VO: `member/common/targeting-engine/.../domain/vo/TargetingDataType.java`
- Enum-like VO: `member/common/targeting-engine/.../domain/vo/TargetingArityType.java`
- Scalar VO: `member/common/targeting-engine/.../domain/vo/TargetingDisplaySequence.java`

---

## 3. Entity 설계 (`domain/entity/`)

### 필수 구조

```java
@Getter
@ToString(onlyExplicitlyIncluded = true)
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class {Entity} implements AuditingEntity {

    // ========== 식별자 ==========
    @ToString.Include
    @EqualsAndHashCode.Include
    private long {entity}Key;

    // ========== 필드 ==========
    private CommonCode {type}Type;         // CommonCode 참조
    private String description;
    private UseYn enabledYn;               // 플래그는 UseYn enum

    // ========== 자식 컬렉션 ==========
    private {Child}Definitions {child}Definitions;  // Collection VO 타입

    // ========== 감사 ==========
    private AuditModifier audit = new AuditModifier();

    // ========== 기본값 상수 ==========
    private static final long DEFAULT_KEY = 0L;
    private static final String DEFAULT_DESCRIPTION = "";

    // ========== 생성자 (private only) ==========
    private {Entity}(long {entity}Key, CommonCode {type}Type, String description) {
        this.{entity}Key = {entity}Key;
        this.{type}Type = {type}Type;
        this.description = description;
    }

    // ========== 팩토리 메서드 ==========
    public static {Entity} create(CommonCode {type}Type, String description) {
        Validator.validateCreate({type}Type, description);
        return new {Entity}(DEFAULT_KEY, {type}Type, description);
    }

    public static {Entity} load(long {entity}Key, CommonCode {type}Type, String description) {
        Validator.validateLoad({entity}Key, {type}Type, description);
        return new {Entity}({entity}Key, {type}Type, description);
    }

    public static {Entity} ofDefault() {
        return new {Entity}(DEFAULT_KEY, null, DEFAULT_DESCRIPTION);
    }

    // ========== 도메인 행위 ==========
    public void assign{Child}Definitions({Child}Definitions definitions) {
        this.{child}Definitions = definitions;
    }

    public boolean isAllowed() {
        return enabledYn == UseYn.Y;
    }

    public {Child} get{Child}By(CommonCode type) {
        return {child}Definitions.get(type);
    }

    // ========== AuditingEntity 구현 ==========
    @Override
    public AuditModifier getAudit() { return audit; }
    @Override
    public AuditModifier getAuditModifier() { return audit; }
    @Override
    public void setAuditModifier(String modifier) { audit.setModifier(modifier); }

    // ========== Inner Validator ==========
    private static class Validator {
        static void validateCreate(CommonCode {type}Type, String description) {
            validate{Type}Type({type}Type);
            validateDescription(description);
        }

        static void validateLoad(long {entity}Key, CommonCode {type}Type, String description) {
            validateKey({entity}Key);
            validate{Type}Type({type}Type);
            validateDescription(description);
        }

        private static void validateKey(long key) {
            if (key <= 0) throw new {Module}InvalidValueException("{entity}Key must be positive");
        }

        private static void validate{Type}Type(CommonCode code) {
            CommonCodeCachesFinder.find(code)
                .children(CommonCodePaths.{Domain}.{Type}.PATH)
                .orElseThrow(() -> new {Module}InvalidValueException("Invalid {type}Type: " + code));
        }

        private static void validateDescription(String description) {
            if (description == null || description.isBlank()) {
                throw new {Module}InvalidValueException("description must not be blank");
            }
        }
    }
}
```

### 규칙
- **절대 금지**: `@Builder`, `@NoArgsConstructor`, public/protected 생성자
- PK: `long {entity}Key` (primitive, not `Long`)
- CommonCode 참조: `CommonCode` 타입
- 플래그: `UseYn` enum (not `String`, not `boolean`)
- 자식 컬렉션: **Collection VO 타입** (not `List<>`)
- `assign*()` 메서드로 자식 주입 (setter가 아닌 도메인 용어 사용)
- `DEFAULT_` 상수로 기본값 정의

### 참조
- `member/common/targeting-engine/.../domain/entity/TargetingContext.java`
- `member/common/targeting-engine/.../domain/entity/TargetingFieldDefinition.java`
- `member/common/targeting-engine/.../domain/entity/TargetingRuleSet.java`

---

## 4. Collection VO 설계 (`domain/entity/collection/`)

### 필수 구조

```java
public record {Entity}Definitions(List<{Entity}> items) {

    // compact constructor
    public {Entity}Definitions {
        if (items == null) {
            items = List.of();
        } else {
            items = List.copyOf(items);  // 불변 보장
        }
    }

    // ========== 팩토리 ==========
    public static {Entity}Definitions empty() {
        return new {Entity}Definitions(List.of());
    }

    public static {Entity}Definitions from(List<{Entity}> items) {
        if (items == null || items.isEmpty()) return empty();
        // 중복 제거 (fieldType 기준)
        var deduplicated = items.stream()
            .collect(Collectors.toMap(
                {Entity}::get{Type}Type,
                Function.identity(),
                (existing, replacement) -> existing,
                LinkedHashMap::new
            ))
            .values()
            .stream()
            .toList();
        return new {Entity}Definitions(deduplicated);
    }

    // ========== 도메인 조회 ==========
    public boolean contains(CommonCode type) {
        return items.stream().anyMatch(item -> item.get{Type}Type().equals(type));
    }

    public {Entity} find(CommonCode type) {
        return items.stream()
            .filter(item -> item.get{Type}Type().equals(type))
            .findFirst()
            .orElse(null);
    }

    public {Entity} get(CommonCode type) {
        return items.stream()
            .filter(item -> item.get{Type}Type().equals(type))
            .findFirst()
            .orElseThrow(() -> new {Module}NotFoundException("..."));
    }

    public boolean isEmpty() {
        return items.isEmpty();
    }
}
```

### 규칙
- Java `record` — 불변성 자동 보장
- compact constructor에서 null 처리 + `List.copyOf()`
- `from()`에서 중복 제거 로직 포함 (필요 시)
- `find()` → null 반환 (선택적 조회), `get()` → 예외 throw (필수 조회)
- `empty()` → 빈 컬렉션 팩토리

### 참조
- `member/common/targeting-engine/.../domain/entity/collection/TargetingFieldDefinitions.java`
- `member/common/targeting-engine/.../domain/entity/collection/TargetingOperatorDefinitions.java`
- `member/common/targeting-engine/.../domain/entity/collection/TargetingRuleGroups.java`

---

## 검증 기준

각 단계 완료 후:
1. **Exception**: 컴파일 성공, DomainException 상속 확인
2. **VO**: 컴파일 성공, `from(CommonCode)` 팩토리 동작
3. **Entity**: `create()` — 유효하지 않은 값 → 예외 발생, 유효한 값 → 정상 인스턴스
4. **Collection VO**: `from()` — null 입력 → empty(), 중복 입력 → 중복 제거
5. **전체**: `./gradlew build` 성공
