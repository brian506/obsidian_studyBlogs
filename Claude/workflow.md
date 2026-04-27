# Claude Code 공통 워크플로우 (Common Workflows)

이 문서는 Claude Code를 활용하여 낯선 코드베이스를 탐색하고, 버그를 수정하며, 리팩토링, 테스트 작성, PR 생성, 세션 관리 등 일상적인 개발 작업을 수행하기 위한 실용적인 워크플로우를 다룹니다.

---

## 1. 새로운 코드베이스 파악하기 (Understand new codebases)

새로운 프로젝트에 합류했을 때 구조를 빠르게 이해하고 관련 코드를 탐색하는 방법입니다.

### 빠른 코드베이스 개요 얻기
1. 프로젝트 루트 디렉토리로 이동: `cd /path/to/project`
2. Claude Code 실행: `claude`
3. 개요 요청: `give me an overview of this codebase` (이 코드베이스의 전반적인 개요를 알려줘)
4. 특정 컴포넌트 심층 분석:
   * `explain the main architecture patterns used here` (사용된 주요 아키텍처 패턴 설명)
   * `what are the key data models?` (핵심 데이터 모델은 무엇인지 확인)
   * `how is authentication handled?` (인증 처리 방식 확인)

**💡 팁:** 넓은 범위의 질문으로 시작해서 특정 영역으로 좁혀가세요. 프로젝트 특화 용어집을 요청하는 것도 좋은 방법입니다.

### 관련 코드 찾기
특정 기능이나 로직이 구현된 위치를 찾고 흐름을 이해합니다.
1. 관련 파일 찾기: `find the files that handle user authentication`
2. 상호작용 이해: `how do these authentication files work together?`
3. 실행 흐름 추적: `trace the login process from front-end to database`

---

## 2. 버그 효율적으로 수정하기 (Fix bugs efficiently)

에러 메시지를 바탕으로 원인을 찾고 수정 사항을 적용합니다.

1. 에러 공유: `I'm seeing an error when I run npm test` (npm test 실행 시 발생하는 에러 공유)
2. 수정 방안 요청: `suggest a few ways to fix the @ts-ignore in user.ts`
3. 수정 적용: `update user.ts to add the null check you suggested`

**💡 팁:** 에러를 재현하는 명령어와 스택 트레이스를 함께 제공하세요. 재현 단계나 간헐적/지속적 발생 여부를 알려주면 더 정확한 해결책을 얻을 수 있습니다.

---

## 3. 코드 리팩토링 (Refactor code)

레거시 코드를 최신 패턴과 모범 사례로 업데이트합니다.

1. 리팩토링 대상 식별: `find deprecated API usage in our codebase` (사용 중단된 API 찾기)
2. 리팩토링 제안 받기: `suggest how to refactor utils.js to use modern JavaScript features`
3. 안전하게 변경 적용: `refactor utils.js to use ES2024 features while maintaining the same behavior`
4. 검증: `run tests for the refactored code` (리팩토링된 코드에 대한 테스트 실행)

---

## 4. 특화된 서브에이전트 사용 (Use specialized subagents)

특정 작업을 더 효과적으로 처리하기 위해 AI 서브에이전트를 활용합니다.

* **사용 가능한 에이전트 확인:** `/agents` 명령어 사용
* **자동 위임:** Claude Code가 작업에 맞는 에이전트를 자동 선택 (`review my recent code changes for security issues`)
* **명시적 지정:** `use the code-reviewer subagent to check the auth module`
* **커스텀 에이전트 생성:** `/agents` 입력 후 "Create New subagent" 선택. 역할, 허용 도구, 시스템 프롬프트를 커스터마이징 가능.

---

## 5. 플랜 모드 (Plan Mode)를 활용한 안전한 코드 분석

플랜 모드는 코드를 직접 수정하지 않고(Read-only) 분석 및 계획 수립만 진행하는 모드입니다. 복잡한 변경을 기획하거나 코드를 탐색할 때 안전하게 사용할 수 있습니다.

* **사용 시기:** 여러 파일에 걸친 변경, 코드 탐색, Claude와의 방향성 반복 논의 등
* **전환 방법:** 세션 중 `Shift+Tab`을 눌러 권한 모드 전환 (`⏸ plan mode on`)
* **새 세션 시작:** `claude --permission-mode plan`
* **예시:** `claude --permission-mode plan -p "Analyze the authentication system and suggest improvements"`

---

## 6. 테스트 작업 (Work with tests)

테스트가 누락된 코드를 찾고 새로운 테스트 케이스를 생성합니다.

1. 미테스트 코드 식별: `find functions in NotificationsService.swift that are not covered by tests`
2. 테스트 스캐폴딩 생성: `add tests for the notification service`
3. 의미 있는 케이스 추가: `add test cases for edge conditions in the notification service`
4. 실행 및 검증: `run the new tests and fix any failures`

---

## 7. PR(Pull Request) 생성 및 문서화 (PRs & Documentation)

* **PR 생성:** `summarize the changes I've made...` -> `create a pr` -> `enhance the PR description...` 순으로 진행. `gh pr create` 사용 시 세션이 PR에 자동 연결됩니다.
* **문서화:** 주석이 없는 코드 식별 (`find functions without proper JSDoc...`) 후 주석 생성을 요청하고 프로젝트 표준에 맞는지 검증합니다.

---

## 8. 이미지 워크플로우 (Work with images)

UI 디자인, 에러 스크린샷, 아키텍처 다이어그램 등을 분석하거나 기반으로 코드를 작성할 수 있습니다.

* **이미지 추가:** 드래그 앤 드롭, 클립보드 복사(ctrl+v), 또는 파일 경로 입력 (`Analyze this image: /path/to/image.png`)
* **활용 예시:**
  * `What does this image show?`
  * `Generate CSS to match this design mockup` (디자인 시안을 바탕으로 CSS 생성)
  * `Here's a screenshot of the error. What's causing it?`

---

## 9. 파일 및 디렉토리 참조 (@ 문법)

Claude가 파일을 직접 읽을 때까지 기다릴 필요 없이 `@`를 사용하여 컨텍스트를 즉시 포함할 수 있습니다.

* **단일 파일:** `Explain the logic in @src/utils/auth.js`
* **디렉토리 구조:** `What's the structure of @src/components?`
* **MCP 리소스:** `Show me the data from @github:repos/owner/repo/issues`

---

## 10. 확장된 사고 (Thinking Mode) 활용

Claude가 복잡한 문제를 단계별로 추론할 수 있도록 하는 기능입니다.

* **기본 설정:** 활성화되어 있으며, 복잡한 아키텍처 결정이나 난해한 버그 추적에 유용.
* **단축키:** `Ctrl+O`로 Verbose 모드를 켜서 추론 과정 확인 가능.
* **토글:** `Option+T` (Mac) 또는 `Alt+T` (Windows/Linux)로 세션 중 켜고 끄기 가능.
* **프롬프트 강제 지시:** 프롬프트 내에 "ultrathink" 키워드 포함.

---

## 11. 이전 세션 이어서 작업하기 (Resume sessions)

작업 컨텍스트를 유지하면서 이전 대화로 돌아갑니다.

* `claude --continue`: 현재 디렉토리의 가장 최근 세션 이어가기
* `claude --resume`: 세션 선택기(Picker) 열기 또는 이름으로 재개
* `claude --from-pr 123`: 특정 PR 번호와 연결된 세션 재개
* 세션 내부에서는 `/resume` 명령어로 다른 대화로 전환 가능.