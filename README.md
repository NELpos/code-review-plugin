# Code Review Plugin

**Tidy First**와 **Modern Software Engineering** 원칙을 기반으로 한 Claude Code 코드 리뷰 플러그인입니다.

## 개요

이 플러그인은 다음 원칙들을 체계적으로 적용하여 코드를 검토합니다:

### Tidy First 4가지 원칙
1. **Guard Clauses** - 조기 반환으로 중첩 제거
2. **Extract Function** - 긴 함수를 작은 함수로 분리
3. **High Cohesion** - 관련된 기능끼리 묶기
4. **Low Coupling** - 의존성 최소화

### Modern Software Engineering 5가지 원칙
1. **Modularity** - 독립적인 모듈로 구성
2. **Cohesion** - 관련된 기능끼리 응집
3. **Separation of Concerns** - 관심사의 분리
4. **Information Hiding** - 정보 은닉과 캡슐화
5. **Coupling** - 최소한의 의존성

## 설치

### 방법 1: 마켓플레이스에서 설치

```bash
# Claude Code 내에서:
/plugin marketplace add <이 레포의 URL>
/plugin install code-review@nelpos-plugins
```

### 방법 2: 로컬 설치

```bash
# 레포 클론
git clone <이 레포의 URL> code-review-plugin

# Claude Code에서 플러그인 디렉토리 지정
claude --plugin-dir ./code-review-plugin
```

### 방법 3: 직접 복사

```bash
# skills 디렉토리만 Claude Code에 복사
mkdir -p ~/.claude/skills
cp -r code-review-plugin/skills/code-review ~/.claude/skills/
```

### 복잡도 분석 스크립트 사용 (선택)

복잡도 분석 스크립트를 사용하려면 TypeScript가 필요합니다:

```bash
# 프로젝트에 이미 TypeScript가 있다면 바로 사용 가능
pnpm install -D ts-node typescript

# 또는 전역 설치
npm install -g ts-node typescript
```

## 사용법

### 기본 사용 (한국어 프롬프트)

Claude Code에서 다음과 같은 한국어 명령어를 사용하면 자동으로 스킬이 트리거됩니다:

```
코드 리뷰해줘
이 코드 어때?
리팩토링 제안해줘
코드 품질 확인해줘
결합도 낮춰줘
중첩된 if문 제거해줘
이 코드 문제점 찾아줘
PR 리뷰해줘
```

또는 명령어로:
```
/review
/code-review
/리뷰
```

### 사용 예시

#### 1. 단일 파일 리뷰
```
사용자: "lib/utils.ts 코드 좀 봐줘"
→ Tidy First 4가지 원칙 적용 → 우선순위별 한글 리포트 생성
```

#### 2. Pull Request 리뷰
```
사용자: "PR #123 머지 전에 검토해줘"
→ gh pr diff 실행 → 변경된 파일별 리뷰 → 통합 리포트 생성
```

#### 3. 프로젝트 전체 분석
```
사용자: "프로젝트 전체 코드 품질 어때?"
→ 소스 파일 스캔 → Top 10 이슈 리스트 → 전체 품질 점수
```

#### 4. 특정 원칙만 검토
```
사용자: "이 코드 결합도만 확인해줘"
→ Low Coupling 원칙만 적용하여 리뷰
```

### 복잡도 분석 스크립트 사용

```bash
# 단일 파일 분석
ts-node scripts/analyze-complexity.ts src/components/Calculator.tsx

# 디렉토리 전체 분석
ts-node scripts/analyze-complexity.ts src/

# 임계값 지정 (기본값: 10)
ts-node scripts/analyze-complexity.ts src/ --threshold 15
```

## 디렉토리 구조

```
code-review-plugin/
├── .claude-plugin/
│   ├── plugin.json                # 플러그인 매니페스트
│   └── marketplace.json           # 셀프 호스팅 마켓플레이스
├── skills/
│   └── code-review/
│       ├── SKILL.md               # 핵심 워크플로우 (Claude가 읽음)
│       ├── references/
│       │   ├── tidy-first.md      # Tidy First 4가지 원칙 체크리스트
│       │   ├── modern-engineering.md  # Modern SE 5가지 원칙 체크리스트
│       │   ├── language-guides/
│       │   │   ├── typescript.md  # TypeScript 특화 가이드
│       │   │   └── javascript.md  # JavaScript 특화 가이드
│       │   └── framework-guides/
│       │       ├── react.md       # React 특화 가이드
│       │       └── nextjs.md      # Next.js 특화 가이드
│       ├── scripts/
│       │   ├── analyze-complexity.ts  # TypeScript 복잡도 분석
│       │   └── tsconfig.json
│       └── assets/
│           └── review-template.md # 리뷰 리포트 템플릿
├── README.md
└── LICENSE
```

## 지원 언어 및 프레임워크

| 언어/프레임워크 | 가이드 | 자동 감지 |
|-----------------|--------|-----------|
| TypeScript | `language-guides/typescript.md` | `tsconfig.json` |
| JavaScript | `language-guides/javascript.md` | `package.json` |
| React | `framework-guides/react.md` | `react` 의존성 |
| Next.js | `framework-guides/nextjs.md` | `next` 의존성 |

## 리뷰 결과 예시

```markdown
# 코드 리뷰 결과

## 요약
- 검토 파일: 3개
- 발견 사항: High 2개, Medium 5개, Low 1개
- 전체 품질 점수: Tidy First 준수율 75%, Modern SE 준수율 82%

## High Priority Issues

### 1. [UserService.ts:45] Low Coupling 미적용
**분류**: Tidy First - Low Coupling
**문제점**: UserService가 구체적인 Database 클래스에 직접 의존
**개선 제안**: 인터페이스를 통한 의존성 주입으로 전환
```

## 확장하기

### 새 언어 추가

`skills/code-review/references/language-guides/` 아래에 파일을 추가하면 자동으로 감지됩니다:

```bash
# 예: Go 언어 가이드 추가
touch skills/code-review/references/language-guides/go.md
```

### 새 프레임워크 추가

`skills/code-review/references/framework-guides/` 아래에 파일을 추가합니다:

```bash
# 예: Vue 프레임워크 가이드 추가
touch skills/code-review/references/framework-guides/vue.md
```

SKILL.md는 수정할 필요 없이 자동으로 해당 언어/프레임워크를 감지합니다.

## 라이선스

MIT License - [LICENSE](LICENSE) 참조

---

**이 플러그인은 어떤 프로젝트에든 설치하여 체계적인 코드 리뷰를 수행할 수 있는 범용 도구입니다!**
