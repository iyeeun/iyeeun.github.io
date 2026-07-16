---
layout: post
title: Claude의 Agent와 Skill 비교하기
date: 2026-07-16 09:10 +0900
description: Claude Code의 Subagent와 Skill은 무엇이 다르고, 각각 언제 써야 하는지 비교해본다
categories: [살펴보기, AI]
tags: [AI활용, claude, claude code, agent, skill]
---

지난 글에서 [Claude의 Skill](/posts/claude의-skill-알아보기/)을 다뤘다.  
Skill은 특정 작업 절차나 도메인 지식을 재사용 가능하게 만드는 확장 모듈이었다.  
그런데 Claude Code에는 비슷해 보이지만 전혀 다르게 동작하는 개념이 하나 더 있다. 바로 **Agent(Subagent)**이다.

Skill과 Agent는 둘 다 '특정 상황에 맞춰 Claude의 동작을 확장한다'는 목적은 비슷하지만, **무엇을 확장하는가**가 다르다.  
Skill이 Claude에게 **지식과 절차**를 주입한다면, Agent는 아예 **별도의 작업자를 만들어 위임**한다.

## Agent(Subagent)란?

Subagent는 특정 작업을 전담하는 **독립적인 AI 작업자**다.  
자신만의 `컨텍스트 윈도우`, `시스템 프롬프트`, `도구 권한`을 가지고 별도로 실행되며, 작업이 끝나면 필요한 결과만 메인 대화로 반환한다.  
메인 대화창에서 파일 탐색, 로그 확인, 대량의 코드 검색을 직접 수행하면 그 결과가 전부 컨텍스트에 쌓인다.  
Subagent는 이런 부수적인 작업을 별도 컨텍스트에서 처리하게 하여, **메인 대화는 요약된 결과만** 받도록 한다.

## 탄생 배경

Claude Code로 복잡한 작업을 진행하다 보면 두 가지 문제가 반복된다.

1\. **컨텍스트 오염**  
코드베이스 전체를 뒤지는 탐색 작업, 긴 로그 분석 같은 작업은 결과물의 양이 많다.  
해당 결과를 메인 대화에 그대로 쌓으면 정작 중요한 판단에 쓸 컨텍스트가 부족해진다.

2\. **반복되는 위임 패턴**  
특정한 요청을 위임하는 패턴이 반복될 경우, 정해진 역할과 도구 권한을 가진 전담 작업자를 미리 정의하여 반복 설명 없이 요청을 실행할 수 있다.  
예를 들면 특정 코드를 찾는 것이나 코드 리뷰에 대한 프롬프트를 정해두고 필요할 때 해당 Agent를 호출하여 작업할 수 있다.

## 기본 동작 원리

### 컨텍스트 격리

Subagent의 가장 핵심적인 특징은 **자신만의 컨텍스트 윈도우**에서 동작한다는 점이다.  
메인 대화가 Subagent에게 작업을 위임하면, Subagent는 별도의 컨텍스트에서 파일을 읽고 코드를 분석하거나 수정한 뒤, 관련성 있는 결과만 메인 대화로 돌려준다. 탐색 과정에서 발생한 방대한 중간 결과물은 메인 대화에 남지 않는다.

### 자동 위임과 명시적 호출

Subagent는 두 가지 방식으로 실행된다.

1\. **자동 위임**: Claude가 대화 맥락을 분석해서, Subagent의 `description`에 부합하는 작업이라고 판단하면 자동으로 위임한다.

2\. **명시적 호출**: 사용자가 특정 Subagent를 지정해서 직접 요청한다.

```
code-reviewer 에이전트로 이 PR을 리뷰해줘
```

Skill의 `/skill-name` 수동 호출과 비슷해 보이지만, Skill은 호출 즉시 현재 대화 컨텍스트에 지침이 로드되는 반면 Subagent는 별도의 컨텍스트에서 작업을 수행한 뒤 결과만 돌아온다는 점이 다르다.

### Subagent 파일 구조

Subagent는 Skill과 마찬가지로 YAML frontmatter + Markdown 본문 형태의 파일로 정의한다.

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices. Use proactively after code changes.
tools: Read, Grep, Glob, Bash
model: sonnet
---

