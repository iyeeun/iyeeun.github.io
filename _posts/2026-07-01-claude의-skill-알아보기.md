---
layout: post
title: Claude의 Skill 알아보기
date: 2026-07-01 21:48 +0900
categories: [살펴보기, AI]
tags: [AI활용, claude, claude code, skill]
---

Claude의 **Skill**은 특정 업무 방식, 절차, 도메인 지식을 **재사용** 가능하게 제공하는 **에이전트 확장 모듈**이다.  
`SKILL.md` 파일을 중심으로 정의하며, Claude가 필요할 때 자동으로 불러오거나 `/skill-name` 형태로 직접 실행할 수 있다.

## 탄생 배경

초기 AI를 사용할 때 매번 같은 지시를 반복하는 문제가 발생했다.  
프로젝트 규칙을 매번 설명해야 하거나 팀의 개발 방식이 AI에게 전달되지 않았고, CLAUDE.md가 길어지는 등 특정 작업 절차를 자동화하기가 어려웠다.

Anthropic은 이를 해결하기 위해 필요할 때만 로드되는 **전문 지식 패키지** 개념을 도입했다.  
Skill은 CLAUDE.md처럼 항상 컨텍스트를 차지하지 않고 관련 작업에서만 로드된다.

## 기본 동작 원리

### SKILL.md 구조

Skill은 `SKILL.md` 파일 하나로 정의된다. 기본 구조는 다음과 같다.

```md
---
name: skill-name
description: When to use this skill — Claude reads this to decide when to load
---

Skill의 실행 절차나 규칙을 여기에 작성한다.
```

frontmatter에는 `name`과 `description`을 필수로 작성해야 한다.  
특히 `description`은 Claude가 **언제 이 Skill을 사용할지** 판단하는 기준이 된다.

### 실행 방식

Skill은 두 가지 방식으로 실행된다.

1\. **수동 실행**: `/skill-name` 형태로 직접 호출한다.

```
/create-component UserAvatar
```

2\. **자동 실행**: Claude가 대화 맥락을 분석하여 `description`에 부합하는 상황이라고 판단하면 자동으로 불러온다.  
예를 들어 description이 `"Use when creating new React components"`라면, "Button 컴포넌트 만들어줘"라고 할 때 Claude가 자동으로 해당 Skill을 로드한다.

### 컨텍스트 관리

CLAUDE.md는 항상 컨텍스트에 포함되지만, Skill은 **필요할 때만 로드**된다.  
이 덕분에 많은 Skill을 정의해도 불필요한 컨텍스트 낭비 없이 관리할 수 있다.

## CLAUDE.md와 차이점

CLAUDE.md에는 항상 알아야 하는 정보가 들어간다면, 특정 상황에서 필요한 작업 방식은 Skill에 정의된다고 볼 수 있다.

| 구분      | 역할             | 로드 시점   |
| --------- | ---------------- | ----------- |
| CLAUDE.md | 프로젝트 헌법    | 항상        |
| Skill     | 전문 작업 매뉴얼 | 필요할 때만 |
| Tool      | 실제 실행 능력   | 호출 시     |

예를 들면 다음과 같이 사용할 수 있다.

**CLAUDE.md**

```
- React + TypeScript
- pnpm 사용
- feature 폴더 구조
- 테스트는 Vitest
```

**Skill** (`.claude/skills/create-component/SKILL.md`)

```
새 컴포넌트 생성 절차:
1. 기존 컴포넌트 분석
2. props interface 작성
3. Storybook 생성
4. 테스트 작성
5. accessibility 체크
```

## 종류

Skill은 크게 4종류로 나눌 수 있다.

### 1. Built-in Skill

Claude Code가 기본 제공하는 Skill이다.  
코드 분석, 디버깅, 대량 작업, 컨텍스트 관리 등을 위한 목적이다.

ex. `/help`, `/compact`, `/debug`, `/simplify`, `/batch`

### 2. Personal Skill

개인 환경에서만 사용하는 Skill이다.  
`~/.claude/skills/`에 저장한다. (Git 공유 대상이 아니다.)  
모든 프로젝트에서 사용 가능하며, 개인 개발 습관과 관련된 내용을 저장할 수 있다.

ex. `/react-review`: 불필요한 useEffect는 없는지, memoization이 필요한지, 렌더링 이슈가 없는지

### 3. Project Skill

