# Vue 2 코드 리뷰 가이드

## 목차
- 개요
- 1. Options API 패턴 (Options API Patterns)
- 2. 컴포넌트 설계 (Component Design)
- 3. 반응성 함정 (Reactivity Pitfalls)
- 4. Mixin과 대안 패턴 (Mixins & Alternatives)
- 5. Vuex 상태 관리 (Vuex State Management)
- 6. 템플릿 모범 사례 (Template Best Practices)
- 7. 성능 최적화 (Performance Optimization)
- 8. 통합 체크리스트


## 개요

**Vue 2 코드 리뷰 가이드**는 Vue 2.x Options API 프로젝트에서 Tidy First 원칙과 Modern Software Engineering 원칙을 적용한 체계적인 코드 리뷰를 위한 문서입니다. Options API 패턴, 컴포넌트 설계, 반응성 함정, Mixin 관리, Vuex 상태 관리, 템플릿 모범 사례, 성능 최적화까지 7가지 핵심 영역을 다룹니다.

**7가지 핵심 검토 영역**:
1. **Options API Patterns** - data 함수 반환, computed 활용, Options 순서 규칙
2. **Component Design** - props 타입 검증, 이벤트 기반 통신, scoped slot 활용
3. **Reactivity Pitfalls** - Vue.set, 배열 반응성, $forceUpdate 제거
4. **Mixins & Alternatives** - mixin 충돌 방지, provide/inject, 유틸 함수 분리
5. **Vuex State Management** - mutation 동기성, 상태 직접 변경 방지, 로컬 상태 분리
6. **Template Best Practices** - v-if/v-for 분리, key 바인딩, v-html XSS 방지
7. **Performance Optimization** - Object.freeze, functional 컴포넌트, 라우트 lazy loading

> Vue 3에서는 Composition API, Teleport, Suspense 등 새로운 기능이 도입되었습니다. 본 가이드는 Vue 2.x Options API 기준이며, Vue 3 마이그레이션 관련 사항은 별도로 안내하지 않습니다.

---

## 1. Options API 패턴 (Options API Patterns)

### 검토 항목

1. `data`가 객체 리터럴이 아닌 함수로 정의되어 있는가?
2. `watch`로 구현된 로직이 `computed`로 대체 가능하지 않은가?
3. Options 순서가 스타일 가이드 권장 순서(name → components → props → data → computed → watch → lifecycle → methods)를 따르는가?
4. `computed` 속성에 부수 효과(side effect)가 포함되어 있지 않은가?
5. `methods`에서 화살표 함수를 사용하여 `this` 바인딩이 깨지지 않는가?

### 개선 패턴

#### 예시 1: data를 객체 대신 함수로 반환

**Before** (나쁜 예):
```vue
<template>
  <div class="user-card">
    <h2>{{ name }}</h2>
    <p>{{ email }}</p>
    <button @click="toggle">{{ isExpanded ? '접기' : '펼치기' }}</button>
    <div v-if="isExpanded">
      <p>{{ bio }}</p>
    </div>
  </div>
</template>

<script>
// data를 객체로 선언 — 모든 인스턴스가 같은 객체를 공유하게 됨
export default {
  name: 'UserCard',
  data: {
    name: '',
    email: '',
    bio: '',
    isExpanded: false,
  },
  methods: {
    toggle() {
      this.isExpanded = !this.isExpanded;
    },
  },
};
</script>
```

**문제점**:
- `data`가 객체 리터럴이면 같은 컴포넌트의 모든 인스턴스가 동일한 참조를 공유함
- 한 인스턴스에서 `isExpanded`를 변경하면 다른 모든 인스턴스에도 반영됨
- Vue는 경고를 출력하지만 런타임에서 미묘한 상태 공유 버그가 발생함
- 리스트 렌더링(`v-for`)에서 사용 시 즉시 문제가 드러남

**After** (좋은 예):
```vue
<template>
  <div class="user-card">
    <h2>{{ name }}</h2>
    <p>{{ email }}</p>
    <button @click="toggle">{{ isExpanded ? '접기' : '펼치기' }}</button>
    <div v-if="isExpanded">
      <p>{{ bio }}</p>
    </div>
  </div>
</template>

<script>
// data를 함수로 선언 — 각 인스턴스가 독립적인 상태를 가짐
export default {
  name: 'UserCard',
  data() {
    return {
      name: '',
      email: '',
      bio: '',
      isExpanded: false,
    };
  },
  methods: {
    toggle() {
      this.isExpanded = !this.isExpanded;
    },
  },
};
</script>
```

**개선 효과**:
- 각 컴포넌트 인스턴스가 독립적인 데이터 객체를 가져 상태 오염 방지
- `v-for`로 여러 `UserCard`를 렌더링해도 각각의 `isExpanded` 상태가 독립적
- Vue 공식 스타일 가이드의 필수(Essential) 규칙 준수
- Vue가 내부적으로 각 인스턴스마다 새로운 반응성 객체를 생성할 수 있음

#### 예시 2: watch를 computed로 대체

**Before** (나쁜 예):
```vue
<template>
  <div class="product-filter">
    <input v-model="searchQuery" placeholder="검색어 입력" />
    <select v-model="selectedCategory">
      <option value="">전체</option>
      <option v-for="cat in categories" :key="cat" :value="cat">{{ cat }}</option>
    </select>
    <ul>
      <li v-for="item in displayedProducts" :key="item.id">
        {{ item.name }} - {{ item.price }}원
      </li>
    </ul>
    <p>총 {{ totalPrice }}원</p>
  </div>
</template>

<script>
// watch로 파생 상태를 관리 — 불필요한 복잡성과 타이밍 문제 발생
export default {
  name: 'ProductFilter',
  data() {
    return {
      searchQuery: '',
      selectedCategory: '',
      products: [],
      displayedProducts: [],
      totalPrice: 0,
    };
  },
  watch: {
    searchQuery() {
      this.filterProducts();
    },
    selectedCategory() {
      this.filterProducts();
    },
    displayedProducts() {
      this.calculateTotal();
    },
  },
  methods: {
    filterProducts() {
      this.displayedProducts = this.products.filter((p) => {
        const matchQuery = p.name.includes(this.searchQuery);
        const matchCategory = !this.selectedCategory || p.category === this.selectedCategory;
        return matchQuery && matchCategory;
      });
    },
    calculateTotal() {
      this.totalPrice = this.displayedProducts.reduce((sum, p) => sum + p.price, 0);
    },
  },
};
</script>
```

**문제점**:
- `displayedProducts`와 `totalPrice`는 다른 데이터에서 파생된 값인데 별도의 `data`로 관리됨
- `watch`가 체인으로 연결되어 변경 순서와 타이밍에 따라 불일치 가능
- 초기 렌더링 시 `watch`가 실행되지 않아 빈 목록이 표시될 수 있음
- 불필요한 중간 상태 업데이트로 렌더링이 여러 번 발생함

**After** (좋은 예):
```vue
<template>
  <div class="product-filter">
    <input v-model="searchQuery" placeholder="검색어 입력" />
    <select v-model="selectedCategory">
      <option value="">전체</option>
      <option v-for="cat in categories" :key="cat" :value="cat">{{ cat }}</option>
    </select>
    <ul>
      <li v-for="item in filteredProducts" :key="item.id">
        {{ item.name }} - {{ item.price }}원
      </li>
    </ul>
    <p>총 {{ totalPrice }}원</p>
  </div>
</template>

<script>
// computed로 파생 상태를 선언적으로 관리 — 자동 캐싱 및 의존성 추적
export default {
  name: 'ProductFilter',
  data() {
    return {
      searchQuery: '',
      selectedCategory: '',
      products: [],
    };
  },
  computed: {
    filteredProducts() {
      return this.products.filter((p) => {
        const matchQuery = p.name.includes(this.searchQuery);
        const matchCategory = !this.selectedCategory || p.category === this.selectedCategory;
        return matchQuery && matchCategory;
      });
    },
    totalPrice() {
      return this.filteredProducts.reduce((sum, p) => sum + p.price, 0);
    },
  },
};
</script>
```

**개선 효과**:
- `computed`는 의존하는 데이터가 변경될 때만 재계산되어 불필요한 연산 방지
- `watch` 체인 제거로 상태 흐름이 명확해지고 타이밍 이슈 해소
- 초기 렌더링 시에도 올바른 값이 자동으로 표시됨
- `data`에서 파생 상태를 제거하여 단일 진실 원천(Single Source of Truth) 유지

#### 예시 3: Options 순서 정리

**Before** (나쁜 예):
```vue
<script>
// Options 순서가 뒤죽박죽 — 코드 가독성 저하
export default {
  methods: {
    fetchData() {
      this.loading = true;
      fetch(`/api/items?page=${this.currentPage}`)
        .then((res) => res.json())
        .then((data) => {
          this.items = data.items;
          this.totalPages = data.totalPages;
        })
        .finally(() => {
          this.loading = false;
        });
    },
    changePage(page) {
      this.currentPage = page;
      this.fetchData();
    },
  },
  data() {
    return {
      items: [],
      loading: false,
      currentPage: 1,
      totalPages: 0,
    };
  },
  name: 'ItemList',
  components: {
    Pagination: () => import('./Pagination.vue'),
  },
  watch: {
    currentPage: 'fetchData',
  },
  props: {
    categoryId: {
      type: Number,
      required: true,
    },
  },
  mounted() {
    this.fetchData();
  },
  computed: {
    hasItems() {
      return this.items.length > 0;
    },
  },
};
</script>
```

**문제점**:
- `methods`가 맨 위에, `name`이 중간에 있어 컴포넌트 파악이 어려움
- 팀원마다 다른 순서로 작성하면 코드 리뷰 시 diff가 불필요하게 커짐
- `data`와 `computed`가 떨어져 있어 상태 구조를 한눈에 파악하기 어려움
- ESLint `vue/order-in-components` 규칙 위반

