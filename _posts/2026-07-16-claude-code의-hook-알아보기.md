---
layout: post
title: Claude Code의 Hook 알아보기
date: 2026-07-16 16:40 +0900
description: Claude Code의 Hook은 무엇이고, Skill이나 Agent와는 어떻게 다른지 알아본다
categories: [살펴보기, AI]
tags: [AI활용, claude, claude code, hook]
---

지금까지 [Skill](/posts/claude의-skill-알아보기/)과 [Agent](/posts/claude의-agent와-skill-비교하기/)를 다뤘다.  
둘 다 Claude의 판단에 개입하는 방식이었다면, **Hook**은 성격이 다르다.  
Hook은 Claude의 판단을 거치지 않고 **정해진 시점에 무조건 실행되는 규칙**이다.

## Hook이란?

Hook은 Claude Code의 세션 생명주기 중 특정 시점에 자동으로 실행되는 `사용자 정의 셸 명령어`, `HTTP 엔드포인트`, 또는 `LLM 프롬프트`다.  
파일을 수정하기 직전, 프롬프트를 제출한 직후, 세션이 시작될 때처럼 **정해진 이벤트가 발생**하면 Claude Code가 등록된 Hook을 실행하고, Hook은 그 결과에 따라 작업을 허용하거나 막을 수 있다.  
가장 중요한 특징은, Hook은 Claude의 판단이 아니라 결정론적인 규칙이라는 점이다.  
Skill이 '이런 상황이면 이렇게 해라'는 지침을 Claude에게 주는 것이라면, Hook은 조건만 맞으면 무조건 실행된다.

## 탄생 배경

Skill이나 CLAUDE.md에 아무리 좋은 지침을 적어놔도, Claude가 그 지침을 놓치거나 다르게 판단하면 실행되지 않을 수 있다.  
이런 문제를 해결하려면 Claude의 판단과 무관하게 항상 실행되는 규칙이 필요하다.  
위험한 명령어를 원천 차단하거나, 파일을 수정할 때마다 자동으로 린트를 돌리거나, 세션이 시작될 때마다 최신 브랜치 정보를 주입하는 것처럼 확률이 아니라 **확실성이 필요한 작업**에는 Hook이 적합하다.

## 기본 동작 원리

### 설정 구조

Hook 설정은 3단계로 구성된다.

1\. **이벤트 선택**: `PreToolUse`, `Stop` 같은 생명주기 이벤트를 고른다.

2\. **Matcher로 필터링**: "Bash 도구에만 적용" 같은 조건으로 범위를 좁힌다.

3\. **Handler 정의**: 조건이 맞으면 실제로 실행할 동작을 정의한다.

### Hook 생명주기 (이벤트)

Hook이 발생하는 시점(이벤트)은 아래처럼 다양하다.

| 구분               | 이벤트                | 발생 시점                                       |
| ------------------ | --------------------- | ----------------------------------------------- |
| **세션 단위**      | `SessionStart`        | 세션이 시작되거나 재개될 때                     |
|                    | `Setup`               | `--init-only` 등으로 사전 준비 작업을 실행할 때 |
|                    | `SessionEnd`          | 세션이 종료될 때                                |
| **턴(turn) 단위**  | `UserPromptSubmit`    | 프롬프트를 제출한 직후, Claude가 처리하기 전    |
|                    | `UserPromptExpansion` | 슬래시 커맨드 등이 프롬프트로 확장될 때         |
|                    | `Stop`                | Claude가 응답을 마쳤을 때                       |
|                    | `StopFailure`         | API 오류로 턴이 종료됐을 때                     |
| **도구 호출 단위** | `PreToolUse`          | 도구 호출 직전(차단 가능)                       |
|                    | `PermissionRequest`   | 권한 승인 다이얼로그가 뜰 때                    |
|                    | `PermissionDenied`    | Auto Mode가 도구 호출을 거부했을 때             |
|                    | `PostToolUse`         | 도구 호출이 성공한 직후                         |
|                    | `PostToolUseFailure`  | 도구 호출이 실패한 직후                         |
|                    | `PostToolBatch`       | 병렬 도구 호출 묶음이 모두 끝난 직후            |
| Subagent 단위      | `SubagentStart`       | Subagent가 새로 생성될 때                       |
|                    | `SubagentStop`        | Subagent 작업이 끝났을 때                       |
| Task 단위          | `TaskCreated`         | `TaskCreate`로 Task가 생성될 때                 |
|                    | `TaskCompleted`       | Task가 완료 처리될 때                           |
| Agent Team 단위    | `TeammateIdle`        | Agent Team의 팀원이 유휴 상태로 전환되기 직전   |
| 워크트리 단위      | `WorktreeCreate`      | Git worktree가 생성될 때                        |
|                    | `WorktreeRemove`      | Git worktree가 제거될 때                        |
| 컨텍스트 압축 단위 | `PreCompact`          | 컨텍스트 압축 직전                              |
|                    | `PostCompact`         | 컨텍스트 압축 직후                              |
| MCP 상호작용 단위  | `Elicitation`         | MCP 서버가 도구 호출 중 사용자 입력을 요청할 때 |
|                    | `ElicitationResult`   | 사용자가 Elicitation에 응답한 직후              |
| 독립 비동기 이벤트 | `Notification`        | Claude Code가 알림을 보낼 때                    |
|                    | `MessageDisplay`      | Claude의 응답 텍스트가 화면에 출력되는 동안     |
|                    | `InstructionsLoaded`  | CLAUDE.md 등 지침 파일이 컨텍스트에 로드될 때   |
|                    | `ConfigChange`        | 세션 도중 설정 파일이 변경될 때                 |
|                    | `CwdChanged`          | 작업 디렉토리가 변경될 때(예: `cd` 실행)        |
|                    | `FileChanged`         | 감시 중인 파일이 디스크에서 변경될 때           |