`.claude/skills/`에 저장되는, 특정 저장소에 포함되는 Skill이다.  
Git으로 공유하여 팀 전체가 동일한 AI 워크플로우를 사용하도록 할 수 있다.

```
.claude/
└── skills/
    ├── create-component/
    │   └── SKILL.md
    ├── release/
    │   └── SKILL.md
    └── migration/
        └── SKILL.md
```

#### Personal Skill vs Project Skill 비교

| 항목     | Personal Skill     | Project Skill    |
| -------- | ------------------ | ---------------- |
| 위치     | `~/.claude/skills` | `.claude/skills` |
| 범위     | 모든 프로젝트      | 현재 프로젝트    |
| 공유     | 개인만 사용        | 팀 공유 가능     |
| 목적     | 개인 생산성        | 팀 개발 표준     |
| Git 관리 | 보통 X             | 보통 O           |

### 4. Plugin Skill

Plugin은 Claude Code에 설치할 수 있는 독립적인 확장 패키지다.  
Plugin Skill은 이 Plugin 안에 포함된 Skill로, 설치 한 번으로 모든 프로젝트에서 사용할 수 있다.  
Project Skill이 특정 저장소에 묶인 반면, Plugin Skill은 설치한 환경 어디서나 동작하기 때문에 조직 전체에 공통 워크플로우를 배포할 때 적합하다.

#### 구조

Plugin은 `plugin.yaml`과 `skills/` 디렉토리로 구성된다.

```
my-org-plugin/
├── plugin.yaml
└── skills/
    └── security-review/
        └── SKILL.md
```

```yaml
# plugin.yaml
name: my-org-plugin
version: 1.0.0
description: Organization-wide security and compliance tools
```

#### 설치 및 호출

```zsh
claude plugin install ./my-org-plugin   # 로컬 설치
claude plugin install github:org/plugin # GitHub에서 설치
```

Plugin Skill은 충돌 방지를 위해 `plugin-name:skill-name` 형태로 호출한다.

```
/my-org-plugin:security-review
```

#### Project Skill vs Plugin Skill 비교

| 항목 | Project Skill      | Plugin Skill              |
| ---- | ------------------ | ------------------------- |
| 위치 | `.claude/skills/`  | Plugin 패키지 내부        |
| 범위 | 현재 프로젝트      | 설치한 모든 프로젝트      |
| 공유 | Git으로 팀 공유    | 패키지로 조직 전체 배포   |
| 목적 | 프로젝트 개발 표준 | 조직 공통 정책/도구       |
| 호출 | `/skill-name`      | `/plugin-name:skill-name` |

보안 정책, 컴플라이언스 체크, 조직 공통 코드 리뷰 기준처럼 **여러 팀이 동일하게 써야 하는 Skill**에 적합하다.

## React 관련 Skill 예시

좋은 Skill은 Claude가 자주 놓치는 **프로젝트 특유의 워크플로우를 강제**하는 것이다.  
아래는 실무 React 프로젝트에서 유용한 Skill 예시다.

### 1. React Component Generator

컴포넌트를 만들 때마다 팀의 규칙을 빠뜨리지 않도록 절차를 자동화한다.

#### SKILL.md 예시

```md
---
name: create-component
description: Use when creating new React components, pages, or UI elements
---

When creating a component:

1. Check existing components in src/components for patterns to follow
2. Define TypeScript interface for props
3. Create the following files:
   - Component.tsx
   - Component.test.tsx
   - Component.stories.tsx

Rules:

- No default export
- Use CSS modules
- Include accessibility attributes (aria-\*, role)
- Follow existing naming conventions
```

#### 사용 방법

```zsh
/create-component UserAvatar

UserAvatar/
├── UserAvatar.tsx
├── UserAvatar.test.tsx
└── UserAvatar.stories.tsx
```

### 2. React Performance Review

성능 문제는 눈에 잘 안 띄는 경우가 많다.  
Skill로 검사 항목을 명시하면 Claude가 빠뜨리지 않고 확인한다.

#### SKILL.md 예시

```md
---
name: react-performance-review
description: Use when reviewing React components for performance issues or optimization
---

Review the given component for:

1. Unnecessary re-renders
   - Check if props cause re-render on every parent update
   - Look for React.memo opportunities

2. Hook misuse
   - useEffect with missing or incorrect dependencies
   - useCallback / useMemo overuse or underuse

3. Unstable references
   - Objects or arrays created inline in render
   - Functions re-created on every render

4. Expensive calculations that should be wrapped in useMemo

5. Component size
   - Large components that should be split
   - Bundle size impact of imports
```