**After** (좋은 예):
```vue
<script>
// Vue 공식 스타일 가이드 권장 순서 준수
export default {
  name: 'ItemList',

  components: {
    Pagination: () => import('./Pagination.vue'),
  },

  props: {
    categoryId: {
      type: Number,
      required: true,
    },
  },

  data() {
    return {
      items: [],
      loading: false,
      currentPage: 1,
      totalPages: 0,
    };
  },

  computed: {
    hasItems() {
      return this.items.length > 0;
    },
  },

  watch: {
    currentPage: 'fetchData',
  },

  mounted() {
    this.fetchData();
  },

  methods: {
    fetchData() {
      this.loading = true;
      fetch(`/api/items?page=${this.currentPage}`)
        .then((res) => res.json())
        .then((data) => {
          this.items = data.items;
          this.totalPages = data.totalPages;
        })
        .finally(() => {
          this.loading = false;
        });
    },
    changePage(page) {
      this.currentPage = page;
      this.fetchData();
    },
  },
};
</script>
```

**개선 효과**:
- `name` → `components` → `props` → `data` → `computed` → `watch` → lifecycle → `methods` 순서로 일관성 확보
- 컴포넌트를 열었을 때 이름, 의존성, 입력(props), 상태(data), 파생(computed) 순으로 빠르게 파악 가능
- ESLint `vue/order-in-components` 규칙과 호환되어 자동 검증 가능
- 팀 전체가 동일한 순서를 따르면 코드 리뷰 속도 향상

---

## 2. 컴포넌트 설계 (Component Design)

### 검토 항목

1. `props`가 배열이 아닌 객체 형태로 타입 검증이 되어 있는가?
2. 자식 컴포넌트가 `$parent`로 부모에 직접 접근하지 않는가?
3. 재사용 가능한 컴포넌트에 scoped slot이 적절히 활용되어 있는가?
4. 컴포넌트 이름이 PascalCase 또는 kebab-case 규칙을 따르는가?
5. 이벤트 이름이 kebab-case를 따르는가? (`$emit('update-item')`)

### 개선 패턴

#### 예시 1: props 타입 검증 강화

**Before** (나쁜 예):
```vue
<template>
  <div class="notification-banner" :class="type">
    <span class="icon">{{ icon }}</span>
    <div class="content">
      <h4>{{ title }}</h4>
      <p>{{ message }}</p>
    </div>
    <button v-if="dismissible" @click="$emit('dismiss')">닫기</button>
  </div>
</template>

<script>
// props를 배열로 선언 — 타입 검증 전혀 없음
export default {
  name: 'NotificationBanner',
  props: ['type', 'title', 'message', 'icon', 'dismissible'],
};
</script>
```

**문제점**:
- `type`에 문자열이 아닌 값이 전달되어도 경고 없이 렌더링됨
- `title`이 누락되어도 빈 `<h4>` 태그가 렌더링됨
- `dismissible`에 문자열 `"false"`가 전달되면 truthy로 평가되어 닫기 버튼이 표시됨
- 다른 개발자가 어떤 props를 어떤 타입으로 전달해야 하는지 알 수 없음

**After** (좋은 예):
```vue
<template>
  <div class="notification-banner" :class="type">
    <span class="icon">{{ icon }}</span>
    <div class="content">
      <h4>{{ title }}</h4>
      <p>{{ message }}</p>
    </div>
    <button v-if="dismissible" @click="$emit('dismiss')">닫기</button>
  </div>
</template>

<script>
// props를 객체로 선언 — 타입, 필수 여부, 기본값, 유효성 검증 포함
export default {
  name: 'NotificationBanner',
  props: {
    type: {
      type: String,
      default: 'info',
      validator(value) {
        return ['info', 'success', 'warning', 'error'].includes(value);
      },
    },
    title: {
      type: String,
      required: true,
    },
    message: {
      type: String,
      required: true,
    },
    icon: {
      type: String,
      default: 'info-circle',
    },
    dismissible: {
      type: Boolean,
      default: false,
    },
  },
};
</script>
```

**개선 효과**:
- 잘못된 `type` 값(예: `'danger'`)이 전달되면 개발 모드에서 경고가 출력됨
- `title`과 `message`가 필수로 표시되어 누락 시 즉시 알 수 있음
- `dismissible`이 Boolean 타입으로 명시되어 `"false"` 문자열이 전달되면 경고 출력
- props 정의가 컴포넌트의 자체 문서 역할을 하여 별도 문서 없이도 사용법 파악 가능

#### 예시 2: $parent 접근을 event emit으로 대체

**Before** (나쁜 예):
```vue
<!-- ChildForm.vue -->
<template>
  <form @submit.prevent="submit">
    <input v-model="localName" placeholder="이름" />
    <input v-model="localEmail" placeholder="이메일" />
    <button type="submit">저장</button>
    <button type="button" @click="cancel">취소</button>
  </form>
</template>

<script>
// $parent로 부모 컴포넌트의 상태와 메서드에 직접 접근
export default {
  name: 'ChildForm',
  data() {
    return {
      localName: '',
      localEmail: '',
    };
  },
  mounted() {
    // 부모의 데이터에 직접 접근 — 부모 구조에 강하게 결합됨
    this.localName = this.$parent.user.name;
    this.localEmail = this.$parent.user.email;
  },
  methods: {
    submit() {
      // 부모의 메서드를 직접 호출 — 부모가 이 메서드를 가지고 있어야 함
      this.$parent.saveUser({
        name: this.localName,
        email: this.localEmail,
      });
      // 부모의 상태를 직접 변경
      this.$parent.isEditing = false;
    },
    cancel() {
      this.$parent.isEditing = false;
      this.$parent.showNotification('편집이 취소되었습니다');
    },
  },
};
</script>
```

**문제점**:
- 부모 컴포넌트의 `user`, `saveUser`, `isEditing`, `showNotification`에 강하게 결합
- 부모 컴포넌트가 변경되면 자식도 함께 수정해야 함
- 다른 부모 컴포넌트에서 재사용할 수 없음
- 단위 테스트 시 부모 컴포넌트를 목킹해야 하는 복잡성 증가

**After** (좋은 예):
```vue
<!-- ChildForm.vue -->
<template>
  <form @submit.prevent="submit">
    <input v-model="localName" placeholder="이름" />
    <input v-model="localEmail" placeholder="이메일" />
    <button type="submit">저장</button>
    <button type="button" @click="cancel">취소</button>
  </form>
</template>

<script>
// props로 데이터를 받고, event로 결과를 전달 — 단방향 데이터 흐름
export default {
  name: 'ChildForm',
  props: {
    user: {
      type: Object,
      required: true,
      validator(value) {
        return typeof value.name === 'string' && typeof value.email === 'string';
      },
    },
  },
  data() {
    return {
      localName: this.user.name,
      localEmail: this.user.email,
    };
  },
  methods: {
    submit() {
      this.$emit('save', {
        name: this.localName,
        email: this.localEmail,
      });
    },
    cancel() {
      this.$emit('cancel');
    },
  },
};
</script>
```

```vue
<!-- ParentPage.vue -->
<template>
  <div>
    <child-form
      v-if="isEditing"
      :user="user"
      @save="saveUser"
      @cancel="cancelEdit"
    />
  </div>
</template>

<script>
import ChildForm from './ChildForm.vue';

export default {
  name: 'ParentPage',
  components: { ChildForm },
  data() {
    return {
      user: { name: '', email: '' },
      isEditing: true,
    };
  },
  methods: {
    saveUser(userData) {
      this.user = userData;
      this.isEditing = false;
    },
    cancelEdit() {
      this.isEditing = false;
      this.showNotification('편집이 취소되었습니다');
    },
    showNotification(message) {
      // 알림 표시 로직
    },
  },
};
</script>
```

**개선 효과**:
- `ChildForm`이 부모 구조에 의존하지 않아 어디서든 재사용 가능
- props + events로 단방향 데이터 흐름이 명확하게 유지됨
- 단위 테스트 시 props만 전달하고 emit된 이벤트만 검증하면 됨
- Vue DevTools에서 이벤트 흐름을 쉽게 추적할 수 있음

#### 예시 3: scoped slot 활용

**Before** (나쁜 예):
```vue
<!-- DataTable.vue -->
<template>
  <table>
    <thead>
      <tr>
        <th v-for="col in columns" :key="col.key">{{ col.label }}</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="row in data" :key="row.id">
        <!-- 모든 셀 렌더링이 하드코딩되어 있음 -->
        <td>{{ row.name }}</td>
        <td>{{ row.email }}</td>
        <td>
          <span :class="row.status === 'active' ? 'green' : 'red'">
            {{ row.status }}
          </span>
        </td>
        <td>
          <button @click="$emit('edit', row)">편집</button>
          <button @click="$emit('delete', row)">삭제</button>
        </td>
      </tr>
    </tbody>
  </table>
</template>

<script>
// 테이블 컴포넌트가 특정 데이터 구조에 종속됨
export default {
  name: 'DataTable',
  props: {
    columns: Array,
    data: Array,
  },
};
</script>
```

**문제점**:
- `name`, `email`, `status` 필드가 하드코딩되어 다른 데이터 구조에 사용 불가
- 셀 렌더링 방식을 변경하려면 `DataTable` 컴포넌트 자체를 수정해야 함
- 액션 버튼(편집/삭제)도 하드코딩되어 다른 액션이 필요한 곳에서 재사용 불가
- 컴포넌트의 범용성이 완전히 상실됨

**After** (좋은 예):
```vue
<!-- DataTable.vue -->
<template>
  <table>
    <thead>
      <tr>
        <th v-for="col in columns" :key="col.key">{{ col.label }}</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="row in data" :key="row.id">
        <td v-for="col in columns" :key="col.key">
          <!-- scoped slot으로 셀 렌더링을 부모에게 위임 -->
          <slot :name="'cell-' + col.key" :row="row" :value="row[col.key]">
            <!-- 기본 렌더링: slot이 제공되지 않으면 단순 텍스트 출력 -->
            {{ row[col.key] }}
          </slot>
        </td>
      </tr>
      <tr v-if="data.length === 0">
        <td :colspan="columns.length">
          <slot name="empty">데이터가 없습니다.</slot>
        </td>
      </tr>
    </tbody>
  </table>
</template>

<script>
// scoped slot으로 셀 렌더링을 유연하게 커스터마이징 가능
export default {
  name: 'DataTable',
  props: {
    columns: {
      type: Array,
      required: true,
    },
    data: {
      type: Array,
      required: true,
    },
  },
};
</script>
```

