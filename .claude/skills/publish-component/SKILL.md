# /publish-component - 디자인→Vue 컴포넌트 마크업 변환

## 설명
디자인 설명이나 시안 이미지를 기반으로 프로젝트 디자인 시스템을 준수하는 Vue 3 SFC 컴포넌트 마크업을 생성한다.

## 사용법
```
/publish-component 회원 상세 페이지의 포인트 이력 테이블 컴포넌트
/publish-component /path/to/design-mockup.png
/publish-component 검색 필터 + 데이터 테이블 + 페이지네이션 레이아웃
```

## 에이전트
publisher (`.claude/agents/publisher.md`)

## 실행 절차

### Step 1: 요구사항 분석
- 텍스트 설명 → 필요한 UI 요소 목록 추출
- 이미지 경로 → Read 도구로 시안 이미지 확인 → UI 요소 분석

### Step 2: 기존 패턴 조사
```
1. web-bo/src/components/uikit/ 에서 사용 가능한 Welfare* 래퍼 목록 수집
2. 유사한 기존 컴포넌트/페이지 검색 (Glob + Grep)
3. 기존 패턴의 구조·스타일·Props 인터페이스 파악
```

### Step 3: UIKit CSS 클래스 선정
```
사용할 UIKit 클래스 결정:
1. 레이아웃: wf-flex, wf-grid, wf-space-*, wf-gap-*
2. 스페이싱: wf-mt-*, wf-mb-*, wf-px-*, wf-py-*, wf-p-*, wf-m-*
3. 타이포: wf-text-* (기존 클래스 확인)
4. 버튼: wf_btn_{size}_{variant}
5. 기타: root.css 토큰 기반 var() 사용
```

### Step 4: Vue SFC 마크업 생성

생성 규칙:

#### Script 영역
```vue
<script setup lang="ts">
// 1. Vue/라이브러리 import
import { ref, computed, onMounted } from 'vue'

// 2. Welfare* 래퍼 import (최우선)
import WelfareButton from '@/components/uikit/WelfareButton.vue'

// 3. composable import
import { useExample } from '@/composables/useExample'

// 4. Props/Emits 인터페이스
interface Props {
  // ...
}
interface Emits {
  (e: 'update', value: string): void
}
const props = defineProps<Props>()
const emit = defineEmits<Emits>()

// 5. reactive state
const loading = ref(false)

// 6. computed

// 7. methods

// 8. lifecycle
onMounted(() => {
  // ...
})
</script>
```

#### Template 영역
```vue
<template>
  <!-- 시맨틱 HTML 사용 -->
  <section class="wf-p-20">
    <!-- 접근성 속성 필수 -->
    <h2>{{ title }}</h2>

    <!-- Welfare* 래퍼 우선 사용 -->
    <WelfareButton
      label="저장"
      class="wf_btn_md_default"
      :disabled="loading"
      @click="handleSubmit"
    />

    <!-- 빈 상태 처리 -->
    <DataTable :value="items" :loading="loading">
      <template #empty>
        <p>데이터가 없습니다.</p>
      </template>
    </DataTable>
  </section>
</template>
```

#### Style 영역
```vue
<style scoped>
/* UIKit 클래스로 해결 안 되는 경우만 */
/* CSS 변수 필수 사용 */
.custom-layout {
  gap: var(--d_16);
  border-radius: var(--br_4);
  color: var(--neutral-color-n-33);
}
</style>
```

### Step 5: 품질 체크리스트
생성된 컴포넌트가 다음을 준수하는지 확인:
- [ ] `<script setup lang="ts">` 사용
- [ ] Welfare* 래퍼 우선 사용
- [ ] PrimeVue 직접 사용 시 unstyled + `:pt` PassThrough
- [ ] CSS 변수 사용, 하드코딩 없음
- [ ] 시맨틱 HTML 태그
- [ ] `<img>` alt 속성
- [ ] 아이콘 버튼 aria-label
- [ ] 폼 요소 label 연결
- [ ] 로딩 상태 표시
- [ ] 빈 상태 처리
- [ ] 에러 핸들링
- [ ] 중복 제출 방지 (disabled)
- [ ] TypeScript 타입 정의

### Step 6: 결과 출력
생성된 Vue SFC 코드와 함께:
- 사용한 UIKit 클래스 목록
- 참조한 기존 컴포넌트
- 주의 사항 (커스텀 스타일 필요 여부 등)