#### 사용 방법

```
/react-performance-review src/components/Dashboard.tsx
```

### 3. Design System 준수

팀에서 Design System을 사용할 때, 커스텀 컴포넌트 대신 Design System 컴포넌트를 쓰도록 강제한다.

#### SKILL.md 예시

```md
---
name: use-design-system
description: Use when creating or reviewing UI to enforce Design System component usage
---

Always prefer Design System components over custom implementations:

| Element | Use                | Do NOT use       |
| ------- | ------------------ | ---------------- |
| Button  | @company/ui/Button | <button>, custom |
| Modal   | @company/ui/Modal  | custom modal     |
| Input   | @company/ui/Input  | raw <input>      |
| Icon    | @company/ui/Icon   | inline SVG       |

Rules:

- Do not create custom button, input, or modal components
- Do not use raw HTML elements for interactive components
- If a Design System component doesn't exist, flag it before implementing
```

새 UI를 작성하거나 기존 컴포넌트를 리뷰할 때 호출하면, Claude가 Design System 컴포넌트를 우선 사용하도록 동작한다.

### 4. Class -> Hooks 마이그레이션

레거시 Class Component를 Hooks 기반으로 전환할 때 빠뜨리기 쉬운 단계를 절차화한다.

#### SKILL.md 예시

```md
---
name: migrate-to-hooks
description: Use when migrating React class components to functional components with hooks
---

Migration procedure:

1. Analyze class component
   - List all state properties
   - List all lifecycle methods and event handlers

2. Map to hooks
   - this.state → useState
   - componentDidMount → useEffect (empty deps)
   - componentDidUpdate → useEffect (with deps)
   - componentWillUnmount → useEffect cleanup
   - shouldComponentUpdate → React.memo

3. Convert PropTypes → TypeScript interface

4. Verify test coverage is maintained after migration

5. Check for non-migratable patterns
   - getDerivedStateFromProps (careful handling needed)
   - Error boundaries (must remain as class component)
```

### 5. Accessibility Check

WCAG 기준을 체크하는 Skill이다.  
Claude는 기본적으로 접근성을 어느 정도 고려하지만, 체크 항목이 없으면 중요한 부분을 빠뜨리기 쉽다.

#### SKILL.md 예시

```md
---
name: accessibility-check
description: Use when reviewing UI components or pages for accessibility compliance
---

Check the given component for WCAG 2.1 AA compliance:

1. Keyboard navigation
   - All interactive elements reachable by Tab
   - Visible focus indicator present
   - Logical tab order

2. Screen reader support
   - Meaningful alt text on images (not "image" or filename)
   - aria-label on icon-only buttons
   - Form inputs have associated <label>

3. Color and contrast
   - Text contrast ratio ≥ 4.5:1 (normal), ≥ 3:1 (large)
   - Information not conveyed by color alone

4. Interactive elements
   - Buttons use <button>, links use <a href>
   - No click handlers on non-interactive elements (div, span)

5. Semantic HTML
   - Heading hierarchy (h1 → h2 → h3, no skipping)
   - Landmark roles (main, nav, header, footer)
```

### 6. Release Check

배포 전 Claude가 누락하기 쉬운 항목을 점검한다.

#### SKILL.md 예시

```md
---
name: release-check
description: Use before deploying to check for common release issues
---

Pre-release checklist:

1. Code quality
   - No console.log, debugger, or TODO left in changed files
   - No hardcoded API URLs or secrets
   - TypeScript errors resolved (tsc --noEmit)

2. Tests
   - All tests passing
   - New features have test coverage

3. Dependencies
   - No unused packages added
   - No known vulnerabilities (npm audit)

4. Environment
   - .env.example updated if new env vars added
   - Feature flags set correctly for production

5. Breaking changes
   - API contract changes documented
   - Database migrations included and reversible
```

### 추천 React Skill 세트

실무 React 프로젝트라면 아래 구성을 추천한다.

```
.claude/
└── skills/
    ├── create-component/     # 신규 개발
    │   └── SKILL.md
    ├── review-react/         # 코드 리뷰
    │   └── SKILL.md
    ├── migrate-to-hooks/     # 레거시 마이그레이션
    │   └── SKILL.md
    ├── accessibility-check/  # WCAG 검사
    │   └── SKILL.md
    ├── performance-review/   # 최적화
    │   └── SKILL.md
    └── release-check/        # 배포 전 검증
        └── SKILL.md
```