```vue
<!-- UserListPage.vue -->
<template>
  <data-table :columns="columns" :data="users">
    <!-- status 컬럼만 커스텀 렌더링 -->
    <template #cell-status="{ value }">
      <span :class="value === 'active' ? 'green' : 'red'">{{ value }}</span>
    </template>
    <!-- actions 컬럼 커스텀 렌더링 -->
    <template #cell-actions="{ row }">
      <button @click="editUser(row)">편집</button>
      <button @click="deleteUser(row)">삭제</button>
    </template>
    <!-- 빈 상태 커스텀 -->
    <template #empty>
      <p>사용자가 없습니다. 새 사용자를 추가하세요.</p>
    </template>
  </data-table>
</template>

<script>
import DataTable from './DataTable.vue';

export default {
  name: 'UserListPage',
  components: { DataTable },
  data() {
    return {
      columns: [
        { key: 'name', label: '이름' },
        { key: 'email', label: '이메일' },
        { key: 'status', label: '상태' },
        { key: 'actions', label: '작업' },
      ],
      users: [],
    };
  },
  methods: {
    editUser(user) { /* 편집 로직 */ },
    deleteUser(user) { /* 삭제 로직 */ },
  },
};
</script>
```

**개선 효과**:
- `DataTable`이 데이터 구조에 독립적이어서 어떤 목록 화면에서도 재사용 가능
- 특정 컬럼만 커스텀 렌더링하고 나머지는 기본 텍스트 출력 활용
- 빈 상태 표시도 slot으로 커스터마이징 가능하여 페이지별 맞춤 메시지 제공
- 새로운 컬럼이나 액션 추가 시 `DataTable` 수정 없이 사용 측에서만 변경

---

## 3. 반응성 함정 (Reactivity Pitfalls)

### 검토 항목

1. 객체에 새 속성을 추가할 때 `Vue.set` 또는 `this.$set`을 사용하는가?
2. 배열 요소를 인덱스로 직접 변경하지 않는가? (`splice` 또는 `Vue.set` 사용)
3. `$forceUpdate`를 사용하고 있다면 반응성 시스템의 올바른 사용으로 대체 가능한가?
4. 중첩 객체의 속성 변경이 반응성 시스템에 의해 감지되는가?
5. `data()`에서 반환하지 않은 속성을 나중에 추가하고 있지 않은가?

### 개선 패턴

#### 예시 1: Vue.set으로 객체 속성 추가

**Before** (나쁜 예):
```vue
<template>
  <div class="user-profile">
    <h2>{{ user.name }}</h2>
    <p v-if="user.address">주소: {{ user.address }}</p>
    <p v-if="user.phone">전화: {{ user.phone }}</p>
    <button @click="addAddress">주소 추가</button>
    <button @click="addPhone">전화번호 추가</button>
    <hr />
    <h3>설정</h3>
    <div v-for="(value, key) in user.settings" :key="key">
      {{ key }}: {{ value }}
    </div>
    <button @click="addSetting">알림 설정 추가</button>
  </div>
</template>

<script>
export default {
  name: 'UserProfile',
  data() {
    return {
      user: {
        name: '홍길동',
        settings: {
          theme: 'dark',
        },
      },
    };
  },
  methods: {
    addAddress() {
      // 직접 속성 추가 — Vue가 변경을 감지하지 못함
      this.user.address = '서울시 강남구';
    },
    addPhone() {
      // 이것도 마찬가지 — 화면이 갱신되지 않음
      this.user.phone = '010-1234-5678';
    },
    addSetting() {
      // 중첩 객체에도 동일한 문제
      this.user.settings.notifications = true;
    },
  },
};
</script>
```

**문제점**:
- Vue 2의 반응성 시스템은 `Object.defineProperty` 기반이라 새 속성 추가를 감지하지 못함
- `this.user.address = '...'`를 실행해도 화면이 갱신되지 않음
- 다른 반응성 변경이 일어날 때 우연히 갱신될 수 있어 버그 재현이 어려움
- 개발자가 "새로고침하면 잘 되는데..."라고 오해하게 만드는 대표적인 함정

**After** (좋은 예):
```vue
<template>
  <div class="user-profile">
    <h2>{{ user.name }}</h2>
    <p v-if="user.address">주소: {{ user.address }}</p>
    <p v-if="user.phone">전화: {{ user.phone }}</p>
    <button @click="addAddress">주소 추가</button>
    <button @click="addPhone">전화번호 추가</button>
    <hr />
    <h3>설정</h3>
    <div v-for="(value, key) in user.settings" :key="key">
      {{ key }}: {{ value }}
    </div>
    <button @click="addSetting">알림 설정 추가</button>
  </div>
</template>

<script>
export default {
  name: 'UserProfile',
  data() {
    return {
      user: {
        name: '홍길동',
        // 나중에 추가될 속성을 미리 초기값으로 선언 (권장)
        address: '',
        phone: '',
        settings: {
          theme: 'dark',
        },
      },
    };
  },
  methods: {
    addAddress() {
      // 이미 선언된 속성이므로 직접 할당해도 반응성 유지
      this.user.address = '서울시 강남구';
    },
    addPhone() {
      this.user.phone = '010-1234-5678';
    },
    addSetting() {
      // 동적으로 추가해야 하는 속성은 반드시 $set 사용
      this.$set(this.user.settings, 'notifications', true);
    },
  },
};
</script>
```

**개선 효과**:
- 미리 선언된 속성(`address`, `phone`)은 초기값이 있어 반응성이 보장됨
- 동적 속성은 `this.$set`을 사용하여 Vue의 반응성 시스템에 올바르게 등록
- 화면이 데이터 변경에 즉시 반응하여 사용자 경험 일관성 확보
- 반응성 관련 디버깅 시간을 대폭 절약할 수 있음

#### 예시 2: 배열 인덱스 직접 변경을 splice/Vue.set으로 대체

**Before** (나쁜 예):
```vue
<template>
  <div class="todo-list">
    <ul>
      <li v-for="(todo, index) in todos" :key="index">
        <span :class="{ done: todo.done }">{{ todo.text }}</span>
        <button @click="toggleDone(index)">완료</button>
        <button @click="updateText(index)">수정</button>
        <button @click="removeItem(index)">삭제</button>
      </li>
    </ul>
    <p>완료: {{ doneCount }} / {{ todos.length }}</p>
  </div>
</template>

<script>
export default {
  name: 'TodoList',
  data() {
    return {
      todos: [
        { text: '장보기', done: false },
        { text: '운동하기', done: false },
        { text: '공부하기', done: false },
      ],
    };
  },
  computed: {
    doneCount() {
      return this.todos.filter((t) => t.done).length;
    },
  },
  methods: {
    toggleDone(index) {
      // 인덱스로 직접 교체 — Vue가 변경을 감지하지 못함
      this.todos[index] = {
        ...this.todos[index],
        done: !this.todos[index].done,
      };
    },
    updateText(index) {
      const newText = prompt('새 내용:', this.todos[index].text);
      if (newText) {
        // 배열 인덱스 직접 할당 — 화면 갱신 안 됨
        this.todos[index] = { ...this.todos[index], text: newText };
      }
    },
    removeItem(index) {
      // length 직접 변경 — 예측 불가능한 동작
      this.todos.length = this.todos.length - 1;
    },
  },
};
</script>
```

**문제점**:
- Vue 2는 `this.todos[index] = newValue` 형태의 배열 인덱스 직접 할당을 감지하지 못함
- `this.todos.length` 직접 변경도 반응성 시스템이 추적하지 못함
- `doneCount` computed 속성이 올바르게 재계산되지 않아 화면 불일치 발생
- 사용자가 버튼을 눌러도 화면이 변하지 않는 것처럼 보임

**After** (좋은 예):
```vue
<template>
  <div class="todo-list">
    <ul>
      <li v-for="(todo, index) in todos" :key="index">
        <span :class="{ done: todo.done }">{{ todo.text }}</span>
        <button @click="toggleDone(index)">완료</button>
        <button @click="updateText(index)">수정</button>
        <button @click="removeItem(index)">삭제</button>
      </li>
    </ul>
    <p>완료: {{ doneCount }} / {{ todos.length }}</p>
  </div>
</template>

<script>
export default {
  name: 'TodoList',
  data() {
    return {
      todos: [
        { text: '장보기', done: false },
        { text: '운동하기', done: false },
        { text: '공부하기', done: false },
      ],
    };
  },
  computed: {
    doneCount() {
      return this.todos.filter((t) => t.done).length;
    },
  },
  methods: {
    toggleDone(index) {
      // $set으로 배열 요소 교체 — 반응성 보장
      this.$set(this.todos, index, {
        ...this.todos[index],
        done: !this.todos[index].done,
      });
    },
    updateText(index) {
      const newText = prompt('새 내용:', this.todos[index].text);
      if (newText) {
        // splice도 Vue가 감지하는 변이 메서드
        this.todos.splice(index, 1, {
          ...this.todos[index],
          text: newText,
        });
      }
    },
    removeItem(index) {
      // splice로 요소 제거 — 반응성 보장
      this.todos.splice(index, 1);
    },
  },
};
</script>
```

**개선 효과**:
- `this.$set`과 `splice`를 사용하여 모든 배열 변경이 반응성 시스템에 감지됨
- `doneCount` computed 속성이 정확하게 재계산됨
- Vue 2에서 감지 가능한 배열 변이 메서드: `push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`
- 화면이 데이터 변경에 즉시 반응하여 예측 가능한 UI 동작 보장

#### 예시 3: $forceUpdate 제거 및 올바른 반응성 적용

