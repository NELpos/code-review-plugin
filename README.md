# Code Review Skill

**Tidy First** + **Modern Software Engineering** 원칙 기반의 멀티 플랫폼 코드 리뷰 에이전트 스킬입니다.

[Agent Skills 오픈 표준](https://agentskills.io) (`SKILL.md`)을 따르며, 다양한 AI 코딩 에이전트에서 동작합니다.

## 지원 플랫폼

| 플랫폼 | 스킬 경로 | 호출 방식 |
|--------|----------|----------|
| **Claude Code** | `.claude/skills/code-review/` | `/code-review` 또는 자동 트리거 |
| **OpenAI Codex** | `.codex/skills/code-review/` | `$code-review` 또는 자동 트리거 |
| **Antigravity** | `.agent/skills/code-review/` | 자동 트리거 |
| **Cursor** | `.cursor/skills/code-review/` | 자동 트리거 |
| 기타 38개+ 에이전트 | [skills.sh](https://skills.sh) 참조 | 에이전트별 상이 |

## 설치

### 방법 1: skills.sh (권장)

[skills.sh](https://skills.sh)를 통해 시스템에 설치된 에이전트를 자동 감지하여 설치합니다:

```bash
# 모든 감지된 에이전트에 설치
npx skills add NELpos/code-review-plugin

# 필요한 스킬만 명시적으로 설치
npx skills add NELpos/code-review-plugin --skill code-review

# 특정 에이전트만 대상
npx skills add -a claude-code NELpos/code-review-plugin
npx skills add -a codex NELpos/code-review-plugin
npx skills add -a antigravity NELpos/code-review-plugin

# 글로벌 설치 (모든 프로젝트에서 사용)
npx skills add -g NELpos/code-review-plugin
```

### Claude Code sub-agent 적용 (선택)

`npx skills`는 스킬(`SKILL.md`) 중심으로 설치됩니다. Claude Code의 sub-agent는 별도 경로(`.claude/agents` 또는 `~/.claude/agents`)를 사용하므로 post-install을 1회 실행하세요.

```bash
# 프로젝트 범위 설치 (.claude/agents)
bash .claude/skills/code-review/scripts/install-claude-subagents.sh

# 글로벌 설치 (~/.claude/agents)
bash ~/.claude/skills/code-review/scripts/install-claude-subagents.sh --global
```

### 방법 2: 수동 설치

```bash
git clone https://github.com/NELpos/code-review-plugin.git /tmp/crp

# Claude Code
cp -r /tmp/crp/skills/code-review .claude/skills/code-review

# OpenAI Codex
cp -r /tmp/crp/skills/code-review .codex/skills/code-review

# Antigravity
cp -r /tmp/crp/skills/code-review .agent/skills/code-review
```

### 방법 3: 글로벌 수동 설치

```bash
git clone https://github.com/NELpos/code-review-plugin.git /tmp/crp

# Claude Code (글로벌)
cp -r /tmp/crp/skills/code-review ~/.claude/skills/code-review

# OpenAI Codex (글로벌)
cp -r /tmp/crp/skills/code-review ~/.codex/skills/code-review

# Antigravity (글로벌)
cp -r /tmp/crp/skills/code-review ~/.gemini/antigravity/skills/code-review
```

### 복잡도 분석 스크립트 (선택)

```bash
# TypeScript 필요
pnpm install -D ts-node typescript
# 또는
npm install -g ts-node typescript
```

## 개요

이 스킬은 9가지 원칙을 체계적으로 적용하여 코드를 검토합니다:

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

## 사용법

에이전트에서 다음과 같은 프롬프트를 입력하면 자동으로 트리거됩니다:

```
코드 리뷰해줘
이 코드 어때?
리팩토링 제안해줘
코드 품질 확인해줘
PR 리뷰해줘
review this code
suggest refactoring
```

### 사용 예시

#### 단일 파일 리뷰
```
"lib/utils.ts 코드 좀 봐줘"
→ Tidy First 4원칙 적용 → 우선순위별 리포트 생성
```

#### Pull Request 리뷰
```
"PR #123 머지 전에 검토해줘"
→ diff 분석 → 변경된 파일별 리뷰 → 통합 리포트
```

#### 프로젝트 전체 분석
```
"프로젝트 전체 코드 품질 어때?"
→ 소스 파일 스캔 → Top 10 이슈 리스트 → 품질 점수
```

### 복잡도 분석 스크립트

```bash
ts-node scripts/analyze-complexity.ts src/
ts-node scripts/analyze-complexity.ts src/ --threshold 15
```

## 디렉토리 구조

```
code-review-plugin/
├── skills/
│   └── code-review/
│       ├── SKILL.md                       # 에이전트가 읽는 핵심 워크플로우
│       ├── references/
│       │   ├── tidy-first.md              # Tidy First 4원칙 체크리스트
│       │   ├── modern-engineering.md      # Modern SE 5원칙 체크리스트
│       │   ├── language-guides/
│       │   │   ├── typescript.md          # TypeScript 특화 가이드
│       │   │   ├── javascript.md          # JavaScript 특화 가이드
│       │   │   └── python.md             # Python 특화 가이드
│       │   └── framework-guides/
│       │       ├── react.md               # React 특화 가이드
│       │       └── nextjs.md              # Next.js 특화 가이드
│       ├── scripts/
│       │   ├── analyze-complexity.ts      # 복잡도 분석 스크립트
│       │   ├── install-claude-subagents.sh # Claude sub-agent 설치 스크립트
│       │   └── tsconfig.json
│       └── assets/
│           ├── review-template.md         # 리뷰 리포트 템플릿
│           └── sub-agents/                # sub-agent 템플릿
│               ├── code-reviewer.md
│               ├── security-reviewer.md
│               └── performance-reviewer.md
├── README.md
└── LICENSE
```

## 지원 언어 및 프레임워크

| 언어/프레임워크 | 가이드 | 자동 감지 |
|-----------------|--------|-----------|
| TypeScript | `language-guides/typescript.md` | `tsconfig.json` |
| JavaScript | `language-guides/javascript.md` | `package.json` |
| Python | `language-guides/python.md` | `requirements.txt`, `setup.py`, `pyproject.toml` |
| React | `framework-guides/react.md` | `react` 의존성 |
| Next.js | `framework-guides/nextjs.md` | `next` 의존성 |

## 리뷰 결과 예시

```markdown
# 코드 리뷰 결과

## 요약
- 검토 파일: 3개
- 발견 사항: High 2개, Medium 5개, Low 1개
- Tidy First 준수율 75%, Modern SE 준수율 82%

## High Priority Issues

### 1. [UserService.ts:45] Low Coupling 미적용
**분류**: Tidy First - Low Coupling
**문제점**: UserService가 구체적인 Database 클래스에 직접 의존
**개선 제안**: 인터페이스를 통한 의존성 주입으로 전환
```

## 확장하기

### 새 언어/프레임워크 추가

```bash
# 언어 가이드 추가
touch skills/code-review/references/language-guides/go.md

# 프레임워크 가이드 추가
touch skills/code-review/references/framework-guides/vue.md
```

SKILL.md 수정 없이 자동으로 감지됩니다.

## 기술 사양

이 스킬은 [Agent Skills 오픈 표준](https://agentskills.io/specification)을 따릅니다:

- **포맷**: SKILL.md (YAML frontmatter + Markdown)
- **Progressive Disclosure**: 메타데이터 → 지시사항 → 리소스 순으로 로딩
- **호환성**: SKILL.md 표준을 지원하는 모든 AI 코딩 에이전트

## 라이선스

MIT License - [LICENSE](LICENSE) 참조