| Skill               | 목적                |
| ------------------- | ------------------- |
| create-component    | 신규 개발           |
| review-react        | 코드 리뷰           |
| migrate-to-hooks    | 레거시 마이그레이션 |
| accessibility-check | WCAG 검사           |
| performance-review  | 최적화              |
| release-check       | 배포 전 검증        |

## 좋은 Skill 설계 원칙

### 1. 지식보다 절차를 넣는다

| 좋음                               | 나쁨                 |
| ---------------------------------- | -------------------- |
| 새 컴포넌트 생성 시 거쳐야 할 단계 | React란 무엇인가     |
| 팀의 코드 리뷰 체크리스트          | 좋은 코드란 무엇인가 |

### 2. 프로젝트 규칙을 명시한다

Claude는 일반 React 지식은 이미 알고 있다.  
Skill에는 **이 프로젝트에서만 적용되는 규칙**을 넣는 것이 효과적이다.

```
Use Zustand for global state (not Redux)
Use React Query for server state (not useEffect + fetch)
Use Zod for all validation schemas
```

### 3. description을 구체적으로 작성한다

Claude가 자동으로 Skill을 불러올 때 `description`을 기준으로 판단하기 때문에, 설명이 명확할수록 정확하게 동작한다.

| 나쁜 예            | 좋은 예                                                                      |
| ------------------ | ---------------------------------------------------------------------------- |
| `React helper`     | `Use when creating new React components, pages, or refactoring existing`     |
| `Performance tool` | `Use when reviewing React components for performance issues or optimization` |

## Skill은 어디서 사용할 수 있나?

SKILL.md 기반 Skill은 **Claude Code 전용** 기능이다.

| 환경                          | Skill 지원           |
| ----------------------------- | -------------------- |
| Claude Code CLI               | O                    |
| VS Code / JetBrains Extension | O (Claude Code 기반) |
| claude.ai 웹                  | X                    |
| Claude API                    | X                    |

claude.ai 웹이나 API를 직접 사용하는 경우에는 SKILL.md가 로드되지 않는다.  
비슷한 효과를 내려면 시스템 프롬프트에 절차를 직접 포함하는 방식으로 대체해야 한다.

## 다른 AI 도구와 비교

Skill과 완전히 동일한 개념을 가진 도구는 현재 없다.  
다만 방향이 비슷한 것들은 있다.

### 프로젝트 지시 파일 (CLAUDE.md에 더 가까운 것들)

다른 도구들도 프로젝트 단위 AI 규칙 파일을 지원한다.  
하지만 이들은 항상 로드되는 방식이라 CLAUDE.md에 가깝고, Skill처럼 상황에 따라 선택적으로 로드되는 개념은 아니다.

| 도구           | 파일                              |
| -------------- | --------------------------------- |
| Cursor         | `.cursorrules`                    |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Windsurf       | `.windsurfrules`                  |

### OpenAI Custom GPTs

목적에 특화된 에이전트를 미리 구성해두는 방식으로, '특정 상황에서 특정 행동 방식을 정의한다'는 점에서 Skill과 방향이 비슷하다.  
다만 Custom GPT는 별도의 에이전트를 만드는 것에 가깝고, Skill은 하나의 Claude 안에서 필요한 모듈을 그때그때 불러오는 구조라 동작 방식이 다르다.

### MCP Prompts

Anthropic이 공개한 오픈 프로토콜인 **MCP(Model Context Protocol)**에는 'Prompts'라는 리소스가 있다.  
MCP 서버가 재사용 가능한 프롬프트 템플릿을 노출하면, 클라이언트가 필요할 때 불러오는 구조다.  
Claude Code뿐 아니라 MCP를 지원하는 다른 AI 도구에서도 동일하게 동작한다.

Claude Skill이 Claude Code에 특화된 모듈이라면, MCP Prompts는 도구에 종속되지 않는 더 범용적인 확장 방식이라고 볼 수 있다.

## 결론

Personal Skill은 나만의 AI 개발 습관 저장소를 만드는 것이다.  
Project Skill은 팀의 AI 개발 프로세스를 코드화하는 것이다.  
좋은 Skill이란 지식을 알려주는 것이 아니라, 프로젝트에서 반복되는 판단과 절차를 자동화하는 것이다.
