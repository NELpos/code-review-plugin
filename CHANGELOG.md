# Code Review Plugin 작업 기록

## Phase 1: 기존 Skill → Plugin 전환 (v1.0.0)

### Step 1. 플러그인 디렉토리 구조 생성

`code-review-skill/`의 기존 파일들을 새로운 플러그인 구조로 이동.

```
code-review-skill/              →  code-review-plugin/skills/code-review/
├── SKILL.md                         ├── SKILL.md
├── references/                      ├── references/
│   ├── tidy-first.md                │   ├── tidy-first.md
│   └── modern-engineering.md        │   └── modern-engineering.md
├── scripts/                         ├── scripts/
│   ├── analyze-complexity.ts        │   ├── analyze-complexity.ts
│   └── tsconfig.json                │   └── tsconfig.json
└── assets/                          └── assets/
    └── review-template.md               └── review-template.md
```

### Step 2. plugin.json 생성

`.claude-plugin/plugin.json` — Claude Code 플러그인 매니페스트 작성.

```json
{
  "name": "code-review",
  "version": "1.0.0",
  "description": "Tidy First + Modern SE 원칙 기반 체계적 코드 리뷰 플러그인",
  "author": { "name": "nelpos" },
  "license": "MIT"
}
```

### Step 3. marketplace.json 생성

`.claude-plugin/marketplace.json` — 셀프 호스팅 마켓플레이스 설정.

### Step 4. SKILL.md 오류 수정

3곳 수정:

| 줄 | Before | After |
|----|--------|-------|
| 116 | `Python: references/language-guides/python.md` | `JavaScript: references/language-guides/javascript.md` |
| 295 | `scripts/analyze-complexity.py` | `scripts/analyze-complexity.ts` |
| 364 | `scripts/analyze-complexity.py` | `scripts/analyze-complexity.ts` |

### Step 5. TypeScript 가이드 작성 (신규)

`references/language-guides/typescript.md` — 1,561줄

| 섹션 | 내용 |
|------|------|
| 1. 타입 안전성 | `any` 제거, 타입 가드, Optional Chaining, 타입 단언 최소화 |
| 2. 제네릭 베스트 프랙티스 | 제약 조건, 유틸리티 타입, 제네릭 남용 주의 |
| 3. 인터페이스/타입 설계 | Discriminated Union, readonly, ISP, 타입 vs 인터페이스 |
| 4. 열거형과 상수 | `as const`, 리터럴 유니온 vs enum |
| 5. 에러 처리 | `unknown` catch, Result 패턴, 커스텀 에러 |
| 6. 모듈 구성 | Barrel export, 순환 의존성, 경로 별칭 |
| 7. 통합 체크리스트 | 우선순위 테이블 (High/Medium/Low) |

### Step 6. JavaScript 가이드 작성 (신규)

`references/language-guides/javascript.md` — 886줄

| 섹션 | 내용 |
|------|------|
| 1. 모던 ES 패턴 | const/let, 구조분해, Optional Chaining, Template Literals |
| 2. 비동기 처리 | async/await, Promise.all/allSettled, 에러 처리 |
| 3. 모듈 시스템 | ESM vs CJS, Named vs Default export, 동적 import |
| 4. 배열/객체 메서드 | 함수형 패턴, 불변성, reduce 남용 주의 |
| 5. 에러 처리 | 커스텀 Error, error cause 체이닝, 전역 핸들러 |
| 6. 통합 체크리스트 | 20개 검토 항목 우선순위 분류 |

### Step 7. React 가이드 작성 (신규)

`references/framework-guides/react.md` — 1,980줄

| 섹션 | 내용 |
|------|------|
| 1. 컴포넌트 패턴 | 크기 제한, SRP, Composition, Compound Component |
| 2. Hooks 규칙 | useEffect 의존성, 커스텀 Hook, useMemo/useCallback |
| 3. 상태 관리 | 파생 상태, 상태 위치, 서버/클라이언트 상태, useReducer |
| 4. 렌더링 최적화 | key prop, 인라인 객체, React.memo, children 패턴 |
| 5. 이벤트 처리 | Controlled/Uncontrolled, 디바운스, React 19 form actions |
| 6. 접근성 (a11y) | 시맨틱 HTML, ARIA, 키보드 네비게이션, aria-live |
| 7. 통합 체크리스트 | 27개 검토 항목 우선순위 분류 |

### Step 8. Next.js 가이드 작성 (신규)

`references/framework-guides/nextjs.md` — 1,630줄

