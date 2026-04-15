
### 1. Serverless 개념

서버가 없는 게 아니라 **서버를 관리하지 않는** 것. AWS의 서버리스 서비스들: Lambda, DynamoDB, Cognito, API Gateway, S3, SNS/SQS, Kinesis Firehose, Aurora Serverless, Step Functions, Fargate.

---

### 2. AWS Lambda ⭐⭐⭐

**EC2 vs Lambda:**

![](../../images/스크린샷%202026-04-12%2020.40.20.png)

**Lambda 한도 — 숫자 암기 필수 ⭐:**

|항목|한도|
|---|---|
|메모리|128MB ~ **10GB**|
|최대 실행 시간|**900초 (15분)**|
|환경 변수|**4KB**|
|/tmp 디스크|512MB ~ 10GB|
|동시 실행|**1,000회** (증가 요청 가능)|
|배포 zip 크기|**50MB** (압축), **250MB** (비압축)|

**RAM 늘리면 CPU + 네트워크 성능도 함께 향상.**

지원 언어: Node.js, Python, Java, C#, Go, Ruby, Rust (Custom Runtime API).

---

### 3. Lambda@Edge vs CloudFront Functions ⭐⭐

엣지 로케이션에서 코드를 실행해 지연 시간 최소화.
![](../../images/스크린샷%202026-04-12%2020.40.33.png)


**CloudFront Functions 용도**: 캐시 키 정규화, 헤더 조작, URL 재작성, JWT 검증.  
**Lambda@Edge 용도**: 외부 서비스 연동, 긴 처리, 파일 접근, 대규모 데이터 통합.

---

### 4. Lambda in VPC ⭐⭐

기본적으로 Lambda는 **AWS 소유 VPC**에서 실행 → 사용자 VPC 내 RDS, ElastiCache 등에 **직접 접근 불가**.

해결책: Lambda를 사용자 VPC에서 실행 → VPC ID, 서브넷, 보안 그룹 지정 → Lambda가 서브넷에 **ENI(Elastic Network Interface)** 생성.

**Lambda + RDS Proxy 패턴 ⭐:** Lambda가 RDS에 직접 연결하면 연결 폭증으로 타임아웃 발생. **RDS Proxy**를 사이에 두면 연결 풀링으로 해결. RDS Proxy는 퍼블릭 접근 불가 → **Lambda 반드시 VPC 배포 필요**.

---

### 5. Amazon DynamoDB ⭐⭐

완전 관리형 **NoSQL** DB. 멀티 AZ 복제, 한 자릿수 밀리초 성능, IAM 통합.

- 항목 최대 크기: **400KB**
- 스키마를 빠르게 바꿔야 할 때 RDS/Aurora보다 적합

**Read/Write Capacity Modes:**

| 모드                   | 특징                        | 비용                  |
| -------------------- | ------------------------- | ------------------- |
| **Provisioned (기본)** | 사전에 RCU/WCU 지정, 오토스케일링 가능 | 프로비저닝 용량 기준         |
| **On-Demand**        | 자동 확장, 용량 계획 불필요          | 더 비쌈, 예측 불가 트래픽에 적합 |

**DAX (DynamoDB Accelerator):** DynamoDB 전용 인메모리 캐시. **마이크로초** 지연. 앱 코드 변경 불필요. 기본 TTL 5분.

- DAX: 개별 객체/쿼리/스캔 캐시 → DynamoDB 앞에 배치
- ElastiCache: 집계 결과 저장 → DAX와 상호 보완적

**DynamoDB Streams:** 테이블 변경(생성/수정/삭제) 이벤트를 실시간으로 스트리밍.

- DynamoDB Stream: 보존 24시간, 소비자 수 제한
- Kinesis Data Streams로 연결 시: 보존 **1년**, 더 많은 소비자

**Global Tables:** 멀티 리전 **active-active** 복제. 어느 리전에서든 읽고 쓰기 가능. 활성화하려면 **DynamoDB Streams 먼저 활성화** 필요.

**TTL:** 만료 타임스탬프 지난 항목 자동 삭제. 웹 세션, 규정 준수 데이터 관리에 활용.

**백업:**

- PITR: 최근 35일, 선택적 활성화, 복구 시 새 테이블 생성
- On-demand: 장기 보관, 성능 영향 없음

---

### 6. AWS API Gateway ⭐⭐

Lambda + API Gateway = **완전 서버리스 REST API**.

**주요 기능:** WebSocket 지원, API 버전/환경 관리, API 키, 요청 스로틀링, Swagger/OpenAPI 통합, 응답 캐싱.

**통합 대상:** Lambda, HTTP 백엔드(ALB, 온프레미스), AWS 서비스(SQS, Step Functions 등).

**Endpoint Types:**

|타입|설명|
|---|---|
|**Edge-Optimized (기본)**|글로벌 클라이언트용, CloudFront 경유|
|**Regional**|같은 리전 클라이언트용, 수동으로 CloudFront 결합 가능|
|**Private**|VPC 내부에서만 접근 (ENI 사용)|

**보안:**

- IAM Roles: 내부 앱
- Cognito: 외부/모바일 사용자
- Custom Authorizer: 자체 로직

**HTTPS + Custom Domain:** ACM 인증서 사용. Edge-Optimized이면 인증서는 **us-east-1**에 있어야 함. Regional이면 해당 리전에.

---

### 7. AWS Step Functions ⭐

Lambda 함수들을 시각적으로 **조율(orchestration)**하는 서버리스 워크플로우 서비스. 시퀀스, 병렬, 조건, 타임아웃, 오류 처리 지원. Lambda 외에도 EC2, ECS, SQS, API Gateway 등과 통합. **사람 승인(Human Approval) 단계** 설정 가능.

---

### 시험 TIP 최종 정리

- **Lambda 최대 실행 시간: 15분** → 그 이상이면 ECS/Fargate/Batch 사용
- **Lambda RAM 늘리면 CPU도 향상** — 성능 튜닝 포인트
- **Lambda + RDS 연결 폭증** → **RDS Proxy** 사용
- **CloudFront Functions**: JS, 1ms, 수백만/s, 뷰어 요청/응답만
- **Lambda@Edge**: Node/Python, 밀리초, 4가지 수정, us-east-1에서 작성
- **DynamoDB 항목 최대: 400KB**
- **DynamoDB Global Tables**: active-active, Streams 활성화 필수
- **API Gateway Edge-Optimized + HTTPS**: 인증서는 **us-east-1**
- **Step Functions**: Lambda 조율, 시각적 워크플로우, 사람 승인 가능