You are a code reviewer. When invoked, analyze the code and provide
specific, actionable feedback on quality, security, and best practices.
```

frontmatter에는 `name`과 `description`이 필수다. `description`은 Claude가 **언제 이 Subagent에게 위임할지** 판단하는 기준이 된다는 점에서 Skill의 `description`과 역할이 같다.

이 외에도 다음과 같은 필드를 지정할 수 있다.

| 필드              | 설명                                                               |
| ----------------- | ------------------------------------------------------------------ |
| `tools`           | Subagent가 사용할 수 있는 도구 목록. 생략하면 모든 도구를 상속     |
| `disallowedTools` | 상속받은 도구 중 금지할 도구 목록                                  |
| `model`           | 사용할 모델(`sonnet`, `opus`, `haiku` 등). 기본값은 메인 세션 상속 |
| `permissionMode`  | 권한 모드(`default`, `acceptEdits`, `plan` 등)                     |
| `skills`          | Subagent 시작 시 미리 로드할 Skill 목록                            |
| `mcpServers`      | Subagent가 사용할 수 있는 MCP 서버                                 |
| `isolation`       | `worktree`로 설정하면 격리된 Git worktree에서 작업                 |
| `maxTurns`        | Subagent가 멈추기 전까지 수행할 최대 턴 수                         |

본문(시스템 프롬프트)은 해당 Subagent가 어떤 역할로 행동해야 하는지를 정의한다.  
중요한 것은 Subagent는 Claude Code의 전체 시스템 프롬프트를 물려받지 않고 **이 시스템 프롬프트와 기본 환경 정보(작업 디렉토리 등)만 받는다**는 것이다.

### 도구 권한 제어

Subagent는 기본적으로 메인 대화가 가진 도구를 그대로 상속한다.  
`tools`를 화이트리스트로, `disallowedTools`를 블랙리스트로 사용해서 권한 범위를 좁힐 수 있다.

```yaml
---
name: safe-researcher
description: Research agent with restricted capabilities
tools: Read, Grep, Glob, Bash
---
```

위처럼 정의하면 이 Subagent는 파일을 읽고 검색만 할 수 있을 뿐, 파일을 수정하거나 MCP 도구를 호출할 수 없다.  
탐색 전용 Subagent는 읽기 권한만, 구현을 맡기는 Subagent는 쓰기 권한까지 부여하는 식으로 역할에 맞게 권한을 최소화할 수 있다.

### Built-in Subagent

Claude Code는 기본 제공 Subagent를 몇 가지 내장하고 있다.

| Subagent        | 모델           | 도구      | 용도                                |
| --------------- | -------------- | --------- | ----------------------------------- |
| Explore         | 메인 세션 상속 | 읽기 전용 | 코드 검색, 파일 탐색                |
| Plan            | 메인 세션 상속 | 읽기 전용 | Plan Mode에서의 사전 조사           |
| general-purpose | 메인 세션 상속 | 모든 도구 | 탐색과 수정이 모두 필요한 복합 작업 |

Explore와 Plan은 코드 수정 없이 조사만 담당하고, general-purpose는 조사부터 실제 파일 수정까지 함께 처리한다.  
이 외에도 `/statusline` 설정을 돕는 `statusline-setup`, Claude Code 사용법을 안내하는 `claude-code-guide`처럼 특정 상황에서만 자동으로 호출되는 보조 Subagent도 있다.

## Subagent 종류 (저장 위치별)

Skill이 Personal / Project / Plugin으로 나뉘었던 것처럼, Subagent도 저장 위치에 따라 범위가 달라진다.

| 위치                        | 범위                 | 우선순위  | 특징                                 |
| --------------------------- | -------------------- | --------- | ------------------------------------ |
| 관리자 배포 설정            | 조직 전체            | 1(최우선) | 관리자가 조직 전체에 배포            |
| `--agents` CLI 플래그       | 현재 세션만          | 2         | 디스크에 저장되지 않는 임시 정의     |
| `.claude/agents/`           | 현재 프로젝트        | 3         | Git으로 팀과 공유                    |
| `~/.claude/agents/`         | 모든 프로젝트        | 4         | 개인 작업 습관 저장                  |
| Plugin의 `agents/` 디렉토리 | Plugin이 설치된 환경 | 5(최하위) | `plugin-name:agent-name` 형태로 참조 |

동일한 이름의 Subagent가 여러 위치에 있으면 우선순위가 높은 쪽이 적용된다. Project Subagent는 팀 전체가 같은 작업자를 공유하도록 Git으로 관리하기 좋고, User Subagent는 여러 프로젝트를 오가며 쓰는 개인 워크플로우에 적합하다.

## Skill과 Agent의 차이

둘 다 필요할 때 불러와서 Claude의 동작을 바꾼다는 점은 비슷하지만, 내부 동작 방식은 근본적으로 다르다.

**Skill**은 현재 대화 컨텍스트에 지침을 **로드**한다.  
Skill이 실행되면 SKILL.md의 내용이 지금 이 대화의 컨텍스트에 추가되고, Claude는 그 지침을 참고하며 계속 같은 대화 흐름 안에서 작업한다.

**Agent**는 작업 자체를 별도 컨텍스트로 **위임**한다.  
Subagent가 호출되면 완전히 분리된 컨텍스트 윈도우가 새로 열리고, 그 안에서 독립적으로 작업이 진행된 뒤 결과만 메인 대화로 돌아온다. 위임된 작업의 세부 과정은 메인 대화에 남지 않는다.

### 비교표

| 구분            | Skill                                    | Agent(Subagent)                                    |
| --------------- | ---------------------------------------- | -------------------------------------------------- |
| 정체            | 지침·절차 문서                           | 독립 실행되는 작업자                               |
| 동작 위치       | 현재 대화 컨텍스트 안                    | 별도의 격리된 컨텍스트 윈도우                      |
| 반환하는 것     | 지침 내용 자체가 대화에 편입             | 요약된 결과만 메인 대화로 반환                     |
| 도구 권한       | 별도 권한 개념 없음(현재 세션 권한 사용) | `tools`/`disallowedTools`로 독립 설정 가능         |
| 모델            | 현재 세션 모델과 동일                    | Subagent별로 다른 모델 지정 가능                   |
| 시스템 프롬프트 | 없음(현재 시스템 프롬프트에 지침 추가)   | 자체 시스템 프롬프트를 가짐                        |
| 병렬 실행       | 불가능(현재 대화 흐름 그대로 이어짐)     | 여러 Subagent를 동시에 병렬 실행 가능              |
| 목적            | "어떻게 할지"를 알려주는 매뉴얼          | "누가 할지"를 정하는 위임 구조                     |
| 파일 형식       | `SKILL.md`                               | `.claude/agents/*.md`                              |
| 호출 형태       | `/skill-name`, 자동 로드                 | `Agent 도구`로 위임, `@agent-name` 언급, 자동 위임 |

Skill은 매뉴얼에 가깝고, Agent는 매뉴얼을 들고 일하는 사람에 가깝다.  
매뉴얼 자체는 누가 수행하든 똑같이 적용될 수 있지만, Agent는 매뉴얼과 별개로 자신만의 역할, 권한, 실행 공간을 갖는다.  
실제로 이 둘은 결합해서 쓸 수 있다. Subagent의 frontmatter에 `skills` 필드를 지정하면, 해당 Subagent가 시작될 때 특정 Skill의 전체 내용을 컨텍스트에 미리 로드해서 시작한다.

```yaml
---
name: react-reviewer
description: Reviews React components using team conventions
tools: Read, Grep, Glob
skills: react-performance-review, accessibility-check
---
You are a React code reviewer. Apply the loaded skills strictly.
```

이렇게 하면 React 리뷰를 전담하는 격리된 작업자(Agent)가 팀의 리뷰 체크리스트(Skill)를 들고 작업을 수행하는 조합이 만들어진다.

## 언제 무엇을 써야 할까

### Skill을 쓰는 경우

- 메인 대화 흐름을 유지한 채로 특정 절차나 규칙만 참고하면 될 때
- 컴포넌트 생성, 코드 컨벤션 적용처럼 **결과가 바로 지금의 대화에 이어져야** 할 때
- 별도로 격리할 필요 없는 가벼운 지침 삽입이 목적일 때

### Agent를 쓰는 경우

- 코드베이스 전체 탐색처럼 **결과물의 양이 많아 메인 컨텍스트를 오염시킬** 작업일 때
- 탐색은 읽기 전용, 구현은 쓰기 권한처럼 **역할별로 도구 권한을 분리**하고 싶을 때
- 여러 작업을 **동시에 병렬로** 처리하고 싶을 때
- 특정 역할에 **더 저렴하거나 더 강력한 모델**을 지정해서 비용을 최적화하고 싶을 때

### 함께 쓰는 경우

- Subagent가 반복적으로 수행하는 작업에 일관된 체크리스트나 컨벤션이 필요할 때

## 결론

Skill은 Claude가 **어떻게 판단하고 행동해야 하는지**를 정의하는 매뉴얼이고, Agent는 **누가, 어떤 권한과 컨텍스트로 그 작업을 수행하는지**를 정의하는 구조다.

컨텍스트를 아끼고 싶다면 Agent로 작업을 격리하고, 판단 기준을 명확히 하고 싶다면 Skill로 절차를 명시한다.  
또 둘을 조합하면 격리된 전담 작업자가 정해진 매뉴얼을 따라 일관되게 작업을 수행하는 구조를 만들 수 있다.
