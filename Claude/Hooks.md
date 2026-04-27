# Claude Code Hooks 완벽 정리 🔗

> **이전 학습**: Skills까지 완료했다면, 이제 Hooks로 완전 자동화하기!

---

## 📋 Hooks란?

**Hooks** = Claude Code의 생명주기 중 특정 시점에 **자동으로 실행**되는 작은 프로그램

```
트리거 이벤트 (파일 저장, 배포 시작, 도구 호출 등)
    ↓
조건 확인 (matcher, if 필드)
    ↓
Hook 실행 (bash 스크립트, HTTP 요청, 또는 LLM 프롬프트)
    ↓
결과 처리 (허용/차단/피드백)
```

---

## 🎯 Hooks vs Skills vs Subagent

|항목|Hooks|Skills|Subagent|
|---|---|---|---|
|**트리거**|자동 (이벤트 발생 시)|수동 (`/skill`) 또는 자동|명시적 호출|
|**목적**|자동화 & 보안 & 검증|반복 작업 저장|독립 작업 실행|
|**실행 시점**|생명주기 특정 시점|사용자가 호출|필요할 때|
|**편집**|JSON|Markdown|Markdown|
|**예시**|파일 저장 시 자동 lint|`/deploy production`|코드 리뷰어|

---

## 🌊 Hook 생명주기 (전체 흐름)

```
┌─────────────────────────────────────────────────────────┐
│                    SessionStart                          │
│             (세션 시작, 환경 설정)                        │
└────────────────────┬────────────────────────────────────┘
                     │
     ┌───────────────┴───────────────┐
     ↓                               ↓
InstructionsLoaded              UserPromptSubmit
(CLAUDE.md 로드)                (사용자 프롬프트)
                                    │
                                    ↓
                            UserPromptExpansion
                           (슬래시 명령어 확장)
                                    │
     ┌──────────────────────────────┴──────────────────────┐
     │        Agentic Loop (도구 호출 반복)                 │
     │                                                      │
     ├─→ PreToolUse (도구 실행 전)                        │
     │   ├─→ PermissionRequest (권한 요청)               │
     │   │   └─→ PermissionDenied (거부되면)              │
     │   │                                                 │
     │   └─→ 도구 실행                                     │
     │       ├─→ PostToolUse (성공)                       │
     │       └─→ PostToolUseFailure (실패)               │
     │                                                      │
     ├─→ SubagentStart / SubagentStop (subagent)        │
     │                                                      │
     └─→ Notification (알림)                              │
                                                          
     CwdChanged (디렉토리 변경)
     FileChanged (파일 변경)
     ConfigChange (설정 변경)
     InstructionsLoaded (지시사항 로드)
                                    │
                                    ↓
                                 Stop
                            (Claude 응답 끝)
                                    │
     ┌──────────────────────┬───────┴────┬──────────────────┐
     ↓                      ↓             ↓                  ↓
 PreCompact          PostCompact   StopFailure        TaskCreated
 (압축 전)          (압축 후)      (API 오류)      (작업 생성)
                                                         │
                                                   TaskCompleted
                                                   (작업 완료)
                                                         │
                                    ┌────────────────────┘
                                    ↓
                            TeammateIdle
                         (팀 동료 대기)
                                    │
                                    ↓
                            SessionEnd
                          (세션 종료)
```

---

## 📍 Hook 설정 위치

### 범위별 저장 위치

|위치|범위|공유 가능|사용 케이스|
|---|---|---|---|
|`~/.claude/settings.json`|모든 프로젝트|❌|개인 자동화|
|`.claude/settings.json`|이 프로젝트만|✅|팀 규칙|
|`.claude/settings.local.json`|이 프로젝트|❌|로컬 테스트|
|Plugin `hooks/hooks.json`|플러그인 활성화|✅|재배포|
|Skill/Agent 프론트매터|컴포넌트 활성화|✅|한정된 범위|

### 기본 구조

