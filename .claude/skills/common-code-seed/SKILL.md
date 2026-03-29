# /common-code-seed - ERD 공통코드 DB 시딩 및 마이그레이션 생성

ERD JSON의 codeIds를 추출하고, 다국어 이름을 부여한 뒤, dev API로 DB에 등록하고, 실제 code_key를 포함한 마이그레이션 SQL을 생성한다.

## 사용법
```
/common-code-seed fnb_promotion          # 특정 스키마
/common-code-seed fnb_promotion promotion # 스키마 + 파일명 설명
/common-code-seed                        # 전체 스키마
```

## API

- **서버**: `https://fnbplus-api-dev.incheondfs.kr`
- **로그인**: `POST /member/bo/auth:login`
- **코드 저장 (depth 1 신규)**: `POST /operation/bo/common-codes:save`
- **코드 저장 (기존 부모 밑)**: `POST /operation/bo/common-codes/{codeKey}:saveChild`
- **캐시 갱신**: `POST /operation/bo/common-codes:refresh`
- **전체 조회 (code_key 포함)**: `GET /operation/oo/common-codes/full`

## 실행 절차

### Step 1: 신규 코드 추출

```bash
node developer/scripts/common-code.mjs --schema {스키마} --dry-run
```

신규 코드 목록을 사용자에게 보여주고 대상을 확인받는다. 0건이면 종료.

### Step 2: 다국어 이름 생성

추출된 코드 트리의 각 노드에 ko-kr, en-us 이름을 부여한다. code_id에서 자연어 이름을 유추한다.

**이름 생성 규칙:**
- UPPER_SNAKE_CASE → 한국어: 의미 번역 / 영어: Title Case
- 예: `DISCOUNT_TYPE` → "할인유형" / "Discount Type"
- 예: `PENDING` → "대기" / "Pending"

테이블로 정리하여 사용자에게 검토 요청:

```
| code_id          | ko-kr      | en-us           |
|------------------|------------|-----------------|
| PROMOTION        | 프로모션    | Promotion       |
| DISCOUNT_TYPE    | 할인유형    | Discount Type   |
| RATE             | 정률       | Rate            |
| AMOUNT           | 정액       | Amount          |
```

사용자가 수정하면 반영한다.

### Step 3: 로그인 및 기존 구조 조회

1. `WebFetch` → 로그인
   ```
   POST /member/bo/auth:login
   Content-Type: application/json
   Body: {"memberId":"master","password":"1234"}
   ```
   응답에서 access token 추출

2. `WebFetch` → 기존 코드 전체 조회
   ```
   GET /operation/oo/common-codes/full
   Authorization: Bearer {token}
   ```
   기존 트리에서 신규 코드의 부모 존재 여부 확인

### Step 4: API 호출로 DB 등록

**판단 로직:**

| 상황 | API | 예시 |
|------|-----|------|
| depth 1 신규 도메인 | `:save` | PROMOTION이 없을 때 |
| 기존 부모 밑 신규 | `:saveChild` | COMMON 밑에 새 그룹 추가 |

**요청 본문 형식:**
```json
[{
  "codeId": "PROMOTION",
  "names": [
    {"languageCode": "ko-kr", "value": "프로모션"},
    {"languageCode": "en-us", "value": "Promotion"}
  ],
  "children": [{
    "codeId": "APPROVAL",
    "names": [
      {"languageCode": "ko-kr", "value": "승인"},
      {"languageCode": "en-us", "value": "Approval"}
    ],
    "children": [...]
  }]
}]
```

호출 순서:
1. `:save` 또는 `:saveChild` 호출
2. `:refresh` 캐시 갱신
3. `/common-codes/full` 재조회 → 생성된 code_key 확인
4. 결과를 사용자에게 보여줌

### Step 5: 마이그레이션 SQL 생성

full API 응답에서 **신규 코드만** 필터링하여:

1. **tb_common_code INSERT문** — 실제 code_key, upper_key 포함
2. **tb_multilingual INSERT문** — target_key = code_key, 언어별 이름
3. `CALL sync_seq(...)` 추가
4. `developer/migration/M6_data/M6_0XX__{description}.sql` 파일로 저장

**tb_multilingual INSERT 형식:**
```sql
INSERT INTO fnb_common.tb_multilingual
  (target_ckey, target_key, desc_ckey, lang_cd, cont_ckey, short_cont, long_cont, reg_id, reg_dttm, mod_id, mod_dttm)
VALUES
  (24, {code_key}, 27, 'ko-kr', 26, '프로모션', '', '', now(), '', now()),
  (24, {code_key}, 27, 'en-us', 26, 'Promotion', '', '', now(), '', now());
```
- `target_ckey = 24`: COMMON_CODE (다국어 TARGET_TYPE)
- `desc_ckey = 27`: CODE_NAME (다국어 DESCRIPTION_TYPE)
- `cont_ckey = 26`: SHORT (다국어 CONTENT_TYPE)
- `target_key`: 생성된 공통코드의 code_key
- `desc_key` 생략: identity 컬럼이므로 자동 생성

## 주의사항

- save API 응답에 code_key가 없으므로 **반드시** full API로 재조회한다
- API 호출 전 사용자에게 요청 본문을 보여주고 확인받는다
- 기존 코드와 중복되지 않도록 common-code.mjs 결과를 기준으로 한다
- access token 만료 시 재로그인한다
