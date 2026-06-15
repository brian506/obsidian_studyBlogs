

## 📌 Subagent란?

**정의:** 특정 작업을 담당하는 전문화된 AI 어시스턴트. 자신의 컨텍스트 윈도우에서 독립적으로 실행되고, 결과만 메인으로 반환.

**핵심 아이디어:**

- 탐색, 로그, 검색 결과가 메인 컨텍스트를 오염시키지 않음
- 각 Subagent는 자신의 시스템 프롬프트, 도구 접근, 권한을 가짐
- 같은 종류의 작업을 반복 생성할 필요 없음

---

## 🏗️ Built-in Subagents (기본 제공)

### 1. Explore

- **모델:** Haiku (빠름)
- **도구:** 읽기 전용 (Write/Edit 불가)
- **용도:** 코드베이스 탐색, 파일 검색
- **언제 쓰이나:** Claude가 코드를 이해해야 할 때, 변경 없이 탐색만

### 2. Plan

- **모델:** 메인 모델 상속
- **도구:** 읽기 전용
- **용도:** Plan Mode에서 계획 수립 전 컨텍스트 수집
- **언제 쓰이나:** Plan Mode 사용할 때 자동

### 3. General-purpose

- **모델:** 메인 모델 상속
- **도구:** 모든 도구 가능
- **용도:** 탐색과 수정을 동시에 필요한 복잡한 작업
- **언제 쓰이나:** 단일 단계로 끝낼 수 없는 멀티스텝 작업

---

## 🚀 Custom Subagent 만드는 법

### 빠른 시작 (Quick Start)

**CLI 명령어:**

```bash
/agents
```

**단계:**

1. Library 탭 → Create new agent
2. Personal 선택 (사용자 레벨, `~/.claude/agents/`)
3. Generate with Claude 선택
4. 역할 설명: "코드를 스캔해서 개선사항을 제안하는 에이전트"
5. 도구 선택 (읽기 전용이면 Read-only tools만)
6. 모델 선택 (Sonnet 추천)
7. 저장

### 파일로 직접 작성

**Subagent 파일 구조:**

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Glob, Grep
model: sonnet
---