**Before** (나쁜 예):
```vue
<template>
  <div class="dashboard">
    <div v-for="widget in widgets" :key="widget.id" class="widget">
      <h3>{{ widget.title }}</h3>
      <p>{{ widget.value }}</p>
      <span>마지막 갱신: {{ widget.lastUpdated }}</span>
    </div>
    <button @click="refreshAll">전체 새로고침</button>
  </div>
</template>

<script>
export default {
  name: 'Dashboard',
  data() {
    return {
      widgets: [],
    };
  },
  created() {
    this.widgets = [
      { id: 1, title: '방문자 수', value: 0 },
      { id: 2, title: '매출', value: 0 },
      { id: 3, title: '주문 수', value: 0 },
    ];
  },
  methods: {
    async refreshAll() {
      const data = await fetch('/api/dashboard').then((r) => r.json());

      // 반응성이 안 되니까 $forceUpdate로 강제 갱신
      this.widgets.forEach((widget, index) => {
        widget.value = data[index].value;
        // 새 속성을 반응성 없이 추가
        widget.lastUpdated = new Date().toLocaleTimeString();
      });

      // 화면이 안 바뀌니까 강제 업데이트
      this.$forceUpdate();
    },
  },
};
</script>
```

**문제점**:
- `$forceUpdate`는 반응성 시스템을 우회하여 전체 컴포넌트를 강제로 재렌더링함
- `lastUpdated`가 `data()`에 초기화되지 않아 반응성이 없어 `$forceUpdate` 없이는 화면에 표시되지 않음
- 자식 컴포넌트까지 불필요하게 재렌더링되어 성능 저하
- `$forceUpdate`가 필요한 상황은 거의 항상 반응성을 잘못 사용한 것을 의미함

**After** (좋은 예):
```vue
<template>
  <div class="dashboard">
    <div v-for="widget in widgets" :key="widget.id" class="widget">
      <h3>{{ widget.title }}</h3>
      <p>{{ widget.value }}</p>
      <span>마지막 갱신: {{ widget.lastUpdated }}</span>
    </div>
    <button @click="refreshAll">전체 새로고침</button>
  </div>
</template>

<script>
export default {
  name: 'Dashboard',
  data() {
    return {
      widgets: [
        // 모든 속성을 초기값과 함께 선언 — 반응성 보장
        { id: 1, title: '방문자 수', value: 0, lastUpdated: '' },
        { id: 2, title: '매출', value: 0, lastUpdated: '' },
        { id: 3, title: '주문 수', value: 0, lastUpdated: '' },
      ],
    };
  },
  methods: {
    async refreshAll() {
      const data = await fetch('/api/dashboard').then((r) => r.json());
      const now = new Date().toLocaleTimeString();

      // 새 배열을 할당하여 반응성을 자연스럽게 트리거
      this.widgets = this.widgets.map((widget, index) => ({
        ...widget,
        value: data[index].value,
        lastUpdated: now,
      }));
    },
  },
};
</script>
```

**개선 효과**:
- `$forceUpdate` 없이도 화면이 올바르게 갱신됨
- 모든 속성이 `data()`에서 초기화되어 반응성이 보장됨
- `map`으로 새 배열을 생성하여 Vue가 변경을 자연스럽게 감지
- 불필요한 전체 재렌더링이 제거되어 성능 향상

---

## 4. Mixin과 대안 패턴 (Mixins & Alternatives)

### 검토 항목

1. Mixin에 정의된 `data`, `methods`, `computed`가 컴포넌트와 이름 충돌을 일으키지 않는가?
2. Mixin 체인이 2단계 이상 깊어지지 않는가? (mixin이 다른 mixin을 포함)
3. Mixin 대신 유틸 함수, provide/inject, 또는 renderless 컴포넌트로 대체 가능한가?
4. Mixin의 암묵적 의존성(mixin이 사용하는 `this.xxx`)이 문서화되어 있는가?
5. 전역 mixin(`Vue.mixin`)을 사용하고 있지 않은가?

### 개선 패턴

#### 예시 1: mixin 충돌을 유틸 함수로 해결

**Before** (나쁜 예):
```vue
<!-- mixins/formMixin.js -->
<script>
export default {
  data() {
    return {
      loading: false,
      errors: {},
    };
  },
  methods: {
    validate() {
      this.errors = {};
      // 폼 필드 유효성 검증 로직
      if (!this.formData.name) {
        this.errors.name = '이름은 필수입니다';
      }
      return Object.keys(this.errors).length === 0;
    },
    async submit() {
      if (!this.validate()) return;
      this.loading = true;
      try {
        await this.onSubmit(this.formData);
      } catch (err) {
        this.errors.general = err.message;
      } finally {
        this.loading = false;
      }
    },
    reset() {
      this.errors = {};
      this.formData = {};
    },
  },
};
</script>
```

```vue
<!-- mixins/paginationMixin.js -->
<script>
export default {
  data() {
    return {
      // formMixin과 이름 충돌!
      loading: false,
      currentPage: 1,
      totalPages: 0,
    };
  },
  methods: {
    // formMixin.reset()과 이름 충돌!
    reset() {
      this.currentPage = 1;
      this.fetchPage();
    },
    async fetchPage() {
      this.loading = true;
      try {
        const res = await fetch(`/api/items?page=${this.currentPage}`);
        const data = await res.json();
        this.items = data.items;
        this.totalPages = data.totalPages;
      } finally {
        this.loading = false;
      }
    },
  },
};
</script>
```

```vue
<!-- UserManagement.vue -->
<script>
import formMixin from '../mixins/formMixin';
import paginationMixin from '../mixins/paginationMixin';

// 두 mixin을 함께 사용하면 loading과 reset이 충돌
export default {
  name: 'UserManagement',
  mixins: [formMixin, paginationMixin],
  data() {
    return {
      formData: { name: '', email: '' },
    };
  },
};
</script>
```

**문제점**:
- `loading`과 `reset`이 두 mixin에 모두 존재하여 나중에 선언된 mixin이 이전 것을 덮어씀
- `paginationMixin.reset()`이 `formMixin.reset()`을 완전히 대체하여 폼 초기화가 불가능
- Mixin이 어떤 `data`나 `methods`를 추가하는지 컴포넌트 코드만 봐서는 알 수 없음
- 디버깅 시 어떤 mixin에서 온 속성인지 추적이 매우 어려움

**After** (좋은 예):
```js
// utils/formHelper.js
// 순수 유틸 함수로 분리 — 이름 충돌 없음
export function createFormState(initialData = {}) {
  return {
    formData: { ...initialData },
    formErrors: {},
    formLoading: false,
  };
}

export function validateRequired(formData, requiredFields) {
  const errors = {};
  requiredFields.forEach((field) => {
    if (!formData[field]) {
      errors[field] = `${field}은(는) 필수입니다`;
    }
  });
  return errors;
}

export async function submitForm(formData, submitFn) {
  try {
    await submitFn(formData);
    return { success: true, error: null };
  } catch (err) {
    return { success: false, error: err.message };
  }
}
```

```js
// utils/paginationHelper.js
export function createPaginationState() {
  return {
    currentPage: 1,
    totalPages: 0,
    pageLoading: false,
  };
}

export async function fetchPage(url, page) {
  const res = await fetch(`${url}?page=${page}`);
  return res.json();
}
```

```vue
<!-- UserManagement.vue -->
<script>
import { createFormState, validateRequired, submitForm } from '../utils/formHelper';
import { createPaginationState, fetchPage } from '../utils/paginationHelper';

export default {
  name: 'UserManagement',
  data() {
    return {
      // 각 유틸의 상태가 명시적으로 선언됨 — 충돌 불가
      ...createFormState({ name: '', email: '' }),
      ...createPaginationState(),
      users: [],
    };
  },
  methods: {
    async handleSubmit() {
      const errors = validateRequired(this.formData, ['name', 'email']);
      if (Object.keys(errors).length > 0) {
        this.formErrors = errors;
        return;
      }
      this.formLoading = true;
      const result = await submitForm(this.formData, (data) =>
        fetch('/api/users', { method: 'POST', body: JSON.stringify(data) })
      );
      this.formLoading = false;
      if (result.success) this.resetForm();
    },
    resetForm() {
      this.formData = { name: '', email: '' };
      this.formErrors = {};
    },
    resetPagination() {
      this.currentPage = 1;
      this.loadUsers();
    },
    async loadUsers() {
      this.pageLoading = true;
      const data = await fetchPage('/api/users', this.currentPage);
      this.users = data.items;
      this.totalPages = data.totalPages;
      this.pageLoading = false;
    },
  },
};
</script>
```

**개선 효과**:
- `formLoading`과 `pageLoading`으로 이름이 분리되어 충돌 가능성 제거
- 각 유틸 함수의 입출력이 명확하여 테스트가 쉬움
- 컴포넌트 코드만 봐도 어떤 상태와 메서드가 존재하는지 한눈에 파악 가능
- `import` 문으로 의존성이 명시적이어서 IDE 지원(자동완성, 타입 추론)이 원활

#### 예시 2: 깊은 mixin 체인을 provide/inject로 대체

**Before** (나쁜 예):
```js
// mixins/baseMixin.js
export default {
  data() {
    return { appName: 'MyApp', theme: 'light' };
  },
  methods: {
    log(message) {
      console.log(`[${this.appName}] ${message}`);
    },
  },
};
```

```js
// mixins/authMixin.js — baseMixin에 의존
import baseMixin from './baseMixin';
export default {
  mixins: [baseMixin],
  data() {
    return { currentUser: null };
  },
  methods: {
    isAuthenticated() {
      this.log('인증 상태 확인');
      return !!this.currentUser;
    },
  },
};
```

```js
// mixins/permissionMixin.js — authMixin에 의존 (3단 체인)
import authMixin from './authMixin';
export default {
  mixins: [authMixin],
  methods: {
    hasPermission(perm) {
      if (!this.isAuthenticated()) return false;
      this.log(`권한 확인: ${perm}`);
      return this.currentUser.permissions.includes(perm);
    },
  },
};
```

```vue
<!-- AdminPage.vue — permissionMixin만 사용하지만 3개의 mixin이 모두 적용됨 -->
<script>
import permissionMixin from '../mixins/permissionMixin';
export default {
  name: 'AdminPage',
  mixins: [permissionMixin],
};
</script>
```

**문제점**:
- `AdminPage`에 `permissionMixin`만 포함했지만 실제로는 3개의 mixin이 모두 병합됨
- `this.appName`, `this.theme`, `this.log` 등이 어디서 오는지 추적하기 매우 어려움
- 중간 mixin 하나를 수정하면 체인의 모든 하위 컴포넌트에 영향
- Vue DevTools에서도 mixin 출처를 구별하기 어려움