| 섹션 | 내용 |
|------|------|
| 1. Server/Client 컴포넌트 | "use client" 경계 최적화, 데이터 패칭 위치 |
| 2. 데이터 페칭 | fetch 옵션, Promise.all 병렬 처리, Suspense 스트리밍 |
| 3. 라우팅/레이아웃 | Route Group, loading.tsx/error.tsx, generateStaticParams |
| 4. 캐싱 전략 | revalidate, revalidateTag, 특성별 캐시 전략 |
| 5. Server Actions | zod 검증, useActionState, Progressive Enhancement |
| 6. 성능 최적화 | next/image, next/font, next/dynamic, Metadata API |
| 7. 미들웨어 | JWT 검증, matcher, i18n, 헤더 포워딩 |
| 8. 통합 체크리스트 | 23개 검토 항목 + Pages→App Router 마이그레이션 표 |

### Step 9. README.md 재작성

플러그인 설치 방법으로 전면 교체:
- 마켓플레이스/로컬/직접복사 3가지 방법
- 디렉토리 구조 업데이트
- Python 가이드, priority-matrix.md 참조 제거

### Step 10. LICENSE 생성

MIT License (2026, nelpos).

### Step 11. GitHub 레포 생성 및 푸시

```bash
gh repo create code-review-plugin --public --source . --push
```

**결과**: https://github.com/NELpos/code-review-plugin

---

## Phase 2: 멀티 플랫폼 Agent Skills 표준 전환 (v1.1.0)

### 배경 조사

skills.sh와 Agent Skills 오픈 표준(agentskills.io)을 조사한 결과:

- **SKILL.md는 Anthropic이 만든 오픈 표준**으로, 38개+ 에이전트에서 동작
- **skills.sh**는 Vercel이 만든 패키지 매니저 (`npx skills add owner/repo`)
- Claude Code, OpenAI Codex, Google Antigravity 모두 동일한 SKILL.md 표준 사용

### Step 12. .claude-plugin/ 제거

플랫폼 특화 디렉토리 삭제:

```
삭제: .claude-plugin/plugin.json
삭제: .claude-plugin/marketplace.json
```

### Step 13. SKILL.md 프론트매터 표준화

```yaml
# Before (Claude Code 전용)
---
name: code-review
description: |
  ...한국어 트리거만...
---

# After (오픈 표준)
---
name: code-review
description: |
  ...한국어 + 영어 트리거...
license: MIT
metadata:
  author: nelpos
  version: "1.0.0"
---
```

### Step 14. README.md 멀티 플랫폼 재작성

| Before | After |
|--------|-------|
| Claude Code 전용 설치법 | 3개 플랫폼 설치법 |
| `/plugin marketplace add` | `npx skills add NELpos/code-review-plugin` |
| `.claude-plugin/` 구조 | `skills/code-review/` 표준 구조 |

지원 플랫폼 표 추가:

| 플랫폼 | 스킬 경로 | 호출 방식 |
|--------|----------|----------|
| Claude Code | `.claude/skills/code-review/` | `/code-review` |
| OpenAI Codex | `.codex/skills/code-review/` | `$code-review` |
| Antigravity | `.agent/skills/code-review/` | 자동 트리거 |
| Cursor | `.cursor/skills/code-review/` | 자동 트리거 |

### Step 15. GitHub 푸시

```bash
git commit -m "멀티 플랫폼 Agent Skills 표준으로 재구성"
git push origin main
```

---

## 최종 산출물

### 파일 목록 (12개)

```
code-review-plugin/
├── skills/code-review/
│   ├── SKILL.md                          # 369줄 (워크플로우)
│   ├── references/
│   │   ├── tidy-first.md                 # 834줄 (Tidy First 체크리스트)
│   │   ├── modern-engineering.md         # 946줄 (Modern SE 체크리스트)
│   │   ├── language-guides/
│   │   │   ├── typescript.md             # 1,561줄 (신규)
│   │   │   └── javascript.md             # 886줄 (신규)
│   │   └── framework-guides/
│   │       ├── react.md                  # 1,980줄 (신규)
│   │       └── nextjs.md                 # 1,630줄 (신규)
│   ├── scripts/
│   │   ├── analyze-complexity.ts         # 396줄
│   │   └── tsconfig.json
│   └── assets/
│       └── review-template.md            # 285줄
├── README.md                             # 210줄
├── LICENSE                               # MIT
└── CHANGELOG.md                          # 이 파일
```

### 통계