### Matcher 패턴

`matcher` 필드는 어떤 상황에서 Hook을 실행할지 필터링한다.

| Matcher 형태              | 동작 방식                              | 예시                                             |
| ------------------------- | -------------------------------------- | ------------------------------------------------ |
| `*`, `""`, 또는 생략      | 모든 경우에 매칭                       | 이벤트가 발생할 때마다 실행                      |
| 문자·숫자·`,`·`|`만 포함 | 정확히 일치하는 문자열(복수 지정 가능) | `Bash`는 Bash 도구만, `Edit|Write`는 둘 중 하나 |
| 그 외 특수문자 포함       | 정규식으로 해석                        | `mcp__memory__.*`는 memory 서버의 모든 도구      |

각 이벤트는 서로 다른 필드를 기준으로 matcher를 검사한다.  
`PreToolUse`/`PostToolUse`는 도구 이름을, `SubagentStart`는 Agent 타입을, `SessionStart`는 세션 시작 방식(`startup`, `resume` 등)을 기준으로 매칭한다.

### Handler 종류

Hook이 실제로 실행하는 동작(Handler)은 5가지 타입이 있다.

| 타입       | 설명                                                       |
| ---------- | ---------------------------------------------------------- |
| `command`  | 셸 명령어를 실행 (가장 널리 쓰이는 방식)                   |
| `http`     | JSON 데이터를 POST 요청으로 지정된 URL에 전송              |
| `mcp_tool` | 이미 연결된 MCP 서버의 도구를 호출                         |
| `prompt`   | Claude 모델에게 단일 턴 프롬프트를 보냄 (yes/no)           |
| `agent`    | Subagent를 띄워서 도구를 활용한 검증을 거친 뒤 판단을 받음 |

`command` 타입이 가장 단순하고 많이 쓰이지만, 판단 자체를 LLM에 맡기고 싶을 때는 `prompt`나 `agent` 타입을 쓸 수 있다.  
다만 이 두 타입은 Hook임에도 내부적으로 LLM 판단을 다시 거친다는 점에서 일반적인 Hook의 결정론적 성격과는 다르다.

### 입력과 출력

Command Hook은 이벤트 정보를 JSON으로 stdin에 받고, 실행 결과는 **종료 코드(exit code)**와 **표준 출력(stdout)**으로 전달한다.

| 종료 코드 | 의미                                                   |
| --------- | ------------------------------------------------------ |
| `0`       | 성공, stdout에 출력한 JSON을 읽어서 세부 동작을 결정   |
| `2`       | 차단, stderr 내용이 Claude에게 오류 메시지로 전달됨    |
| 그 외     | 대부분 이벤트에서 비차단 오류로 처리되고 작업은 계속됨 |

예를 들어 `PreToolUse`에서 exit 2를 반환하면 해당 도구 호출 자체가 차단된다.  
반대로 `PostToolUse`처럼 이미 실행이 끝난 뒤 발생하는 이벤트는 exit 2를 반환해도 작업을 되돌릴 수는 없고, Claude에게 오류 메시지만 전달된다.  
exit 코드만으로는 "차단" 또는 "무시" 두 가지밖에 표현할 수 없기 때문에, 더 세밀한 제어가 필요하면 `exit 0`으로 종료하면서 **JSON을 출력**하는 방식을 쓴다.  
`continue`, `systemMessage`, `hookSpecificOutput` 같은 필드로 대화를 완전히 중단시키거나, 사용자에게 경고 메시지를 보여주거나, 특정 이벤트에 특화된 결정을 반환할 수 있다.

### 예시: `rm` 명령어 차단

예를 들어 위험한 `rm` 명령어를 차단하는 Hook은 다음과 같이 구성된다.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh"
          }
        ]
      }
    ]
  }
}
```

`PreToolUse` 이벤트가 발생했을 때, `matcher`가 `Bash` 도구 호출인지 확인하고, 조건이 맞으면 `block-rm.sh` 스크립트를 실행하는 구조다.  
이 Handler 스크립트는 이벤트 정보를 표준 입력(stdin)으로 받아 위험한 명령어인지 검사한다.

```bash
#!/bin/bash
# .claude/hooks/block-rm.sh
COMMAND=$(jq -r '.tool_input.command')

if echo "$COMMAND" | grep -q 'rm -rf'; then
  jq -n '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: "Destructive command blocked by hook"
    }
  }'
