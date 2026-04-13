### 1. Amazon CloudWatch ⭐⭐

**CloudWatch Metrics:**

- AWS 모든 서비스의 지표(Metric) 제공. 네임스페이스에 속하며 타임스탬프 필수
- Dimension(측정 기준) 최대 **10개**
- **사용자 지정 지표** 생성 가능 (예: RAM — EC2 기본 지표에는 RAM 없음)

**Metric Streams:** Kinesis Firehose 또는 Datadog, Splunk 등 외부 서비스로 거의 실시간 스트리밍 가능. 일부 지표만 필터링 가능.

---

**CloudWatch Logs 구조:**

- **Log Group**: 애플리케이션 단위 (보존 기간, 액세스 제어 설정 공유)
- **Log Stream**: 인스턴스/컨테이너 단위 (동일 소스의 이벤트 시퀀스)
- 로그 전송 대상: S3, Kinesis Data Streams, Kinesis Firehose, Lambda, OpenSearch

**CloudWatch Logs 소스:** SDK, EC2 에이전트, Lambda, ECS, Elastic Beanstalk, VPC Flow Logs, API Gateway, Route 53, CloudTrail 등

**S3 Export vs Subscription:**

- S3 Export: **최대 12시간** 소요, 실시간 아님. API = `CreateExportTask`
- **실시간으로 스트리밍**하려면 → **구독 필터(Subscription Filter)** 사용

---

**CloudWatch Logs Agent vs Unified Agent ⭐:**

![](../../images/스크린샷%202026-04-13%2014.35.20.png)

> EC2 기본 지표에는 RAM이 없음 → Unified Agent로 커스텀 지표 수집 필요

---

**CloudWatch Alarms:**

- 상태: **OK / ALARM / INSUFFICIENT_DATA**
- 주요 타겟 3가지: EC2 인스턴스(중지/재부팅/복구), EC2 Auto Scaling, **SNS**
- **EC2 Instance Recovery**: 경보 위반 시 다른 호스트로 이동. 동일 사설IP/공인IP/EIP/메타데이터/배치 그룹 유지
- 경보 테스트: `aws cloudwatch set-alarm-state` CLI 명령어로 강제 트리거 가능
- **경보는 Metric Filter에 기반해 생성 가능**

---

### 2. Amazon EventBridge ⭐⭐

(구 이름: CloudWatch Events)

- **Schedule**: CRON 표현식으로 주기적 작업 예약
- **Event Pattern**: 특정 서비스 이벤트에 반응 (IAM 로그인 실패, EC2 상태 변경 등)
- 다양한 대상에 Lambda, SQS, SNS 등 트리거

**Event Bus:**

- Default Event Bus (AWS 서비스), Partner Event Bus (Zendesk 등), Custom Event Bus
- 이벤트 **아카이브** + **재생** 기능 ← 장애 재현, 디버깅에 유용
- 리소스 기반 정책으로 다른 계정/리전 이벤트 수신 가능

**Schema Registry:** 이벤트 버스의 이벤트 구조를 분석해 스키마 자동 추론. 코드 생성 지원, 버전 관리 가능.

---

### 3. CloudWatch Insights 4종류 ⭐

|Insight|용도|대상|
|---|---|---|
|**Container Insights**|컨테이너 지표+로그 수집/집계|ECS, EKS, Fargate, K8s on EC2|
|**Lambda Insights**|Lambda 성능 모니터링|Lambda (Lambda 계층으로 제공)|
|**Contributor Insights**|Top-N 기여자 분석|VPC Flow Logs, DNS 등 모든 로그|
|**Application Insights**|앱 + 연관 서비스 자동 대시보드|EC2 기반 앱 (Java, .NET 등)|

- Contributor Insights: 불량 호스트 식별, 트래픽 최다 사용자, 오류 최다 URL 파악
- Application Insights: SageMaker ML 내부 사용, 문제 알림은 EventBridge + SSM OpsCenter로 전송

---

### 4. AWS CloudTrail ⭐⭐⭐

- **기본 활성화**, 계정 내 모든 API 호출 기록 (콘솔, SDK, CLI, AWS 서비스 포함)
- 로그 저장: CloudWatch Logs 또는 **S3**
- 전체/단일 리전 Trail 생성 → 멀티 리전 이벤트 한곳으로 집계 가능
- **리소스 삭제됐을 때 → CloudTrail로 먼저 조사!**

**이벤트 3종류:**

|이벤트 유형|기본 로깅|내용|
|---|---|---|
|**Management Events**|✅ 기본 활성화|리소스 생성/삭제/수정 (IAM, EC2, CloudTrail 등)|
|**Data Events**|❌ 기본 비활성|S3 객체 수준 (GetObject, PutObject 등), Lambda Invoke|
|**Insights Events**|❌ 별도 활성|비정상 활동 감지 (IAM 급증, 한도 초과 등)|

**CloudTrail Insights:** 정상 패턴 기준선 생성 후 쓰기 이벤트 지속 분석. 이상 감지 시 콘솔 표시, S3 전송, EventBridge 이벤트 생성.

**이벤트 보존:** 기본 **90일** → 장기 보존 필요 시 **S3로 전송 + Athena**로 분석

---

### 5. AWS Config ⭐⭐

- AWS 리소스의 **규정 준수(Compliance)** 여부 기록 및 평가
- 리전별 서비스 (멀티 리전 설정 필요), 구성 데이터 S3 저장 + Athena 분석 가능
- **Config Rules는 동작을 예방하지 못함** — 평가/보고만. 거부 기능 없음!
- 프리 티어 없음: 구성 항목당 $0.003 / 규칙 평가당 $0.001

**Config Rules:**

- AWS 관리형 규칙 75개 이상 / Lambda로 커스텀 규칙 생성 가능
- 트리거: 구성 변경 시 또는 주기적으로

**Remediations (자동 수정):**

- SSM 자동화 문서로 미준수 리소스 자동 수정
- 수정 후에도 미준수이면 **Remediation Retries** 설정 가능

**Notifications:**

- 미준수 발생 → **EventBridge** 알림
- 구성 변경/준수 상태 → **SNS** 알림

---

### 6. CloudWatch vs CloudTrail vs Config ⭐⭐⭐ (초빈출 비교)


**ALB 예시로 기억하기:**

- **CloudWatch**: 연결 수, 오류 코드 비율, 성능 대시보드
- **CloudTrail**: 누가 ALB 설정을 변경했는가
- **Config**: ALB 보안 그룹 규칙 변경 이력, SSL 인증서 항상 할당 규칙

---

### 시험 TIP 최종 정리

- **EC2 RAM 지표** → 기본 없음, **Unified Agent** 필요
- **CloudWatch Logs → S3 Export**: 최대 12시간 소요, **실시간 아님**
- **실시간 로그 스트리밍** → **Subscription Filter**
- **CloudTrail 기본 보존: 90일** → 장기 보존 = S3 + Athena
- **Data Events 기본 비활성** (S3 객체 수준, Lambda Invoke)
- **Config Rules = 예방 불가, 평가/보고만** (거부 기능 없음)
- **리소스 삭제 원인 조사** → **CloudTrail**
- **구성이 언제 바뀌었는지** → **Config**
- **성능/트래픽 모니터링** → **CloudWatch**