**After** (좋은 예):
```vue
<!-- App.vue — 최상위에서 provide로 공유 데이터 제공 -->
<script>
export default {
  name: 'App',
  provide() {
    return {
      appName: this.appName,
      theme: this.theme,
      logger: this.log,
      auth: {
        getUser: () => this.currentUser,
        isAuthenticated: () => !!this.currentUser,
      },
    };
  },
  data() {
    return {
      appName: 'MyApp',
      theme: 'light',
      currentUser: null,
    };
  },
  methods: {
    log(message) {
      console.log(`[${this.appName}] ${message}`);
    },
  },
};
</script>
```

```vue
<!-- AdminPage.vue — inject로 필요한 것만 명시적으로 주입받음 -->
<template>
  <div v-if="canAccess">
    <h1>관리자 페이지</h1>
    <!-- 관리자 콘텐츠 -->
  </div>
  <div v-else>
    <p>접근 권한이 없습니다.</p>
  </div>
</template>

<script>
export default {
  name: 'AdminPage',
  inject: ['auth', 'logger'],
  computed: {
    canAccess() {
      if (!this.auth.isAuthenticated()) return false;
      const user = this.auth.getUser();
      return user && user.permissions.includes('admin');
    },
  },
  mounted() {
    this.logger('관리자 페이지 접근');
  },
};
</script>
```

**개선 효과**:
- Mixin 체인이 완전히 제거되어 의존성 관계가 단순해짐
- `inject`에 명시된 항목만 사용 가능하여 암묵적 의존성 제거
- `App.vue`만 수정하면 모든 하위 컴포넌트에 자동 반영
- 컴포넌트별로 필요한 의존성만 `inject`하여 결합도 최소화

---

## 5. Vuex 상태 관리 (Vuex State Management)

### 검토 항목

1. Mutation 안에서 비동기 작업(API 호출 등)을 수행하고 있지 않은가?
2. 컴포넌트에서 `$store.state`를 직접 변경하고 있지 않은가? (mutation을 통해야 함)
3. UI 전용 로컬 상태(모달 열림, 입력값 등)가 Vuex에 저장되어 있지 않은가?
4. Action에서 mutation을 호출하지 않고 state를 직접 변경하는 곳은 없는가?
5. Vuex 모듈이 네임스페이스(`namespaced: true`)를 사용하고 있는가?

### 개선 패턴

#### 예시 1: mutation에서 비동기 로직 제거

**Before** (나쁜 예):
```js
// store/modules/products.js
// mutation 안에서 비동기 API 호출 — Vuex의 핵심 규칙 위반
export default {
  namespaced: true,
  state: {
    products: [],
    loading: false,
    error: null,
  },
  mutations: {
    // mutation은 동기적이어야 하는데 async를 사용
    async FETCH_PRODUCTS(state) {
      state.loading = true;
      state.error = null;
      try {
        // mutation 안에서 API 호출 — 상태 변경 추적 불가
        const response = await fetch('/api/products');
        const data = await response.json();
        state.products = data;
      } catch (err) {
        state.error = err.message;
      } finally {
        state.loading = false;
      }
    },
    async DELETE_PRODUCT(state, productId) {
      try {
        await fetch(`/api/products/${productId}`, { method: 'DELETE' });
        state.products = state.products.filter((p) => p.id !== productId);
      } catch (err) {
        state.error = err.message;
      }
    },
  },
};
```

**문제점**:
- Mutation은 반드시 동기적이어야 하는 Vuex의 핵심 규칙 위반
- Vue DevTools의 시간 여행 디버깅(Time-travel Debugging)이 작동하지 않음
- 상태 변경 시점을 정확히 추적할 수 없어 디버깅이 극도로 어려움
- mutation 로그에 비동기 완료 전 상태가 기록되어 실제 상태와 불일치

**After** (좋은 예):
```js
// store/modules/products.js
// action에서 비동기 처리, mutation은 동기적 상태 변경만 담당
export default {
  namespaced: true,
  state: {
    products: [],
    loading: false,
    error: null,
  },
  mutations: {
    SET_LOADING(state, isLoading) {
      state.loading = isLoading;
    },
    SET_PRODUCTS(state, products) {
      state.products = products;
    },
    REMOVE_PRODUCT(state, productId) {
      state.products = state.products.filter((p) => p.id !== productId);
    },
    SET_ERROR(state, error) {
      state.error = error;
    },
  },
  actions: {
    async fetchProducts({ commit }) {
      commit('SET_LOADING', true);
      commit('SET_ERROR', null);
      try {
        const response = await fetch('/api/products');
        const data = await response.json();
        commit('SET_PRODUCTS', data);
      } catch (err) {
        commit('SET_ERROR', err.message);
      } finally {
        commit('SET_LOADING', false);
      }
    },
    async deleteProduct({ commit }, productId) {
      try {
        await fetch(`/api/products/${productId}`, { method: 'DELETE' });
        commit('REMOVE_PRODUCT', productId);
      } catch (err) {
        commit('SET_ERROR', err.message);
      }
    },
  },
};
```

**개선 효과**:
- Mutation이 순수 동기 함수로 유지되어 Vue DevTools의 시간 여행 디버깅이 정상 작동
- 각 mutation이 단일 책임(하나의 상태 변경)을 가져 추적이 용이
- Action에서 비동기 흐름을 관리하여 mutation 호출 순서가 명확
- 에러 발생 시 어느 단계에서 실패했는지 정확히 파악 가능

#### 예시 2: 직접 state 변경을 mutation으로 대체

**Before** (나쁜 예):
```vue
<template>
  <div class="cart">
    <div v-for="item in cartItems" :key="item.id">
      <span>{{ item.name }} - {{ item.price }}원</span>
      <input
        type="number"
        :value="item.quantity"
        @input="updateQuantity(item, $event.target.value)"
      />
      <button @click="removeFromCart(item)">제거</button>
    </div>
    <p>총액: {{ totalPrice }}원</p>
  </div>
</template>

<script>
export default {
  name: 'ShoppingCart',
  computed: {
    cartItems() {
      return this.$store.state.cart.items;
    },
    totalPrice() {
      return this.cartItems.reduce((sum, item) => sum + item.price * item.quantity, 0);
    },
  },
  methods: {
    updateQuantity(item, qty) {
      // Vuex state를 직접 변경 — strict 모드에서 에러 발생
      item.quantity = parseInt(qty, 10);
    },
    removeFromCart(item) {
      // 배열에서 직접 제거 — mutation을 거치지 않음
      const index = this.$store.state.cart.items.indexOf(item);
      this.$store.state.cart.items.splice(index, 1);
    },
  },
};
</script>
```

**문제점**:
- Vuex의 `strict: true` 모드에서 "mutation 핸들러 밖에서 상태를 변경했습니다" 에러 발생
- 상태 변경이 DevTools에 기록되지 않아 디버깅 불가
- 어느 컴포넌트에서 상태를 변경했는지 추적할 수 없음
- 여러 컴포넌트에서 같은 상태를 직접 변경하면 예측 불가능한 동작 발생

**After** (좋은 예):
```js
// store/modules/cart.js
export default {
  namespaced: true,
  state: {
    items: [],
  },
  getters: {
    totalPrice(state) {
      return state.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
    },
  },
  mutations: {
    UPDATE_QUANTITY(state, { itemId, quantity }) {
      const item = state.items.find((i) => i.id === itemId);
      if (item) {
        item.quantity = Math.max(1, quantity);
      }
    },
    REMOVE_ITEM(state, itemId) {
      state.items = state.items.filter((i) => i.id !== itemId);
    },
  },
};
```

```vue
<template>
  <div class="cart">
    <div v-for="item in cartItems" :key="item.id">
      <span>{{ item.name }} - {{ item.price }}원</span>
      <input
        type="number"
        :value="item.quantity"
        @input="updateQuantity(item.id, $event.target.value)"
      />
      <button @click="removeFromCart(item.id)">제거</button>
    </div>
    <p>총액: {{ totalPrice }}원</p>
  </div>
</template>

<script>
import { mapState, mapGetters, mapMutations } from 'vuex';

export default {
  name: 'ShoppingCart',
  computed: {
    ...mapState('cart', { cartItems: 'items' }),
    ...mapGetters('cart', ['totalPrice']),
  },
  methods: {
    ...mapMutations('cart', {
      commitUpdateQuantity: 'UPDATE_QUANTITY',
      commitRemoveItem: 'REMOVE_ITEM',
    }),
    updateQuantity(itemId, qty) {
      this.commitUpdateQuantity({ itemId, quantity: parseInt(qty, 10) });
    },
    removeFromCart(itemId) {
      this.commitRemoveItem(itemId);
    },
  },
};
</script>
```

**개선 효과**:
- 모든 상태 변경이 mutation을 통해 이루어져 strict 모드에서도 정상 작동
- DevTools에서 모든 mutation 기록을 확인할 수 있어 디버깅 용이
- `mapState`, `mapGetters`, `mapMutations`로 Vuex 연동 코드가 깔끔해짐
- getter에서 `totalPrice`를 계산하여 여러 컴포넌트에서 일관된 값 사용 가능

#### 예시 3: UI 로컬 상태를 Vuex에서 분리

**Before** (나쁜 예):
```js
// store/modules/ui.js
// UI 전용 로컬 상태가 모두 Vuex에 저장됨
export default {
  namespaced: true,
  state: {
    isModalOpen: false,
    modalType: '',
    searchInput: '',
    isDropdownOpen: false,
    activeTab: 'general',
    formDraft: {
      name: '',
      email: '',
      message: '',
    },
    tooltipVisible: false,
    scrollPosition: 0,
  },
  mutations: {
    TOGGLE_MODAL(state, type) {
      state.isModalOpen = !state.isModalOpen;
      state.modalType = type || '';
    },
    SET_SEARCH_INPUT(state, value) {
      state.searchInput = value;
    },
    SET_DROPDOWN(state, isOpen) {
      state.isDropdownOpen = isOpen;
    },
    SET_ACTIVE_TAB(state, tab) {
      state.activeTab = tab;
    },
    UPDATE_FORM_DRAFT(state, { field, value }) {
      state.formDraft[field] = value;
    },
    SET_TOOLTIP(state, visible) {
      state.tooltipVisible = visible;
    },
    SET_SCROLL(state, position) {
      state.scrollPosition = position;
    },
  },
};
```