```json
{
  "hooks": {
    "PreToolUse": [              // 이벤트
      {
        "matcher": "Bash",        // 필터
        "hooks": [
          {
            "type": "command",    // 핸들러 타입
            "command": "..."       // 실행할 명령
          }
        ]
      }
    ]
  }
}
```

---

## 🔧 Hook 핸들러 4가지 타입

### 1️⃣ Command Hooks (Bash 스크립트)

```json
{
  "type": "command",
  "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/validate.sh",
  "timeout": 30,
  "async": false,
  "statusMessage": "검증 중..."
}
```

**특징**:

- stdin으로 JSON 받음
- stdout/stderr로 응답
- exit code로 결과 전달 (0=성공, 2=차단)
- 가장 강력하고 유연함

### 2️⃣ HTTP Hooks (원격 서버)

```json
{
  "type": "http",
  "url": "http://localhost:8080/hooks/pre-tool-use",
  "timeout": 30,
  "headers": {
    "Authorization": "Bearer $API_TOKEN"
  },
  "allowedEnvVars": ["API_TOKEN"]
}
```

**특징**:

- POST 요청으로 JSON 전송
- 응답 바디로 결과 반환
- 원격 서버와 통합
- 비동기 처리 쉬움

### 3️⃣ Prompt Hooks (LLM 평가)

```json
{
  "type": "prompt",
  "prompt": "이 명령어를 실행해도 될까? $ARGUMENTS",
  "model": "claude-opus-4-6",
  "timeout": 30
}
```

**특징**:

- LLM에게 평가 맡김
- yes/no 결정 반환
- `$ARGUMENTS`로 입력값 참조
- 복잡한 판단에 유용

### 4️⃣ Agent Hooks (Subagent 실행)

```json
{
  "type": "agent",
  "prompt": "이 코드를 검토해줘: $ARGUMENTS",
  "model": "claude-sonnet-4-6"
}
```

**특징**:

- 읽기 도구 (Read, Grep, Glob) 사용 가능
- 더 복잡한 검증 가능
- 가장 강력하지만 느림

---

## 🎯 Hook 이벤트 30가지 (생명주기순)

### 세션 이벤트

#### 🟢 SessionStart

**언제**: 세션 시작/재개 **용도**: 환경 초기화, 변수 설정

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",  // 또는 resume, clear, compact
        "hooks": [{
          "type": "command",
          "command": "source ~/.nvm/nvm.sh && nvm use 20"
        }]
      }
    ]
  }
}
```

#### 🔴 SessionEnd

**언제**: 세션 종료 **용도**: 정리, 로깅, 백업

```json
{
  "matcher": "other",  // clear, resume, logout, prompt_input_exit 등
  "hooks": [{
    "type": "command",
    "command": "echo 'Session ended' >> ~/session-log.txt"
  }]
}
```

### 프롬프트 이벤트

#### UserPromptSubmit

**언제**: 사용자가 프롬프트 입력 **용도**: 프롬프트 검증/필터링

```json
{
  "hooks": {
    "UserPromptSubmit": [{
      "hooks": [{
        "type": "command",
        "command": ".claude/hooks/check-prompt.sh"
      }]
    }]
  }
}
```

#### UserPromptExpansion

**언제**: 슬래시 명령어 확장 (`/skill arg`) **용도**: 특정 스킬 차단, 컨텍스트 주입

```json
{
  "matcher": "deploy",  // 스킬 이름
  "hooks": [{
    "type": "command",
    "if": "command starts with 'deploy'",
    "command": "check-approval.sh"
  }]
}
```

### 도구 이벤트

#### PreToolUse ⭐ (가장 중요!)

**언제**: 도구 호출 전 **용도**: 위험한 명령 차단, 권한 확인

```bash
#!/bin/bash
# .claude/hooks/block-rm.sh

COMMAND=$(jq -r '.tool_input.command')

