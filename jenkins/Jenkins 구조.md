#Jenkins

![사진](../images/jenkins1.jpg)

## Controller
### 역할  
- Jenkins 시스템의 중앙 제어 서버 역할
- 작업을 관리하고 스케쥴링하며, Agent 노드 모니터링
### 주요 기능 
- 어떤 프로젝트(job)을 빌드하고 배포할지(JenkinsFile 등) 저장
- Git Push 와 같은 트리거가 발생하면 어떤 작업을 실행할지 결정
- Agent 노드에게 빌드,테스트 등에 대한 작업 지시
- Agent 로부터 작업 결과물 가져와 UI에 표시

## Agent

### 역할
- Controller 부터 명령을 받아 실제 작업(빌드,테스트,배포)을 하는 역할
- 작업이 완료되면 Controller 에게 결과 전달
### 주요 기능
- 컨트롤러와는 다른 서버(물리 장비, 가상 머신, Docker 컨테이너 등)에서 실행
- 컨트롤러의 지시에 따라 Git 소스코드를 내려받고, 컴파일, 테스트, Docker 이미지 빌드, 배포 스크립트 실행 등 모든 실질적인 작업을 처리

## 작업 흐름

1. **트리거(trigger)** : 개발자가 Git에 코드를 푸시하면, Webhook을 통해 젠킨스 **Controller**에 알림
2. **파이프라인 시작** : Controller 는 Git 저장소에 연결된 파이프라인을 찾아 실행
3. **Agent 할당** : Controller 는 JenkinsFile 에 명시된 Agent 조건을 확인하고, 유휴 상태인 Agent 에게 작업 할당
4. **작업공간(workspace)준비** : Agent 는 해당 작업을 위한 임시폴더를 생성
5. **명령 실행** : Controller 가 Agent 에게 JenkinsFile의 작업 Stage(순서)를 보냄
	-  `Checkout`: Agent가 Git 코드를 Workspace로 내려받음
	-  `Build`: Agent가 Workspace에서 Gradle/Maven 빌드를 실행
    -  `Deploy`: Agent가 Docker 이미지를 빌드하고, 새로운 컨테이너를 실행
6. **결과 전송** : Agent 는 모든 결과물을 Controller 에게 전송
7. **완료** : Controller 는 최종 결과를 UI 에 표기하고 파이프라인 종료