**문제점**:
- 툴팁 표시 여부, 드롭다운 상태 등 단일 컴포넌트에서만 사용하는 상태가 Vuex에 저장됨
- 모든 UI 상호작용이 mutation을 트리거하여 DevTools 로그가 불필요한 기록으로 넘침
- 컴포넌트가 언마운트되어도 Vuex에 상태가 남아 메모리 누수 가능성
- Vuex는 여러 컴포넌트가 공유하는 전역 상태를 위한 도구인데 로컬 상태까지 관리하면 복잡도만 증가

**After** (좋은 예):
```vue
<!-- ContactPage.vue -->
<template>
  <div>
    <!-- 로컬 상태: 이 컴포넌트에서만 사용 -->
    <input v-model="searchInput" placeholder="검색" />
    <div class="tabs">
      <button
        v-for="tab in tabs"
        :key="tab.key"
        :class="{ active: activeTab === tab.key }"
        @click="activeTab = tab.key"
      >
        {{ tab.label }}
      </button>
    </div>

    <form @submit.prevent="submitForm">
      <input v-model="form.name" placeholder="이름" />
      <input v-model="form.email" placeholder="이메일" />
      <textarea v-model="form.message" placeholder="메시지"></textarea>
      <button type="submit">전송</button>
    </form>

    <!-- 전역 상태: Vuex에서 가져옴 -->
    <p>로그인 사용자: {{ currentUser.name }}</p>
    <notification-list :notifications="notifications" />
  </div>
</template>

<script>
import { mapState, mapActions } from 'vuex';

export default {
  name: 'ContactPage',
  data() {
    return {
      // UI 로컬 상태 — 이 컴포넌트에서만 사용
      searchInput: '',
      activeTab: 'general',
      form: {
        name: '',
        email: '',
        message: '',
      },
      tabs: [
        { key: 'general', label: '일반' },
        { key: 'support', label: '기술 지원' },
        { key: 'sales', label: '영업' },
      ],
    };
  },
  computed: {
    // 전역 상태 — 여러 컴포넌트에서 공유
    ...mapState('auth', ['currentUser']),
    ...mapState('notification', ['notifications']),
  },
  methods: {
    ...mapActions('contact', ['sendMessage']),
    async submitForm() {
      // 전역 상태 변경만 Vuex action 사용
      await this.sendMessage(this.form);
      this.form = { name: '', email: '', message: '' };
    },
  },
};
</script>
```

**개선 효과**:
- UI 로컬 상태(`searchInput`, `activeTab`, `form`)가 컴포넌트 `data`로 이동하여 Vuex 부담 감소
- DevTools mutation 로그에 의미 있는 비즈니스 로직 변경만 기록됨
- 컴포넌트가 언마운트되면 로컬 상태도 자동으로 정리됨
- Vuex에는 인증 정보, 알림 등 실제 전역 공유가 필요한 상태만 남김

---

## 6. 템플릿 모범 사례 (Template Best Practices)

### 검토 항목

1. `v-if`와 `v-for`가 같은 요소에 함께 사용되고 있지 않은가?
2. `v-for`에 고유한 `:key`가 바인딩되어 있는가? (인덱스가 아닌 고유 ID)
3. `v-html`에 사용자 입력이 직접 바인딩되어 있지 않은가? (XSS 위험)
4. 템플릿 표현식이 단순한가? (복잡한 로직은 computed로 이동)
5. 이벤트 핸들러에 인라인 복잡 로직이 포함되어 있지 않은가?

### 개선 패턴

#### 예시 1: v-if와 v-for 분리

**Before** (나쁜 예):
```vue
<template>
  <div class="user-list">
    <!-- v-if와 v-for가 같은 요소에 사용됨 -->
    <div
      v-for="user in users"
      v-if="user.isActive"
      :key="user.id"
      class="user-card"
    >
      <h3>{{ user.name }}</h3>
      <p>{{ user.email }}</p>
      <span class="role">{{ user.role }}</span>
    </div>

    <!-- 중첩 반복에서도 같은 문제 -->
    <div v-for="group in groups" :key="group.id">
      <h4>{{ group.name }}</h4>
      <span
        v-for="member in group.members"
        v-if="member.status !== 'banned'"
        :key="member.id"
      >
        {{ member.name }}
      </span>
    </div>
  </div>
</template>

<script>
export default {
  name: 'UserList',
  data() {
    return {
      users: [],
      groups: [],
    };
  },
};
</script>
```

**문제점**:
- Vue 2에서 `v-for`는 `v-if`보다 우선순위가 높아 모든 항목을 먼저 순회한 뒤 조건을 검사함
- 1000명 중 100명만 활성 상태여도 매 렌더링마다 1000번 조건을 평가함
- ESLint `vue/no-use-v-if-with-v-for` 규칙 위반
- 의도와 다르게 비활성 사용자도 렌더 함수에서 처리되어 불필요한 VNode 생성

**After** (좋은 예):
```vue
<template>
  <div class="user-list">
    <!-- computed로 필터링한 결과만 렌더링 -->
    <div
      v-for="user in activeUsers"
      :key="user.id"
      class="user-card"
    >
      <h3>{{ user.name }}</h3>
      <p>{{ user.email }}</p>
      <span class="role">{{ user.role }}</span>
    </div>

    <!-- 중첩 구조: template 태그로 v-for와 v-if 분리 -->
    <div v-for="group in groups" :key="group.id">
      <h4>{{ group.name }}</h4>
      <template v-for="member in group.members">
        <span
          v-if="member.status !== 'banned'"
          :key="member.id"
        >
          {{ member.name }}
        </span>
      </template>
    </div>
  </div>
</template>

<script>
export default {
  name: 'UserList',
  data() {
    return {
      users: [],
      groups: [],
    };
  },
  computed: {
    activeUsers() {
      return this.users.filter((user) => user.isActive);
    },
  },
};
</script>
```

**개선 효과**:
- `computed`로 필터링하면 `users`가 변경될 때만 재계산되어 렌더링 비용 절감
- 활성 사용자만 VNode이 생성되어 DOM 업데이트 범위 최소화
- `<template>` 태그를 사용하면 불필요한 래퍼 DOM 요소 없이 `v-for`와 `v-if`를 분리 가능
- ESLint 규칙을 준수하여 팀 전체의 코드 일관성 향상

#### 예시 2: 고유 key 사용

**Before** (나쁜 예):
```vue
<template>
  <div class="task-board">
    <!-- key로 인덱스를 사용 — 정렬/삭제 시 버그 발생 -->
    <div
      v-for="(task, index) in tasks"
      :key="index"
      class="task-card"
    >
      <input v-model="task.title" />
      <select v-model="task.priority">
        <option value="low">낮음</option>
        <option value="medium">보통</option>
        <option value="high">높음</option>
      </select>
      <textarea v-model="task.description"></textarea>
      <button @click="removeTask(index)">삭제</button>
    </div>
    <button @click="addTask">작업 추가</button>
    <button @click="sortByPriority">우선순위 정렬</button>
  </div>
</template>

<script>
export default {
  name: 'TaskBoard',
  data() {
    return {
      tasks: [
        { title: '디자인 검토', priority: 'high', description: '' },
        { title: 'API 연동', priority: 'medium', description: '' },
        { title: '문서 작성', priority: 'low', description: '' },
      ],
    };
  },
  methods: {
    addTask() {
      this.tasks.push({ title: '', priority: 'medium', description: '' });
    },
    removeTask(index) {
      this.tasks.splice(index, 1);
    },
    sortByPriority() {
      const order = { high: 0, medium: 1, low: 2 };
      this.tasks.sort((a, b) => order[a.priority] - order[b.priority]);
    },
  },
};
</script>
```

**문제점**:
- 인덱스를 key로 사용하면 항목 삭제/정렬 시 Vue의 DOM 재사용 로직이 잘못 작동
- 첫 번째 항목 삭제 시 나머지 항목의 인덱스가 변경되어 input 상태가 뒤섞임
- 정렬 후 input에 입력 중인 텍스트가 다른 항목으로 이동하는 버그 발생
- `v-model`로 바인딩된 폼 요소가 있을 때 특히 문제가 심각해짐

**After** (좋은 예):
```vue
<template>
  <div class="task-board">
    <!-- 고유 ID를 key로 사용 — 정렬/삭제 시에도 안전 -->
    <div
      v-for="task in tasks"
      :key="task.id"
      class="task-card"
    >
      <input v-model="task.title" />
      <select v-model="task.priority">
        <option value="low">낮음</option>
        <option value="medium">보통</option>
        <option value="high">높음</option>
      </select>
      <textarea v-model="task.description"></textarea>
      <button @click="removeTask(task.id)">삭제</button>
    </div>
    <button @click="addTask">작업 추가</button>
    <button @click="sortByPriority">우선순위 정렬</button>
  </div>
</template>

<script>
let nextId = 1;

export default {
  name: 'TaskBoard',
  data() {
    return {
      tasks: [
        { id: nextId++, title: '디자인 검토', priority: 'high', description: '' },
        { id: nextId++, title: 'API 연동', priority: 'medium', description: '' },
        { id: nextId++, title: '문서 작성', priority: 'low', description: '' },
      ],
    };
  },
  methods: {
    addTask() {
      this.tasks.push({
        id: nextId++,
        title: '',
        priority: 'medium',
        description: '',
      });
    },
    removeTask(taskId) {
      this.tasks = this.tasks.filter((t) => t.id !== taskId);
    },
    sortByPriority() {
      const order = { high: 0, medium: 1, low: 2 };
      // 정렬은 새 배열로 — 원본 변이보다 예측 가능
      this.tasks = [...this.tasks].sort((a, b) => order[a.priority] - order[b.priority]);
    },
  },
};
</script>
```