if echo "$COMMAND" | grep -q 'rm -rf'; then
  jq -n '{
    "hookSpecificOutput": {
      "hookEventName": "PreToolUse",
      "permissionDecision": "deny",
      "permissionDecisionReason": "위험한 명령어"
    }
  }'
else
  exit 0  # 허용
fi
```

**Matcher 값**: 도구 이름 (Bash, Edit, Write, Read, Glob, Grep, Agent, WebFetch, WebSearch, AskUserQuestion)

#### PostToolUse

**언제**: 도구 성공 후 **용도**: 결과 검증, 로깅

```json
{
  "matcher": "Write|Edit",
  "hooks": [{
    "type": "command",
    "command": "npm run lint -- $FILE"
  }]
}
```

#### PostToolUseFailure

**언제**: 도구 실패 후 **용도**: 에러 로깅, 조사

```json
{
  "hooks": [{
    "type": "command",
    "command": "echo 'Tool failed: $ERROR' >> ~/.errors.log"
  }]
}
```

### 권한 이벤트

#### PermissionRequest

**언제**: 권한 대화 표시 **용도**: 자동 승인/거부, 입력 수정

```json
{
  "matcher": "Bash",
  "hooks": [{
    "type": "command",
    "command": ".claude/hooks/approve-safe-commands.sh"
  }]
}
```

#### PermissionDenied

**언제**: Auto mode에서 거부됨 **용도**: 로깅, 재시도 허용

```json
{
  "hooks": [{
    "type": "command",
    "command": "jq -n '{\"hookSpecificOutput\": {\"hookEventName\": \"PermissionDenied\", \"retry\": true}}'"
  }]
}
```

### 파일/설정 이벤트

#### FileChanged

**언제**: 감시 중인 파일 변경 **용도**: 환경 재로드 (direnv 같은 도구)

```json
{
  "hooks": {
    "FileChanged": [
      {
        "matcher": ".envrc|.env",
        "hooks": [{
          "type": "command",
          "command": "direnv reload"
        }]
      }
    ]
  }
}
```

#### CwdChanged

**언제**: 작업 디렉토리 변경 **용도**: 디렉토리별 환경 설정

```json
{
  "hooks": {
    "CwdChanged": [{
      "hooks": [{
        "type": "command",
        "command": "[ -f .envrc ] && direnv load"
      }]
    }]
  }
}
```

#### ConfigChange

**언제**: 설정 파일 변경 **용도**: 설정 감사, 정책 강제

```json
{
  "matcher": "project_settings",  // user_settings, local_settings, policy_settings, skills
  "hooks": [{
    "type": "command",
    "command": ".claude/hooks/audit-config.sh"
  }]
}
```

#### InstructionsLoaded

**언제**: CLAUDE.md 또는 규칙 파일 로드 **용도**: 감사 로깅, 준수 확인

```json
{
  "matcher": "session_start",  // nested_traversal, path_glob_match, include, compact
  "hooks": [{
    "type": "command",
    "command": "echo 'Loaded: $FILE' >> ~/.audit.log"
  }]
}
```

### Subagent/팀 이벤트

#### SubagentStart / SubagentStop

**언제**: Subagent 시작/종료 **용도**: 로깅, 컨텍스트 주입

```json
{
  "hooks": {
    "SubagentStart": [
      {
        "matcher": "Explore|code-reviewer",
        "hooks": [{
          "type": "command",
          "command": "echo 'Agent starting: $TYPE' >> ~/.agent.log"
        }]
      }
    ]
  }
}
```

#### TaskCreated / TaskCompleted

**언제**: 작업 생성/완료 (Agent Teams) **용도**: 작업 검증, 품질 게이트

```json
{
  "hooks": {
    "TaskCreated": [{
      "hooks": [{
        "type": "command",
        "command": "check-task-format.sh"  // 작업 이름 형식 검사
      }]
    }]
  }
}
```

#### TeammateIdle

**언제**: 팀 동료가 대기할 때 **용도**: 완료 기준 강제

```json
{
  "hooks": {
    "TeammateIdle": [{
      "hooks": [{
        "type": "command",
        "command": "[ -f ./dist/output.js ] || exit 2  # 빌드 필요"
      }]
    }]
  }
}
```

### 대화 흐름 이벤트

#### Stop / StopFailure

**언제**: Claude 응답 끝 또는 API 오류 **용도**: 품질 확인, 재시도

```json
{
  "hooks": {
    "Stop": [{
      "hooks": [{
        "type": "command",
        "command": "[ $QUALITY_SCORE -gt 80 ] || exit 2  # 품질 낮음"
      }]
    }]
  }
}
```

### 컨텍스트 이벤트

#### PreCompact / PostCompact

**언제**: 컨텍스트 압축 전/후 **용도**: 로깅, 상태 업데이트

```json
{
  "matcher": "auto",  // 또는 manual
  "hooks": [{
    "type": "command",
    "command": "echo 'Compacting...' > ~/.compact-status"
  }]
}
```

### 고급 이벤트

#### WorktreeCreate / WorktreeRemove

**언제**: 고립된 작업 디렉토리 생성/제거 **용도**: SVN/Perforce 같은 대체 VCS

```json
{
  "hooks": {
    "WorktreeCreate": [{
      "hooks": [{
        "type": "command",
        "command": "svn checkout $URL $(jq -r '.name')"
      }]
    }]
  }
}
```

#### Notification

**언제**: 알림 전송 **용도**: 맞춤 알림 채널 (Slack, Discord 등)

```json
{
  "matcher": "permission_prompt",
  "hooks": [{
    "type": "http",
    "url": "https://hooks.slack.com/services/YOUR/WEBHOOK"
  }]
}
```

#### Elicitation / ElicitationResult

**언제**: MCP 서버가 사용자 입력 요청 **용도**: 자동 응답 또는 커스텀 UI

```json
{
  "hooks": {
    "Elicitation": [{
      "matcher": "anthropic/memory",
      "hooks": [{
        "type": "command",
        "command": ".claude/hooks/auto-respond.sh"
      }]
    }]
  }
}
```

---

## 💡 실전 Hook 예제

### 예제 1: Bash 명령 검증

```bash
#!/bin/bash
# .claude/hooks/validate-bash.sh

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')
EVENT=$(echo "$INPUT" | jq -r '.hook_event_name')