| 항목 | 수치 |
|------|------|
| 총 파일 수 | 13개 |
| 신규 작성 콘텐츠 | 6,057줄 (가이드 4개) |
| 전체 콘텐츠 | ~9,100줄 |
| GitHub 커밋 | 2개 |
| 지원 플랫폼 | 38개+ (Agent Skills 표준) |

### 다른 프로젝트에서 설치

```bash
npx skills add NELpos/code-review-plugin
```

---

## Phase 3: Skill 품질 고도화 + Sub-agent 위임 체계 + npx skills 배포 정비 (v1.2.0)

### Step 16. Skill 문서 구조 재정비 (Progressive Disclosure 최적화)

`skills/code-review/SKILL.md`를 실행 중심으로 재작성:

- Frontmatter를 `name`, `description` 중심으로 정리
- 본문에서 트리거 설명/장문 예시를 제거하고 실행 절차를 우선 배치
- MUST/SHOULD 규칙, 검증 루프, 산출물 규격을 명시

결과:
- `SKILL.md` 113줄 수준으로 경량화 후, sub-agent 정책 반영 버전 159줄로 정착

### Step 17. 리뷰 템플릿 경량화

`skills/code-review/assets/review-template.md`를 실행 템플릿으로 단순화:

- High/Medium/Low 우선순위 중심 출력
- 원칙별 관찰 섹션 정리
- `Sub-agent 실행 요약` 섹션 추가

### Step 18. 참조 문서 탐색성 개선

100줄 이상 참조 문서 상단에 목차 추가:

- `references/tidy-first.md`
- `references/modern-engineering.md`
- `references/language-guides/typescript.md`
- `references/language-guides/javascript.md`
- `references/language-guides/python.md`
- `references/framework-guides/react.md`
- `references/framework-guides/nextjs.md`

### Step 19. 개선 기준 문서화

신규 파일: `docs/improvment_skills.md`

포함 내용:
- description/body 역할 분리 원칙
- 500줄 이하 유지 원칙
- MUST/SHOULD 사용 원칙
- 검증 루프
- sub-agent 위임/통합 체크리스트

### Step 20. Sub-agent 위임 정책 정식 도입

`skills/code-review/SKILL.md`에 아래 섹션 추가:

- `Sub-agent 위임 정책`
- `Sub-agent 호출 패킷`
- `Sub-agent 결과 통합 규칙`

위임 기준 예시:
- 파일 수 8개 이상
- diff 400줄 이상
- 다중 언어/프레임워크
- 긴 로그/탐색으로 컨텍스트 소비가 큰 경우

### Step 21. Sub-agent 운영 플레이북 추가

신규 파일: `skills/code-review/references/sub-agent-playbook.md`

포함 내용:
- 역할 분리 (`code-reviewer`, `security-reviewer`, `performance-reviewer`, `framework-reviewer`)
- 호출 패킷 표준 (priority/file/line/principle/problem/fix/evidence)
- 통합 규칙
- 운영 주의사항
- BP 근거 출처 링크

### Step 22. Sub-agent 템플릿 자산 추가

신규 파일:

- `skills/code-review/assets/sub-agents/code-reviewer.md`
- `skills/code-review/assets/sub-agents/security-reviewer.md`
- `skills/code-review/assets/sub-agents/performance-reviewer.md`

### Step 23. Claude Code 로컬 에이전트 파일 추가

신규 파일:

- `.claude/agents/code-reviewer.md`
- `.claude/agents/security-reviewer.md`
- `.claude/agents/performance-reviewer.md`
- `.claude/agents/framework-reviewer.md`

### Step 24. npx skills 배포 후 Sub-agent 적용 경로 정비

신규 파일:

- `skills/code-review/scripts/install-claude-subagents.sh`

기능:
- 설치된 skill 내부의 `assets/sub-agents/*.md`를 `.claude/agents` 또는 `~/.claude/agents`로 복사
- 프로젝트 범위/글로벌 범위 설치 지원

### Step 25. README 배포 가이드 업데이트

`README.md` 개선:

- `npx skills add ... --skill code-review` 명시
- `npx skills` 설치 후 Claude sub-agent post-install 절차 추가
- 최신 디렉토리 구조 반영 (`sub-agent-playbook`, `assets/sub-agents`, `install-claude-subagents.sh`)

### Step 26. 배포 적합성 리뷰 완료

리뷰 결론:

- `npx skills` 기준 스킬 배포 구조(`skills/code-review/SKILL.md`)는 적합
- Claude sub-agent는 별도 경로를 사용하므로 post-install 절차 필요
- 문서/스크립트로 해당 간극을 보완해 사용자 적용 경로를 명확화