**개선 효과**:
- 고유 `id`를 key로 사용하여 항목 삭제/정렬 시에도 DOM 요소와 데이터의 매핑이 유지됨
- input에 입력 중인 텍스트, 포커스 상태 등이 정렬 후에도 올바른 항목에 남아있음
- Vue의 가상 DOM diffing이 정확하게 작동하여 최소한의 DOM 조작만 수행
- 서버에서 받은 데이터의 ID를 그대로 사용하면 추가 ID 생성 로직도 불필요

#### 예시 3: v-html XSS 방지

**Before** (나쁜 예):
```vue
<template>
  <div class="comment-section">
    <div v-for="comment in comments" :key="comment.id" class="comment">
      <strong>{{ comment.author }}</strong>
      <!-- 사용자 입력을 v-html로 직접 렌더링 — XSS 공격에 취약 -->
      <div v-html="comment.content"></div>
    </div>

    <div class="editor">
      <textarea v-model="newComment" placeholder="댓글 작성..."></textarea>
      <!-- 미리보기도 v-html로 렌더링 -->
      <div class="preview" v-html="newComment"></div>
      <button @click="submitComment">등록</button>
    </div>
  </div>
</template>

<script>
export default {
  name: 'CommentSection',
  data() {
    return {
      newComment: '',
      comments: [],
    };
  },
  methods: {
    async submitComment() {
      const response = await fetch('/api/comments', {
        method: 'POST',
        body: JSON.stringify({ content: this.newComment }),
        headers: { 'Content-Type': 'application/json' },
      });
      const saved = await response.json();
      this.comments.push(saved);
      this.newComment = '';
    },
  },
};
</script>
```

**문제점**:
- 사용자가 `<script>alert('XSS')</script>` 또는 `<img onerror="..." src="">` 같은 악성 HTML을 입력할 수 있음
- 미리보기에서도 v-html을 사용하여 입력 즉시 스크립트가 실행될 수 있음
- 저장된 댓글을 다른 사용자가 볼 때 Stored XSS 공격이 가능
- 쿠키 탈취, 세션 하이재킹, 피싱 등 심각한 보안 위협

**After** (좋은 예):
```vue
<template>
  <div class="comment-section">
    <div v-for="comment in comments" :key="comment.id" class="comment">
      <strong>{{ comment.author }}</strong>
      <!-- 안전한 HTML만 렌더링: sanitize 처리 -->
      <div v-html="sanitize(comment.content)"></div>
    </div>

    <div class="editor">
      <textarea v-model="newComment" placeholder="댓글 작성..."></textarea>
      <!-- 미리보기도 sanitize 적용 -->
      <div class="preview" v-html="sanitize(newComment)"></div>
      <button @click="submitComment">등록</button>
    </div>
  </div>
</template>

<script>
import DOMPurify from 'dompurify';

export default {
  name: 'CommentSection',
  data() {
    return {
      newComment: '',
      comments: [],
    };
  },
  methods: {
    sanitize(html) {
      // 허용할 태그와 속성만 화이트리스트로 지정
      return DOMPurify.sanitize(html, {
        ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br', 'ul', 'ol', 'li', 'code'],
        ALLOWED_ATTR: ['href', 'target', 'rel'],
      });
    },
    async submitComment() {
      // 서버에 저장하기 전에도 sanitize 적용
      const sanitizedContent = this.sanitize(this.newComment);
      const response = await fetch('/api/comments', {
        method: 'POST',
        body: JSON.stringify({ content: sanitizedContent }),
        headers: { 'Content-Type': 'application/json' },
      });
      const saved = await response.json();
      this.comments.push(saved);
      this.newComment = '';
    },
  },
};
</script>
```

**개선 효과**:
- DOMPurify가 `<script>`, `onerror`, `onclick` 등 모든 위험한 HTML을 제거
- 화이트리스트 방식으로 허용된 태그(`b`, `i`, `a` 등)만 렌더링
- 클라이언트(미리보기)와 서버 전송 시 모두 sanitize하여 이중 방어
- `v-html`을 사용해야 하는 정당한 사유가 있을 때에도 안전하게 처리 가능

---

## 7. 성능 최적화 (Performance Optimization)

### 검토 항목

1. 변경되지 않는 대량 데이터에 `Object.freeze`가 적용되어 있는가?
2. 상태가 없는 프레젠테이셔널 컴포넌트에 `functional: true`가 사용되어 있는가?
3. 라우트 컴포넌트에 lazy loading(`() => import()`)이 적용되어 있는가?
4. 큰 리스트에 가상 스크롤(virtual scrolling)이 적용되어 있는가?
5. 불필요한 watcher가 `immediate: true`로 과도하게 실행되고 있지 않은가?

### 개선 패턴

#### 예시 1: Object.freeze로 정적 데이터 최적화

**Before** (나쁜 예):
```vue
<template>
  <div class="country-selector">
    <select v-model="selectedCountry">
      <option
        v-for="country in countries"
        :key="country.code"
        :value="country.code"
      >
        {{ country.name }} ({{ country.code }})
      </option>
    </select>
    <div class="chart">
      <div
        v-for="point in chartData"
        :key="point.x"
        class="bar"
        :style="{ height: point.y + 'px' }"
      ></div>
    </div>
  </div>
</template>

<script>
// 대량 정적 데이터가 모두 반응성 객체로 변환됨
export default {
  name: 'CountrySelector',
  data() {
    return {
      selectedCountry: 'KR',
      // 250개 국가 데이터 — 절대 변경되지 않음
      countries: [
        { code: 'KR', name: '대한민국', continent: 'Asia', population: 51780000 },
        { code: 'US', name: '미국', continent: 'North America', population: 331900000 },
        { code: 'JP', name: '일본', continent: 'Asia', population: 125800000 },
        // ... 247개 더
      ],
      // 수천 개의 차트 데이터 포인트
      chartData: generateChartData(5000),
    };
  },
};

function generateChartData(count) {
  return Array.from({ length: count }, (_, i) => ({
    x: i,
    y: Math.random() * 100,
    label: `Point ${i}`,
  }));
}
</script>
```

**문제점**:
- Vue 2는 `data()`에 반환된 모든 객체를 재귀적으로 순회하며 `Object.defineProperty`로 getter/setter를 추가함
- 250개 국가 객체의 모든 속성(`code`, `name`, `continent`, `population`)에 getter/setter가 생성됨
- 5000개 차트 데이터 포인트까지 합하면 초기화 시 수만 개의 반응성 감시자가 생성됨
- 변경되지 않는 데이터에 반응성을 추가하는 것은 순수한 메모리 및 CPU 낭비

**After** (좋은 예):
```vue
<template>
  <div class="country-selector">
    <select v-model="selectedCountry">
      <option
        v-for="country in countries"
        :key="country.code"
        :value="country.code"
      >
        {{ country.name }} ({{ country.code }})
      </option>
    </select>
    <div class="chart">
      <div
        v-for="point in chartData"
        :key="point.x"
        class="bar"
        :style="{ height: point.y + 'px' }"
      ></div>
    </div>
  </div>
</template>

<script>
// 정적 데이터를 컴포넌트 밖에서 freeze — 반응성 변환 건너뜀
const COUNTRIES = Object.freeze([
  Object.freeze({ code: 'KR', name: '대한민국', continent: 'Asia', population: 51780000 }),
  Object.freeze({ code: 'US', name: '미국', continent: 'North America', population: 331900000 }),
  Object.freeze({ code: 'JP', name: '일본', continent: 'Asia', population: 125800000 }),
  // ... 247개 더
]);

function generateChartData(count) {
  const data = Array.from({ length: count }, (_, i) => ({
    x: i,
    y: Math.random() * 100,
    label: `Point ${i}`,
  }));
  return Object.freeze(data);
}

const CHART_DATA = generateChartData(5000);

export default {
  name: 'CountrySelector',
  data() {
    return {
      selectedCountry: 'KR',
    };
  },
  // freeze된 데이터는 computed나 created에서 반환하여 반응성 변환 회피
  computed: {
    countries() {
      return COUNTRIES;
    },
    chartData() {
      return CHART_DATA;
    },
  },
};
</script>
```

**개선 효과**:
- `Object.freeze`된 객체는 Vue 2가 반응성 변환을 건너뛰어 초기화 시간 대폭 단축
- 5000개 데이터 포인트 기준으로 초기화 시간이 수십 ms에서 수 ms로 감소
- 메모리 사용량도 getter/setter 제거로 상당히 감소
- `selectedCountry`만 반응성으로 관리되어 필요한 곳에만 리소스 할당

#### 예시 2: functional 컴포넌트 활용

**Before** (나쁜 예):
```vue
<!-- StaticBadge.vue -->
<template>
  <span class="badge" :class="variant">
    <i v-if="icon" :class="'icon-' + icon"></i>
    {{ label }}
  </span>
</template>

<script>
// 상태도, 라이프사이클도, watch도 없는 순수 프레젠테이셔널 컴포넌트
// 그런데 일반 컴포넌트로 선언되어 불필요한 인스턴스가 생성됨
export default {
  name: 'StaticBadge',
  props: {
    label: {
      type: String,
      required: true,
    },
    variant: {
      type: String,
      default: 'default',
    },
    icon: {
      type: String,
      default: '',
    },
  },
};
</script>
```

```vue
<!-- 부모 컴포넌트에서 수백 개의 Badge를 렌더링 -->
<template>
  <div class="tag-cloud">
    <!-- 500개의 StaticBadge 인스턴스가 생성됨 -->
    <static-badge
      v-for="tag in tags"
      :key="tag.id"
      :label="tag.name"
      :variant="tag.color"
      :icon="tag.icon"
    />
  </div>
</template>
```

**문제점**:
- `StaticBadge`는 `data`, `computed`, `watch`, 라이프사이클 훅이 없는 순수 출력 컴포넌트
- 500개 렌더링 시 500개의 Vue 인스턴스가 생성되어 메모리와 초기화 비용 발생
- 각 인스턴스에 반응성 시스템, 이벤트 시스템 등이 불필요하게 설정됨
- 목록이 커질수록 성능 차이가 선형적으로 증가