# 위험한 명령 차단
if echo "$COMMAND" | grep -qE '^rm -rf|^dd |^:(){:|' ; then
  jq -n "{
    \"hookSpecificOutput\": {
      \"hookEventName\": \"$EVENT\",
      \"permissionDecision\": \"deny\",
      \"permissionDecisionReason\": \"위험한 명령어\"
    }
  }"
  exit 0
fi

# git add -A 할 때 .env 제외
if echo "$COMMAND" | grep -q 'git add.*-A' ; then
  jq -n "{
    \"hookSpecificOutput\": {
      \"hookEventName\": \"$EVENT\",
      \"permissionDecision\": \"allow\",
      \"updatedInput\": {
        \"command\": \"git add :(exclude).env\"
      }
    }
  }"
  exit 0
fi

exit 0  # 다른 명령은 허용
```

**설정**:

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/validate-bash.sh"
      }]
    }]
  }
}
```

### 예제 2: 파일 저장 시 자동 Lint

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "if": "Edit(*.ts)",
            "command": "npx eslint --fix $FILE"
          },
          {
            "type": "command",
            "if": "Edit(*.py)",
            "command": "black $FILE && flake8 $FILE"
          }
        ]
      }
    ]
  }
}
```

### 예제 3: 배포 전 테스트 강제

```bash
#!/bin/bash
# .claude/hooks/pre-deploy-check.sh

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')