You are a code reviewer. When invoked, analyze the code and provide
specific, actionable feedback on quality, security, and best practices.
```

**저장 위치 (우선순위):**

|위치|범위|우선순위|용도|
|---|---|---|---|
|Managed settings|조직|1 (최고)|팀 전체|
|CLI `--agents`|현재 세션|2|일회성 테스트|
|`.claude/agents/`|현재 프로젝트|3|팀 공유 (Git)|
|`~/.claude/agents/`|모든 프로젝트|4|개인 사용|
|Plugin `agents/`|Plugin 활성화|5 (최하)|Plugin 제공|

---

## ⚙️ Subagent 설정 (Frontmatter 필드)

### 필수 필드

- **name:** 소문자/하이픈만 사용 (예: `code-reviewer`)
- **description:** Claude가 언제 이 Subagent를 쓸지 판단하는 기준

### 선택 필드

|필드|설명|예시|
|---|---|---|
|`tools`|사용 가능한 도구 (화이트리스트)|`Read, Glob, Grep`|
|`disallowedTools`|제외할 도구 (블랙리스트)|`Write, Edit`|
|`model`|사용할 모델|`sonnet`, `opus`, `haiku`, 또는 `inherit`|
|`permissionMode`|권한 모드|`default`, `acceptEdits`, `auto`, `dontAsk`, `bypassPermissions`, `plan`|
|`maxTurns`|최대 실행 턴 수|`10`|
|`skills`|미리 로드할 Skill|`api-conventions, error-handling`|
|`mcpServers`|MCP 서버|Inline 또는 참조|
|`memory`|영속 메모리 범위|`user`, `project`, `local`|
|`background`|백그라운드에서 실행|`true` / `false`|
|`effort`|노력 레벨|`low`, `medium`, `high`, `xhigh`, `max`|
|`isolation`|Git Worktree 격리|`worktree`|
|`color`|UI 표시 색상|`red`, `blue`, `green`, 등|

---

## 🛠️ 도구 제어

### Available Tools

```
Read, Write, Edit, Bash, Grep, Glob, Execute
```

### 도구 지정 방법

**화이트리스트 (tools 필드):**

```yaml
tools: Read, Grep, Glob, Bash
```

→ 이 4개만 사용 가능

**블랙리스트 (disallowedTools 필드):**

```yaml
disallowedTools: Write, Edit
```

→ 이 2개 제외하고 모두 가능

### Agent Tool로 Subagent 제한

```yaml
tools: Agent(worker, researcher), Read, Bash
```

→ worker와 researcher Subagent만 생성 가능

---

## 🧠 영속 메모리 (Persistent Memory)

**목적:** Subagent가 대화를 넘어 지식을 축적하게 함

**설정:**

```yaml
memory: user  # 또는 project, local
```

**범위:**

|범위|경로|용도|
|---|---|---|
|`user`|`~/.claude/agent-memory/<name>/`|모든 프로젝트에서 학습 축적|
|`project`|`.claude/agent-memory/<name>/`|프로젝트 특화 지식 (Git에 체크인)|
|`local`|`.claude/agent-memory-local/<name>/`|프로젝트 특화 (Git 미체크인)|

**사용 팁:**

- Subagent 시스템 프롬프트에 메모리 업데이트 지침 포함
- 작업 전: "메모리를 확인해"
- 작업 후: "배운 내용을 메모리에 저장해"

---

## 🎯 Subagent 사용 패턴

### 1. 자동 위임 (Automatic Delegation)

Claude가 자동으로 판단해서 Subagent 호출

```
"배포해줘"
→ deploy Subagent가 조건에 맞으면 Claude가 자동 위임
```

**팁:** Subagent description에 "use proactively" 추가하면 적극적 위임 유도

### 2. 명시적 호출 (3가지 방법)

**자연어 언급:**

```
Use the test-runner subagent to fix failing tests
```

**@-mention:**

```
@"code-reviewer (agent)" look at the auth changes
```

→ 특정 Subagent 확실히 지정

**세션 전체:**

```bash
claude --agent code-reviewer
```

→ 전체 세션이 이 Subagent의 프롬프트/도구/모델 사용

### 3. 포그라운드 vs 백그라운드

**포그라운드 (기본):**

- 메인 대화 차단
- 권한 프롬프트 노출
- 결과 받을 때까지 대기

**백그라운드:**

- 메인과 병렬 실행
- 시작 전 권한 미리 수집
- 계속 일하면서 Subagent 진행

**백그라운드 명령:**

```
"run this in the background"
또는 Ctrl+B
```

---

## 📋 Common Patterns (자주 쓰이는 패턴)

### 1. 고용량 작업 격리

테스트 실행, 로그 처리 등이 많은 텍스트를 생성할 때

```
Use a subagent to run the test suite and report only the failing tests
```

### 2. 병렬 리서치

독립적인 여러 조사를 동시 진행

```
Research the auth, database, and API modules in parallel using separate subagents
```

### 3. Subagent 체이닝

순차적으로 여러 Subagent 실행

```
Use the code-reviewer to find issues, then use the optimizer to fix them
```

---

## Subagent vs 메인 대화: 언제 뭘 쓸 건가?

### Subagent 쓰세요:

- ✅ 로그/탐색 결과가 많이 생김
- ✅ 도구 제한이 필요
- ✅ 독립적인 작업
- ✅ 특정 도메인 전문성 필요

### 메인 대화 쓰세요:

- ✅ 자주 반복 피드백 필요
- ✅ 단계 간 컨텍스트 공유 많음
- ✅ 빠른 수정 필요
- ✅ 간단한 작업

---

## 🪝 Hooks (고급 제어)

### Subagent 레벨 Hooks

Subagent 실행 중에만 작동

```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-command.sh"
```

### 세션 레벨 Hooks (settings.json)

Subagent 시작/종료 감지

```json
{
  "hooks": {
    "SubagentStart": [{"matcher": "db-agent", "hooks": [...]}],
    "SubagentStop": [{"hooks": [...]}]
  }
}
```

---

## 🔒 Subagent 비활성화

### 특정 Subagent 차단

```json
{
  "permissions": {
    "deny": ["Agent(Explore)", "Agent(my-custom-agent)"]
  }
}
```

또는 CLI:

```bash
claude --disallowedTools "Agent(Explore)"
```

---

## 📚 실전 예시

### Code Reviewer

```yaml
---
name: code-reviewer
description: Expert code review. Use proactively after code changes.
tools: Read, Grep, Glob, Bash
model: sonnet
---

You are a senior code reviewer. Focus on:
1. Code clarity and readability
2. Security issues
3. Performance
4. Best practices

Organize feedback by priority:
- Critical issues
- Warnings
- Suggestions
```

### Debugger

```yaml
---
name: debugger
description: Fix errors and test failures
tools: Read, Edit, Bash, Grep
---

You are an expert debugger. Process:
1. Capture error and stack trace
2. Identify reproduction steps
3. Isolate failure location
4. Implement minimal fix
5. Verify solution
```

### Database Query Validator

```yaml
---
name: db-reader
description: Execute read-only database queries
tools: Bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-readonly-query.sh"
---

You are a database analyst with read-only access.
Execute SELECT queries. Block INSERT/UPDATE/DELETE/DROP.
```

---

## 🎓 핵심 요약

|개념|설명|
|---|---|
|**격리**|각 Subagent는 독립된 컨텍스트 = 메인 오염 X|
|**전문화**|description과 tools로 역할 명확히|
|**재사용성**|반복되는 작업은 Subagent로 정의|
|**도구 제어**|tools/disallowedTools로 권한 제한|
|**메모리**|memory 옵션으로 학습 축적|
|**자동화**|조건 맞으면 Claude가 자동 위임|
|**병렬화**|여러 Subagent 동시 실행 가능|

---

## 🔗 다음 학습

- **Agent Teams:** 여러 Agent가 독립 컨텍스트에서 통신 (더 진화된 형태)
- **Skills:** 메인 컨텍스트에서 재사용 프롬프트
- **Plugins:** Subagent 팀 공유/배포
- **MCP Servers:** Subagent에 외부 도구 연결