else
  exit 0  # 결정 없음: 평소 권한 흐름 그대로 진행
fi
```

결과는 `rm -rf`가 포함된 명령이면 exit 0과 함께 `permissionDecision: "deny"` JSON을 출력해서 도구 호출 자체를 막는다.  
안전한 명령이면 아무 JSON 없이 `exit 0`만 반환하고, 이 경우 평소 권한 흐름(사용자 승인 등)을 그대로 따른다.

## 위치별 Hook의 종류

Hook도 Skill이나 Agent와 마찬가지로 정의 위치에 따라 범위와 공유 여부가 달라진다.

| 위치                          | 범위                          | 공유 여부                           |
| ----------------------------- | ----------------------------- | ----------------------------------- |
| `~/.claude/settings.json`     | 모든 프로젝트                 | 개인 로컬 전용                      |
| `.claude/settings.json`       | 현재 프로젝트                 | Git으로 팀과 공유 가능              |
| `.claude/settings.local.json` | 현재 프로젝트                 | 공유 안 됨(자동으로 gitignore 처리) |
| 관리자 배포 설정              | 조직 전체                     | 관리자가 통제                       |
| Plugin의 `hooks/hooks.json`   | Plugin이 활성화된 동안        | Plugin과 함께 배포                  |
| Skill/Agent frontmatter       | 해당 컴포넌트가 활성화된 동안 | 해당 파일에 정의되어 함께 배포      |

Skill이나 Agent의 frontmatter에도 `hooks` 필드를 직접 정의할 수 있어서 해당 Skill이나 Agent가 활성화된 동안에만 적용되고 끝나면 자동으로 정리되는 **범위가 한정된 Hook**을 만들 수 있다.

```yaml
---
name: secure-operations
description: Perform operations with security checks
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/security-check.sh"
---
```

## Skill, Agent, Hook 비교

셋 다 '특정 상황에서 Claude의 동작에 개입한다'는 목적은 비슷하지만 개입하는 층이 다르다.

| 구분              | Skill                        | Agent(Subagent)             | Hook                                   |
| ----------------- | ---------------------------- | --------------------------- | -------------------------------------- |
| 역할              | 지침, 절차 문서              | 독립 실행되는 작업자        | 이벤트 발생 시 실행되는 규칙           |
| 개입 방식         | Claude에게 지침을 제공       | 작업을 별도 컨텍스트로 위임 | 이벤트를 가로채서 직접 실행/차단       |
| 실행 주체         | Claude(지침을 참고해서 판단) | 별도의 Subagent 프로세스    | 셸 명령어·HTTP·MCP 도구 등 외부 실행체 |
| 실행 보장         | Claude가 따를지는 확률적     | Claude가 위임할지는 확률적  | 조건만 맞으면 항상 실행(결정론적)      |
| 트리거            | 대화 맥락, 수동 호출         | 대화 맥락, 수동 호출        | 생명주기 이벤트(`PreToolUse` 등)       |
| 되돌릴 수 있는 것 | 판단 기준                    | 작업 결과                   | 도구 호출 자체를 사전에 차단 가능      |

정리하면 Skill은 **무엇을 알아야 하는지**, Agent는 **누가 수행하는지**, Hook은 **언제, 무조건 무엇이 실행되는지**를 담당한다고 볼 수 있다.  
세 개념은 함께 조합할 수도 있다. 예를 들어 특정 Agent에게만 적용되는 Hook을 정의해서, 그 Agent가 파일을 수정할 때마다 자동으로 보안 검사를 돌리게 만들 수 있다. Skill 안에 Hook을 넣어서, 그 Skill이 활성화된 동안에는 항상 특정 검증 스크립트가 실행되도록 강제할 수도 있다.

## 언제 무엇을 써야 할까

### Skill을 쓰는 경우

- Claude가 특정 절차나 컨벤션을 참고해서 판단하면 되는 경우
- 강제성보다는 가이드에 가까운 지침일 때

### Agent를 쓰는 경우

- 컨텍스트를 격리해서 별도로 처리하고 싶은 작업일 때
- 역할별로 도구 권한을 나누고 싶을 때

### Hook을 쓰는 경우

- Claude의 판단 여부와 무관하게 **반드시** 실행되어야 하는 작업일 때(보안 검사, 린트, 포맷팅)
- 위험한 명령어를 원천적으로 차단해야 할 때
- 파일 수정, 커밋, 세션 시작 등 특정 시점에 자동으로 부가 작업을 실행하고 싶을 때

## 결론

Skill과 Agent가 Claude의 판단 범위 안에서 동작하는 확장이라면, Hook은 그 판단 바깥에서 무조건 실행되는 안전장치에 가깝다.  
좋은 워크플로우는 셋을 적절히 나눠 쓰는 것이다. 세 도구는 아래와 같은 기준으로 사용하면 된다.

- 그냥 하라고 알려주는 것으로 충분하면 Skill
- 따로 떼어서 시키는 것이 필요하면 Agent
- 무조건 실행되도록 강제하는 것이 필요하면 Hook