if echo "$COMMAND" | grep -q 'npm run deploy' ; then
  # 테스트 실행
  if ! npm test 2>&1 ; then
    jq -n '{
      "decision": "block",
      "reason": "테스트 실패 - 배포 불가"
    }'
    exit 0
  fi

  # 린트 체크
  if ! npm run lint 2>&1 ; then
    jq -n '{
      "decision": "block",
      "reason": "린트 오류 - 배포 불가"
    }'
    exit 0
  fi
fi

exit 0
```

### 예제 4: 환경 변수 자동 로드

```bash
#!/bin/bash
# .claude/hooks/load-env.sh
# SessionStart, CwdChanged, FileChanged에서 호출

if [ -n "$CLAUDE_ENV_FILE" ] && [ -f .envrc ] ; then
  eval "$(direnv export bash)" >> "$CLAUDE_ENV_FILE"
fi

exit 0
```

### 예제 5: 권한 자동 승인 (안전한 명령만)

```bash
#!/bin/bash
# .claude/hooks/auto-approve-safe.sh

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')

# 읽기 전용 명령은 자동 승인
if echo "$COMMAND" | grep -qE '^git status|^npm test|^ls |^cat ' ; then
  jq -n '{
    "hookSpecificOutput": {
      "hookEventName": "PermissionRequest",
      "decision": {
        "behavior": "allow"
      }
    }
  }'
  exit 0
fi

# 나머지는 사용자 승인 필요
exit 0
```

---

## ⚡ Exit Code와 JSON 응답

### Exit Code 의미

|코드|의미|사용|
|---|---|---|
|0|성공|stdout의 JSON 파싱, 또는 아무것도 없으면 허용|
|2|차단|이벤트 차단, stderr 피드백|
|1, 3+|오류|비차단 오류, 실행 계속|

### JSON 응답 형식

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny|ask|defer",
    "permissionDecisionReason": "이유 설명"
  }
}
```

또는 일반 응답:

```json
{
  "decision": "block",
  "reason": "이유",
  "additionalContext": "Claude에게 추가할 정보",
  "continue": false,
  "stopReason": "중단 이유"
}
```

---

## 🔒 Matcher와 IF 필드

### Matcher (그룹 필터)

```json
{
  "matcher": "Bash",           // 정확한 이름
  "matcher": "Bash|Edit|Write", // | 분리 목록
  "matcher": "^Notebook",       // 정규식 (특수문자 포함)
  "matcher": "mcp__.*__write.*" // MCP 도구
}
```

### IF 필드 (개별 필터)

```json
{
  "if": "Bash(rm *)",          // rm 명령어만
  "if": "Edit(*.ts)",          // TypeScript 파일만
  "if": "Bash(git push *)",     // git push만
  "if": "Write(src/**/*)"       // src 아래만
}
```

---

## 🌐 MCP Server 도구 매칭

MCP 도구는 `mcp__<server>__<tool>` 형식:

```json
{
  "matcher": "mcp__memory__.*",        // memory 서버의 모든 도구
  "matcher": "mcp__.*__write.*",       // 모든 서버의 write 도구
  "matcher": "mcp__github__search.*"   // GitHub 검색 도구
}
```

---

