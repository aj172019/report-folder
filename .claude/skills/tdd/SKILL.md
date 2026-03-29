# /tdd - 테스트 주도 개발

## 설명
새 기능 구현 시 RED-GREEN-REFACTOR 사이클을 적용한다. 테스트를 먼저 작성하고, 최소 구현 후 리팩토링한다.

## 사용법
```
/tdd member 타겟팅 메타 CommandService
/tdd member 회원 등급 변경 로직
```

## 적용 기준
- **적용**: 대규모 작업 (6개 이상 파일 변경, 복잡한 비즈니스 로직)
- **제외**: 단순 CRUD, 설정 변경, DTO 추가, 컨벤션 수정

## 절차

### Phase 1: RED - 실패하는 테스트 작성
1. 구현할 기능의 기대 동작을 테스트로 정의
2. `common:tester` testFixtures 활용 (존재하는 경우)
3. 테스트 실행하여 **실패 확인**

```bash
cd member && ./gradlew test --tests "{테스트클래스명}"
```

테스트 작성 규칙:
- 테스트 클래스: `{대상클래스}Test.java`
- 테스트 메서드: `@DisplayName`으로 한글 설명
- Given-When-Then 구조
- 경계 조건, 예외 케이스 포함

### Phase 2: GREEN - 최소 구현
1. 테스트를 통과하는 **최소한의 코드**만 작성
2. 완벽한 코드가 아니어도 됨 — 테스트만 통과하면 OK
3. 테스트 실행하여 **성공 확인**

```bash
cd member && ./gradlew test --tests "{테스트클래스명}"
```

### Phase 3: REFACTOR - 리팩토링
1. 테스트가 통과하는 상태를 유지하면서 코드 정리
2. 중복 제거, 네이밍 개선, 패턴 준수
3. 프로젝트 컨벤션에 맞게 조정
4. 테스트 재실행하여 **여전히 성공 확인**

```bash
cd member && ./gradlew test --tests "{테스트클래스명}"
```

### Phase 4: 반복
다음 기능 단위에 대해 Phase 1-3 반복.

## 테스트 구조 예시

```java
@SpringBootTest
@Transactional
class TargetingMetaCommandServiceTest {

    @Autowired
    private TargetingMetaCommandService commandService;

    @Autowired
    private DSLContext dsl;

    @Test
    @DisplayName("컨텍스트를 정상적으로 생성한다")
    void createContext_success() {
        // Given
        var request = CreateContextRequest.builder()
            .contextType("COUPON")
            .contextDescription("쿠폰 타겟팅")
            .build();

        // When
        var result = commandService.createContext(request);

        // Then
        assertThat(result.contextKey()).isNotNull();
        assertThat(result.contextType()).isEqualTo("COUPON");
    }
}
```

## 원칙
- 테스트 없이 구현 코드를 먼저 작성하지 않는다
- GREEN 단계에서 과도한 구현을 하지 않는다 (테스트가 요구하는 만큼만)
- REFACTOR 단계에서 새 기능을 추가하지 않는다 (새 기능 → 새 RED 사이클)
