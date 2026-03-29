# /code-quality - 코드 품질 감사

## 설명
변경된 코드를 프로젝트 코딩 표준 기반으로 심층 리뷰한다. 이슈를 심각도별로 분류하여 보고한다.

## 사용법
```
/code-quality                    # git diff 기반 자동 감지
/code-quality member/common/...  # 특정 파일/디렉토리 대상
```

## 심각도 분류

| 심각도 | 설명 | 조치 |
|--------|------|------|
| **BLOCKER** | 빌드 실패, 런타임 에러, 보안 취약점, 데이터 정합성 위험 | Developer에게 반드시 피드백 |
| **WARNING** | 프로젝트 규정 위반, 성능 이슈, 유지보수성 저하 | Developer에게 피드백 |
| **NIT** | 컨벤션 미세 위반, 포매팅, 네이밍 개선 | Reviewer가 직접 수정 |

## 체크리스트

### 의존성 주입
- [ ] Field Injection (`@Autowired` 필드) 금지 → `@RequiredArgsConstructor` + `private final` 생성자 주입
- **심각도**: WARNING

### 레이어 책임 분리
- [ ] Controller에 비즈니스 로직 없음 (조건 분기, 데이터 변환 등)
- [ ] Service는 concrete class (interface + Impl 분리 아님)
- [ ] Command/Query 분리 준수 (CommandService / QueryService)
- **심각도**: WARNING

### 매핑 규칙
- [ ] MapStruct 사용 금지 → 정적 팩토리 메서드 (`of()`, `from()`, `toEntity()`)
- **심각도**: WARNING

### 코드 스타일
- [ ] `for`문 대신 Stream API 우선 사용
- [ ] 하드코딩된 설정값 금지 → `application.yml` 또는 CommonCode
- **심각도**: NIT ~ WARNING (맥락에 따라)

### Entity 규칙
- [ ] `AuditingEntity` 상속 여부
- [ ] CommonCode 검증 로직 포함 여부 (CommonCode 타입 필드가 있는 경우)
- **심각도**: WARNING

### 어노테이션 순서
올바른 순서:
```
1. 컴포넌트 어노테이션 (@RestController, @Service, @Repository, @Component)
2. @Order
3. @Transactional
4. 기타 커스텀/프레임워크 어노테이션
5. Lombok 어노테이션 (@Getter, @Builder, @RequiredArgsConstructor 등)
```
- **심각도**: NIT

### 보안
- [ ] SQL Injection 가능성 (jOOQ에서 raw SQL 사용 시)
- [ ] 인증/인가 체크 누락
- [ ] 민감 정보 로깅
- **심각도**: BLOCKER

### 기타
- [ ] 불필요한 코드나 중복 없음
- [ ] 작업 범위 외 변경 없음
- [ ] 적절한 예외 처리

## 출력 형식

```
## 코드 품질 감사 결과

### BLOCKER (N건)
1. `파일명:라인` — [카테고리] 이슈 설명
   → 권장 수정: {구체적 수정 방법}

### WARNING (N건)
1. `파일명:라인` — [카테고리] 이슈 설명
   → 권장 수정: {구체적 수정 방법}

### NIT (N건)
1. `파일명:라인` — [카테고리] 이슈 설명
   → 직접 수정 완료 / 권장 수정: {내용}

### 요약
- 총 이슈: N건 (BLOCKER: X, WARNING: Y, NIT: Z)
- 직접 수정: N건
- Developer 피드백 필요: N건
```

### DDD Entity 설계 규칙
- [ ] Entity에 public/protected constructor 없음 (private constructor + static factory만)
- [ ] Entity에 `@Builder` 없음
- [ ] Entity에 `@NoArgsConstructor` 없음
- [ ] Entity에 `create()`/`load()` 팩토리 메서드 존재
- [ ] Entity에 inner `Validator` 클래스 존재
- [ ] Entity가 `AuditingEntity` implements (extends 아님)
- [ ] Entity의 자식 컬렉션이 Collection VO 사용 (raw `List<>` 금지)
- **심각도**: WARNING

### Collection VO 규칙
- [ ] Collection VO는 Java record
- [ ] compact constructor에서 `null` 체크 + `List.copyOf()` 불변 보장
- [ ] `empty()` / `from()` 팩토리 존재
- [ ] `find()` (null 반환) / `get()` (예외 throw) 구분
- **심각도**: WARNING

### Value Object 규칙
- [ ] VO는 도메인 행위 포함 (단순 getter-only data holder 아님)
- [ ] private 생성자 + static 팩토리 또는 static final 인스턴스
- [ ] CommonCode 타입 래핑 시 `from(CommonCode)` 팩토리 존재
- **심각도**: WARNING

### DTO 3계층 규칙
- [ ] Request DTO = Java record + Jakarta validation
- [ ] Query DTO = Lombok class + private ctor + `of()` + `assign*()` 메서드
- [ ] Response DTO = Lombok class + private ctor + `of(Query)` 팩토리
- [ ] MapStruct 미사용
- **심각도**: WARNING

### Port/Adapter 규칙
- [ ] Port 인터페이스가 `domain/port/` 에 위치 (프레임워크 의존 없음)
- [ ] Adapter 구현체가 `infrastructure/adaptor/` 에 위치
- [ ] Repository가 Command/Query로 분리
- [ ] Repository에 generic CRUD 메서드(`save`, `findAll`) 없음 — use-case 기반 메서드명
- **심각도**: WARNING

### N+1 쿼리 방지
- [ ] Assembler에서 루프 내 쿼리 호출 없음
- [ ] batch load → grouping → assign 패턴 사용
- **심각도**: BLOCKER

### Exception 규칙
- [ ] 모듈별 추상 베이스 예외 존재 (`{Module}Exception extends DomainException`)
- [ ] 구체 예외: `{Module}NotFoundException`, `{Module}InvalidValueException`
- [ ] generic `RuntimeException` 또는 `Exception` 직접 throw 없음
- **심각도**: WARNING

### 응답 규칙
- [ ] Controller가 `CommonResponse<T>` 반환 (`ResponseEntity` 아님)
- [ ] `ResponseUtils.ok()` / `ResponseUtils.created()` 사용
- **심각도**: WARNING

### Service 규칙
- [ ] Service가 concrete class (interface + Impl 분리 아님)
- [ ] CommandService: `@Transactional` per method
- [ ] QueryService: `@Transactional(readOnly = true)`
- **심각도**: WARNING

## 원칙
- 체크리스트를 기계적으로 적용하되, 맥락을 고려하여 심각도를 판단
- 기존 코드의 문제는 지적하지 않음 — 변경된 코드만 대상
- NIT 이슈는 가능하면 직접 수정하여 Developer 부담을 줄임