## 📚 완벽한 설정 예제

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/init-env.sh",
            "statusMessage": "환경 초기화 중..."
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(rm *)",
            "command": ".claude/hooks/validate-rm.sh",
            "timeout": 10
          },
          {
            "type": "command",
            "if": "Bash(git push *)",
            "command": ".claude/hooks/check-push.sh"
          }
        ]
      },
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "if": "Edit(*.ts)",
            "command": "npx tsc --noEmit"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "async": true,
            "command": "npm run lint -- $FILE"
          }
        ]
      }
    ],
    "FileChanged": [
      {
        "matcher": ".envrc|.env",
        "hooks": [
          {
            "type": "command",
            "command": "direnv load"
          }
        ]
      }
    ]
  }
}
```

---

## 🔥 고급 패턴

### 패턴 1: Async Hook (비동기 백그라운드)

```json
{
  "type": "command",
  "async": true,           // 백그라운드에서 실행
  "asyncRewake": true,     // 완료 시 Claude 깨우기
  "command": "npm run build"
}
```

### 패턴 2: HTTP Hook (원격 검증)

```json
{
  "type": "http",
  "url": "https://my-api.com/validate",
  "headers": {
    "Authorization": "Bearer $TOKEN",
    "X-Session": "${CLAUDE_SESSION_ID}"
  },
  "allowedEnvVars": ["TOKEN"]
}
```

### 패턴 3: Prompt Hook (LLM 판정)

```json
{
  "type": "prompt",
  "prompt": "이 PR이 배포해도 될 품질인가? $ARGUMENTS",
  "model": "claude-opus-4-6"
}
```

### 패턴 4: Tool-Specific Hook

```json
{
  "matcher": "mcp__memory__.*",
  "hooks": [{
    "type": "command",
    "command": "log-memory-ops.sh"
  }]
}
```

---

## 🛠️ 환경 변수 참조

Hook에서 사용 가능한 환경 변수:

```bash
$CLAUDE_PROJECT_DIR     # 프로젝트 루트
${CLAUDE_PLUGIN_ROOT}   # 플러그인 루트
${CLAUDE_PLUGIN_DATA}   # 플러그인 데이터 디렉토리
$CLAUDE_ENV_FILE        # 환경 변수 파일 (SessionStart, CwdChanged, FileChanged)
$CLAUDE_SESSION_ID      # 세션 ID
$CLAUDE_CODE_REMOTE     # "true"이면 원격 환경
```

---

## 🚀 실전 워크플로우

### 워크플로우 1: 자동 린팅 & 포매팅

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "if": "Edit(*.ts)",
            "async": true,
            "command": "npx eslint --fix $FILE && npx prettier --write $FILE",
            "statusMessage": "코드 형식 정리 중..."
          }
        ]
      }
    ]
  }
}
```

### 워크플로우 2: 위험한 명령 차단

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/security-check.sh"
          }
        ]
      }
    ]
  }
}
```

### 워크플로우 3: 배포 품질 게이트

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "if": "Bash(deploy|npm run deploy)",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/pre-deploy-checks.sh"
          }
        ]
      }
    ]
  }
}
```

---

## 📊 모든 이벤트 한눈에

```
세션 이벤트:        SessionStart, SessionEnd
프롬프트 이벤트:    UserPromptSubmit, UserPromptExpansion
도구 이벤트:        PreToolUse, PostToolUse, PostToolUseFailure
권한 이벤트:        PermissionRequest, PermissionDenied
파일 이벤트:        FileChanged, CwdChanged
설정 이벤트:        ConfigChange, InstructionsLoaded
Subagent 이벤트:   SubagentStart, SubagentStop
팀 이벤트:          TaskCreated, TaskCompleted, TeammateIdle
흐름 이벤트:        Stop, StopFailure
컨텍스트 이벤트:   PreCompact, PostCompact
고급 이벤트:        WorktreeCreate, WorktreeRemove
알림 이벤트:        Notification
MCP 이벤트:         Elicitation, ElicitationResult
```

---

## 🎯 체크리스트

### 기본 Hook 설정

```
[ ] 첫 번째 Hook 만들기 (PreToolUse)
[ ] .claude/settings.json에 작성
[ ] 테스트 실행
[ ] 에러 로그 확인
```

### 실전 적용

```
[ ] 린팅 Hook 추가 (PostToolUse)
[ ] 권한 Hook 추가 (PermissionRequest)
[ ] 환경 초기화 Hook (SessionStart)
[ ] 팀원과 공유
```

### 고급 기능

```
[ ] Async Hook 추가
[ ] HTTP Hook 통합
[ ] MCP Tool Hook
[ ] 에러 처리 개선
```

---

**다음**: Hooks를 Skills와 Subagent와 함께 사용하면 완벽한 자동화 워크플로우 완성!