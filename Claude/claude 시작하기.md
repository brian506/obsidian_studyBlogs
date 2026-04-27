## Command, Agent, Skill의 차이

### Command
> 내가 부를 떄 실행되는 프롬프트 템플릿
- 메인 컨텍스트와 공유

`.claude/commands/pr-review.md`
-> PR을 분석하고 보안 이슈, 로직 버그, 코드 스타일을 점검해라

- 사람이 `/pr-review` 를 직접 입력해서 실행
- 현재 대화 컨텍스트 안에서 바로 동작
- 반복되는 워크플로우를 명령어로 등록해두는 것

### Agent
> 독립된 방에서 일하는 별도 AI 인스턴스
- 완전 격리된 새 컨텍스트

`.claude/agents/qa-tester.md`
1. 메인 Claude : "qa-tester"에게 테스트 맡겨
2. Subagent : 격리된 컨텍스트에서 테스트 실행
3. 결과만 메인 컨텍스트로 반환

- Claude 가 자동으로 판단하거나, 메인이 명시적으로 위임
- 격리된 새 컨텍스트에서 실행됨 -> 메인 컨텍스트를 오염시키지 않음
- 작업이 끝나면 결과만 메인으로 반환하고 자신의 메모리는 버림

### Skill
> Claude 가 스스로 찾아 읽는 참고 문서
- 내용이 현재 컨텍스트에 주입

`.claude/skills/deploy/SKILL.md`
```yml
name: deploy 
description: 사용자가 "배포해줘", "커밋해줘", "deploy" 라고 하면 실행. 
--- 
# Deploy Pipeline 
Step 1: 빌드 & 테스트 
Step 2: 커밋 & 푸시
```

- description 필드가 트리거 역할을 함. 
- 개발자가 호출하지 않아도 됨
- 로드된 내용은 현재 컨텍스트에 주입되어 Claude가 참고

|상황|선택|
|---|---|
|매일 반복하는 작업을 자동화|Command|
|복잡한 작업을 메인 흐름과 분리|Agent|
|특정 도메인 지식/절차를 Claude에게 가르치기|Skill|
|간단한 단일 작업|셋 다 불필요|

**토큰 사용은 메인 컨텍스트 작업에서 이전 작업이 중첩되면서 토큰 사용량이 늘어나는 것**
- 로그가 많이 발생하거나, 중간에 개발자가 개입해야 한다면 Agent를 사용하여 메인 컨텍스트와 분리해서 실행할 것

## CLAUDE.md

**포함해야 하는 요소들**
- 프로젝트 설명
- 빌드/실행 명령어
- 스택(언어, DB, 라이브러리 등)
- 프로젝트 구조
- 자주 하는 실수
- 환경 설정
- 주의사항 
- 외부 API/서비스
-> 최대한 500자 이내로

**넣으면 안되는 것들**
- 비밀번호, API 키
- 매우 상세한 코드 예제
- 히스토리, 회의록

```
# 프로젝트명 
## 개요 한두 문장으로 이 프로젝트가 뭔지 설명 
## 빌드 & 실행 
### 개발 환경 준비 
\`\`\`bash docker compose up -d ./gradlew build ./gradlew test \`\`\` 
## 스택 Spring Boot 4.0.3 · Java 21 · PostgreSQL 16 
## 프로젝트 구조 
## 아키텍처 규칙 
Controller → Business → Implement → DataAccess Business에서 Repository 직접 참조 금지 
## 자주 하는 실수 
- QueryDSL Q클래스 없으면: ./gradlew compileQuerydsl - Bean 등록 안 되면: @Component 확인 
## 외부 서비스 
-CoolSMS: 문자 발송 - Redis: 캐싱 
## 참고 문서 
- 아키텍처: .claude/skills/feature-dev/references/architecture.md 
- 배포: .claude/skills/deploy/SKILL.md
```