**After** (좋은 예):
```vue
<!-- StaticBadge.vue -->
<template functional>
  <span class="badge" :class="props.variant">
    <i v-if="props.icon" :class="'icon-' + props.icon"></i>
    {{ props.label }}
  </span>
</template>

<script>
// functional: true — 인스턴스 없이 렌더링, 훨씬 가벼움
export default {
  name: 'StaticBadge',
  functional: true,
  props: {
    label: {
      type: String,
      required: true,
    },
    variant: {
      type: String,
      default: 'default',
    },
    icon: {
      type: String,
      default: '',
    },
  },
};
</script>
```

```vue
<!-- 부모 컴포넌트 — 사용법은 동일 -->
<template>
  <div class="tag-cloud">
    <!-- 500개를 렌더링해도 인스턴스가 생성되지 않음 -->
    <static-badge
      v-for="tag in tags"
      :key="tag.id"
      :label="tag.name"
      :variant="tag.color"
      :icon="tag.icon"
    />
  </div>
</template>
```

**개선 효과**:
- `functional: true` 컴포넌트는 Vue 인스턴스를 생성하지 않아 메모리 사용량 대폭 감소
- 반응성 시스템, 라이프사이클 훅, 이벤트 리스너 등이 설정되지 않아 초기화 비용 절감
- 500개 렌더링 기준으로 일반 컴포넌트 대비 렌더링 속도 약 2~4배 향상
- `<template functional>` 구문으로 SFC에서도 쉽게 적용 가능 (props는 `props.xxx`로 접근)

#### 예시 3: 라우트 lazy loading

**Before** (나쁜 예):
```js
// router/index.js
// 모든 페이지 컴포넌트를 정적 import — 초기 번들에 모두 포함됨
import Vue from 'vue';
import VueRouter from 'vue-router';
import Home from '../views/Home.vue';
import About from '../views/About.vue';
import Dashboard from '../views/Dashboard.vue';
import UserList from '../views/UserList.vue';
import UserDetail from '../views/UserDetail.vue';
import Settings from '../views/Settings.vue';
import Reports from '../views/Reports.vue';
import AdminPanel from '../views/AdminPanel.vue';
import HelpCenter from '../views/HelpCenter.vue';
import NotFound from '../views/NotFound.vue';

Vue.use(VueRouter);

const routes = [
  { path: '/', component: Home },
  { path: '/about', component: About },
  { path: '/dashboard', component: Dashboard },
  { path: '/users', component: UserList },
  { path: '/users/:id', component: UserDetail },
  { path: '/settings', component: Settings },
  { path: '/reports', component: Reports },
  { path: '/admin', component: AdminPanel },
  { path: '/help', component: HelpCenter },
  { path: '*', component: NotFound },
];

export default new VueRouter({ routes, mode: 'history' });
```

**문제점**:
- 10개의 페이지 컴포넌트가 모두 초기 번들에 포함되어 첫 로딩 시간이 김
- 사용자가 `/admin` 페이지를 한 번도 방문하지 않아도 `AdminPanel` 코드가 다운로드됨
- 각 페이지가 사용하는 라이브러리(차트, 에디터 등)까지 초기 번들에 포함
- SPA가 커질수록 초기 로딩 시간이 선형적으로 증가

**After** (좋은 예):
```js
// router/index.js
// lazy loading으로 페이지별 코드 분할 — 필요할 때만 다운로드
import Vue from 'vue';
import VueRouter from 'vue-router';

Vue.use(VueRouter);

// 자주 방문하는 페이지만 정적 import (초기 번들 포함)
import Home from '../views/Home.vue';
import NotFound from '../views/NotFound.vue';

const routes = [
  // Home과 NotFound만 초기 번들에 포함
  { path: '/', component: Home },
  {
    path: '/about',
    component: () => import(/* webpackChunkName: "about" */ '../views/About.vue'),
  },
  {
    path: '/dashboard',
    component: () => import(/* webpackChunkName: "dashboard" */ '../views/Dashboard.vue'),
  },
  {
    path: '/users',
    component: () => import(/* webpackChunkName: "users" */ '../views/UserList.vue'),
  },
  {
    path: '/users/:id',
    // 같은 chunkName으로 그룹화 — 관련 페이지를 하나의 청크로 묶음
    component: () => import(/* webpackChunkName: "users" */ '../views/UserDetail.vue'),
  },
  {
    path: '/settings',
    component: () => import(/* webpackChunkName: "settings" */ '../views/Settings.vue'),
  },
  {
    path: '/reports',
    component: () => import(/* webpackChunkName: "reports" */ '../views/Reports.vue'),
  },
  {
    path: '/admin',
    // 관리자만 접근하므로 별도 청크로 분리
    component: () => import(/* webpackChunkName: "admin" */ '../views/AdminPanel.vue'),
    meta: { requiresAuth: true, role: 'admin' },
  },
  {
    path: '/help',
    component: () => import(/* webpackChunkName: "help" */ '../views/HelpCenter.vue'),
  },
  { path: '*', component: NotFound },
];

export default new VueRouter({ routes, mode: 'history' });
```

**개선 효과**:
- 초기 번들 크기가 대폭 감소하여 첫 페이지 로딩 시간 단축 (TTI 개선)
- 사용자가 실제로 방문하는 페이지의 코드만 다운로드
- `webpackChunkName`으로 관련 페이지를 하나의 청크로 그룹화 가능
- Vue CLI 프로젝트에서는 추가 설정 없이 동적 `import()`만으로 코드 분할 적용

---

## 8. 통합 체크리스트

### Options API 패턴
- [ ] `data`가 함수로 정의되어 각 인스턴스가 독립적인 상태를 가지는가?
- [ ] `watch`로 구현된 파생 상태가 `computed`로 대체 가능하지 않은가?
- [ ] Options 순서가 name → components → props → data → computed → watch → lifecycle → methods를 따르는가?
- [ ] `computed` 속성에 부수 효과(API 호출, DOM 조작)가 포함되어 있지 않은가?
- [ ] `methods`에서 화살표 함수 대신 일반 함수를 사용하여 `this` 바인딩이 올바른가?

### 컴포넌트 설계
- [ ] `props`가 배열이 아닌 객체 형태로 타입, 필수 여부, 기본값이 정의되어 있는가?
- [ ] 자식 컴포넌트가 `$parent`나 `$refs`를 통해 부모에 직접 접근하지 않는가?
- [ ] 재사용 컴포넌트에서 하드코딩된 렌더링 대신 scoped slot이 활용되어 있는가?
- [ ] 컴포넌트 이름이 2단어 이상이며 PascalCase 또는 kebab-case를 따르는가?
- [ ] `$emit` 이벤트 이름이 kebab-case를 따르는가?

### 반응성 함정
- [ ] 객체에 새 속성을 추가할 때 `Vue.set` 또는 `this.$set`을 사용하는가?
- [ ] 배열 요소를 인덱스로 직접 변경하지 않고 `splice` 또는 `Vue.set`을 사용하는가?
- [ ] `$forceUpdate`가 사용되고 있다면 반응성을 올바르게 수정하여 제거 가능한가?
- [ ] `data()`에서 반환하지 않은 속성을 나중에 동적으로 추가하고 있지 않은가?
- [ ] 중첩 객체의 속성 변경이 반응성 시스템에 의해 감지되는가?

### Mixin과 대안 패턴
- [ ] Mixin 간에 `data`, `methods`, `computed` 이름 충돌이 없는가?
- [ ] Mixin 체인이 2단계 이상 깊어지지 않는가?
- [ ] 전역 mixin(`Vue.mixin`)을 사용하고 있지 않은가?
- [ ] Mixin 대신 유틸 함수, provide/inject, renderless 컴포넌트로 대체 가능한가?
- [ ] Mixin의 암묵적 의존성(`this.xxx`)이 문서화되어 있는가?

### Vuex 상태 관리
- [ ] Mutation 안에서 비동기 작업(API 호출, setTimeout 등)을 수행하고 있지 않은가?
- [ ] 컴포넌트에서 `$store.state`를 직접 변경하지 않고 mutation을 사용하는가?
- [ ] UI 전용 로컬 상태(모달, 입력값)가 Vuex 대신 컴포넌트 `data`에 있는가?
- [ ] Vuex 모듈에 `namespaced: true`가 설정되어 있는가?
- [ ] getter를 활용하여 파생 상태를 중복 계산하지 않는가?

### 템플릿 모범 사례
- [ ] `v-if`와 `v-for`가 같은 요소에 함께 사용되고 있지 않은가?
- [ ] `v-for`에 고유한 `:key`가 바인딩되어 있는가? (인덱스 사용 지양)
- [ ] `v-html`에 사용자 입력이 직접 바인딩되지 않고 sanitize 처리가 되어 있는가?
- [ ] 템플릿 표현식이 단순하며 복잡한 로직은 computed로 이동되어 있는가?
- [ ] 이벤트 핸들러에 인라인 복잡 로직이 포함되어 있지 않은가?

### 성능 최적화
- [ ] 변경되지 않는 대량 데이터에 `Object.freeze`가 적용되어 있는가?
- [ ] 상태 없는 프레젠테이셔널 컴포넌트에 `functional: true`가 사용되어 있는가?
- [ ] 라우트 컴포넌트에 `() => import()` lazy loading이 적용되어 있는가?
- [ ] 큰 리스트에 가상 스크롤(vue-virtual-scroller 등)이 적용되어 있는가?
- [ ] 불필요한 watcher나 `deep: true` 옵션이 과도하게 사용되고 있지 않은가?
- [ ] 이미지, 컴포넌트 등 무거운 리소스에 지연 로딩이 적용되어 있는가?

---

### Before/After 제공 가이드

모든 리뷰 제안은 다음 형식으로 제공합니다:

```markdown
**Before** (문제점):
[현재 코드]

**문제점**:
- 구체적인 문제 설명 (성능, 보안, 유지보수성 등)

**After** (개선안):
[개선된 코드]

**개선 효과**:
- 구체적인 수치 또는 정성적 개선 효과
```

---

**이 가이드를 활용하여 Vue 2.x Options API 프로젝트의 체계적인 코드 리뷰를 수행하세요!**
