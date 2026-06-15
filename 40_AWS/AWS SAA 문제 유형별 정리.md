

## 1. 스토리지 클래스 선택 (S3 Tier)

### 핵심 판단 기준

|조건|정답|
|---|---|
|접근 패턴 예측 불가|S3 Intelligent-Tiering|
|30일 후 드물게 접근, 즉시 복원|S3 Standard-IA|
|단일 AZ 허용, 드물게 접근|S3 One Zone-IA|
|분기에 한 번, 밀리초 복원|S3 Glacier Instant Retrieval|
|장기 아카이브, 몇 시간 복원 허용|S3 Glacier Deep Archive|
|처음 30일 자주 → 이후 4년 유지|Standard → Standard-IA (수명 주기)|

### 자주 나오는 오답 패턴

- "예측 불가"인데 Standard-IA 고름 → 검색 비용 과다 발생
- "즉시 복원 필요"인데 Glacier Flexible 고름 → 복원 시간 수 시간
- "단일 AZ" 옵션은 가용성 요건 없을 때만 선택 가능

---

## 2. 데이터 전송 방식 선택

### 핵심 판단 기준

|데이터 크기 / 조건|정답|
|---|---|
|수십 GB ~ 수 TB, 인터넷 가능|S3 Transfer Acceleration + 멀티파트 업로드|
|10TB ~ 80TB, 인터넷 한계|Snowball Edge (물리 배송)|
|수십 PB 이상|Snowmobile|
|지속적 실시간 동기화|DataSync|
|인터넷 대역폭 최소화 + 안정적 전용 연결|Direct Connect|
|인터넷 대역폭 문제 없음, 전 세계 사이트 → S3 빠르게|S3TA + 멀티파트|

### 자주 나오는 오답 패턴

- "네트워크 대역폭 없음" 상황에서 DataSync 고름 → DataSync는 네트워크 필요
- Snowball은 Glacier에 직접 전송 불가 → S3 경유 후 수명 주기 정책으로 Glacier 이동
- "빠른 동기화" = DataSync / "일회성 대용량 오프라인" = Snowball 구분 필요

---

## 3. 스토리지 아키텍처 (EBS / EFS / Instance Store / FSx)

### 핵심 판단 기준

|요건|정답|
|---|---|
|여러 EC2가 동시 공유, Linux|EFS|
|여러 EC2가 동시 공유, Windows/SMB|FSx for Windows File Server|
|HPC, 병렬 처리, S3 통합|FSx for Lustre|
|단일 EC2 전용 영구 저장|EBS|
|최고 I/O 성능, 임시 데이터|EC2 Instance Store|
|온프레미스 NFS → S3 캐싱|S3 File Gateway|
|온프레미스 전체 데이터 로컬 유지 + S3 백업|Volume Gateway (Stored 모드)|

### 자주 나오는 오답 패턴

- "여러 AZ에 걸친 EC2 공유" → EBS 불가 (단일 AZ 종속)
- EBS FSR(빠른 스냅샷 복원) 미설정 시 → Lazy Loading으로 초기 I/O 지연
- Instance Store는 재시작/종료 시 데이터 소멸 → 영구 저장 목적에 부적합

---

## 4. 메시지 / 이벤트 아키텍처 (SQS / SNS / Kinesis / EventBridge)

### 핵심 판단 기준

|요건|정답|
|---|---|
|순서 보장, 정확히 한 번 처리|SQS FIFO|
|하나의 이벤트 → 여러 서비스 동시 전달|SNS + SQS Fan-out|
|실시간 스트리밍, 멀티 컨슈머|Kinesis Data Streams|
|스트리밍 → S3/Redshift로 적재|Kinesis Data Firehose|
|스케줄링 기반 작업 실행|EventBridge + Lambda|
|S3 이벤트 → 여러 서비스 동시 라우팅|S3 → EventBridge|
|기존 MQTT/AMQP 브로커 마이그레이션|Amazon MQ|

### 자주 나오는 오답 패턴

- "중복 메시지" 문제: SQS 표준 큐에서 처리 시간 > 가시성 제한 시간이면 재노출 → 가시성 제한 시간 연장
- SNS 단독으로는 순서 보장 안 됨 (SNS FIFO + SQS FIFO 필요)
- Kinesis Data Analytics는 Firehose 데이터 직접 읽기 불가
- "결합도 낮추기" = SQS or SNS / "실시간 분석" = Kinesis 구분

---

## 5. 데이터베이스 선택 (RDS / Aurora / DynamoDB / ElastiCache)

### 핵심 판단 기준

|요건|정답|
|---|---|
|관계형, 읽기 부하 분산|RDS Read Replica|
|관계형, 장애 복구(Failover)|RDS Multi-AZ|
|관계형, 고성능 읽기 + 자동 확장|Aurora + Auto Scaling|
|예측 불가 서버리스 워크로드|Aurora Serverless|
|OS 직접 접근 필요|RDS Custom|
|밀리초 NoSQL, 서버리스|DynamoDB|
|DynamoDB 읽기 캐시 (코드 변경 없이)|DAX|
|일반 DB 쿼리 캐시 (코드 변경 필요)|ElastiCache Redis|
|고가용성 캐시, 복잡한 데이터 타입|ElastiCache Redis|
|단순 분산 캐시|ElastiCache Memcached|

### 자주 나오는 오답 패턴

- "재해 복구" 목적에 Read Replica 고름 → Read Replica는 비동기 복제, 자동 Failover 없음
- ElastiCache는 앱 코드 수정 필요 → "코드 변경 없이" 조건이면 DAX 또는 다른 접근
- "운영 DB 복사 즉시 스테이징" → Aurora DB Cloning (mysqldump는 오래 걸리고 부하 발생)
- RDS 암호화 전환 = 스냅샷 복사 시 암호화 활성화 → 새 인스턴스 복원 (직접 활성화 불가)

---

## 6. Lambda 연동 패턴

### 핵심 판단 기준

| 요건                           | 정답                                        |
| ---------------------------- | ----------------------------------------- |
| Lambda → RDS 연결 폭증           | RDS Proxy 추가                              |
| Lambda에 HTTPS 엔드포인트 (간단한 구조) | Lambda 함수 URL                             |
| Lambda에 HTTPS + 다양한 기능       | API Gateway + Lambda                      |
| Lambda → VPC 내 DB 접근         | Lambda를 VPC 서브넷에 배치                       |
| Lambda가 S3 업로드 트리거           | S3 이벤트 알림 → Lambda                        |
| EventBridge가 Lambda 호출       | Lambda 리소스 기반 정책에 events.amazonaws.com 허용 |

### 자주 나오는 오답 패턴

- Lambda 기본은 AWS 소유 VPC → 사용자 VPC의 RDS/ElastiCache 직접 접근 불가
- RDS Proxy는 VPC 내부 전용 → Lambda도 같은 VPC에 있어야 함
- Lambda 실행 역할(IAM Role) ≠ 리소스 기반 정책: 누가 Lambda를 호출하느냐는 리소스 기반 정책

---

## 7. 보안 서비스 선택

### 핵심 판단 기준

|요건|정답|
|---|---|
|암호 자동 교체 + DB 통합|Secrets Manager|
|구성값/암호 저장 (교체 불필요)|SSM Parameter Store|
|S3 PII 자동 탐지|Macie|
|이미지/비디오 부적절 콘텐츠 탐지|Rekognition|
|EC2 취약점 스캔|Inspector|
|계정 이상 활동 탐지|GuardDuty|
|SQL Injection, XSS, IP 차단|WAF|
|대규모 DDoS 방어|Shield Advanced|
|여러 계정 WAF 중앙 관리|Firewall Manager|
|VPC 트래픽 패킷 단위 필터링|Network Firewall|
|감사 목적 암호화 키 사용 로그|SSE-KMS + CloudTrail|

### 자주 나오는 오답 패턴

- GuardDuty = 탐지 서비스 (차단 불가) / WAF = 차단 가능
- Inspector = 소프트웨어 취약점 / GuardDuty = 비정상 행동 탐지
- Shield Standard = 무료 자동 / Shield Advanced = $3,000/월, DDoS 전문팀 지원
- 외부에서 가져온 인증서 → ACM이 자동 갱신 안 함 → EventBridge + Config로 만료 감지

---

## 8. 네트워크 / VPC 구성

### 핵심 판단 기준

| 요건                     | 정답                                      |
| ---------------------- | --------------------------------------- |
| 프라이빗 EC2 → S3 (인터넷 없이) | S3 Gateway VPC Endpoint (무료)            |
| 프라이빗 EC2 → 나머지 AWS 서비스 | Interface VPC Endpoint                  |
| 타사 방화벽 어플라이언스 경유       | Gateway Load Balancer                   |
| 다른 계정 VPC 특정 서비스만 연결   | AWS PrivateLink                         |
| 여러 VPC + 온프레미스 허브 연결   | Transit Gateway                         |
| 온프레미스 ↔ AWS 암호화 터널     | Site-to-Site VPN                        |
| 온프레미스 ↔ AWS 전용 물리선     | Direct Connect                          |
| VPN 장애 시 백업            | Site-to-Site VPN을 Direct Connect와 병렬 구성 |

### 자주 나오는 오답 패턴

- VPC Peering은 전이(Transitive) 불가 → A-B-C라도 A→C는 별도 피어링 필요
- ALB는 인터넷 연결형일 때 반드시 퍼블릭 서브넷에 배치 (EC2는 프라이빗 OK)
- WAF는 NLB(L4)에 직접 연결 불가 → ALB, API Gateway, CloudFront에만 적용
- `0.0.0.0/0` NACL 아웃바운드 차단 시 AWS 내부 통신도 끊김 주의

---

## 9. 로드 밸런서 / Auto Scaling

### 핵심 판단 기준

| 요건                      | 정답                           |
| ----------------------- | ---------------------------- |
| HTTP 라우팅, 컨테이너, 마이크로서비스 | ALB                          |
| 고성능, UDP, 고정 IP         | NLB                          |
| 타사 방화벽 앞에 배치            | GWLB                         |
| HTTP 오류 감지해서 자동 교체      | ALB + ASG (NLB는 L7 오류 감지 불가) |
| 특정 지표 목표값 유지 (CPU 40%)  | 대상 추적(Target Tracking) 스케일링  |
| 예측 가능한 시간대 사전 확장        | 예약(Scheduled) 스케일링           |
| 업무 시작 전 미리 확장           | 예약 스케일링 (동적 스케일링은 반응이 늦음)    |

### 자주 나오는 오답 패턴

- NLB는 HTTP 오류 코드 감지 불가 → ALB로 교체 후 상태 확인 활성화
- 예약 스케일링 ≠ 동적 스케일링: 시간 예측 가능하면 Scheduled, 불규칙이면 Target Tracking
- ASG + SQS 스케일링: 단순 메시지 수보다 인스턴스당 백로그 지표가 더 정확

---

## 10. 분석 서비스 선택 (Athena / Redshift / Glue / QuickSight)

### 핵심 판단 기준

| 요건                       | 정답                     |
| ------------------------ | ---------------------- |
| S3에서 서버리스 SQL 즉시 분석      | Athena                 |
| 대용량 복잡한 데이터 웨어하우스        | Redshift               |
| 서버리스 ETL, 데이터 카탈로그       | AWS Glue               |
| BI 대시보드 시각화              | QuickSight             |
| 데이터 레이크 중앙 관리, 행/열 접근 제어 | Lake Formation         |
| 텍스트 전문 검색, 비정형 데이터       | OpenSearch             |
| 실시간 스트림 SQL 분석           | Kinesis Data Analytics |

### 자주 나오는 오답 패턴

- Redshift는 Multi-AZ 미지원, 데이터 로드 필요 → 단순 S3 쿼리라면 Athena가 적합
- Kinesis Data Analytics(Flink)는 Firehose 데이터 직접 읽기 불가
- EMR은 빅데이터 Hadoop/Spark 클러스터 → 간단한 쿼리에는 과도한 선택

---

## 11. 재해 복구 (DR) 전략

### 핵심 판단 기준

| 요건                       | 정답                                        |
| ------------------------ | ----------------------------------------- |
| 가장 저렴, RPO/RTO 높음        | Backup & Restore                          |
| DB만 실시간 복제, 나머지 꺼둠       | Pilot Light                               |
| 최소 규모 전체 시스템 상시 운영       | Warm Standby                              |
| 두 리전 풀 프로덕션 동시 운영        | Hot Site / Multi-Site                     |
| 30분 이내 복구, 최소 데이터 손실     | Aurora 교차 리전 복제 + Route 53 Active-Passive |
| DR 리전 데이터 최신 + 인프라 축소 운영 | Aurora Global DB + Warm Standby           |

### 자주 나오는 오답 패턴

- 스냅샷 복원 = Backup & Restore → RPO는 스냅샷 간격만큼 데이터 손실 발생
- Active-Active vs Active-Passive 구분: 기본이 안 올 때만 전환 = Active-Passive
- RDS Multi-AZ ≠ 재해 복구 전략: 같은 리전 내 AZ 장애에만 대응

---

## 12. 접근 제어 / IAM / 권한 관리

### 핵심 판단 기준

| 요건                      | 정답                              |
| ----------------------- | ------------------------------- |
| 조직 전체 계정 S3 접근 제한       | 버킷 정책 + aws:PrincipalOrgID      |
| 여러 계정 전체 서비스 제한         | SCP (서비스 제어 정책)                 |
| EC2 → S3 접근 권한          | IAM 역할(Role)을 EC2에 연결           |
| AWS 계정 없는 사용자에게 대시보드 공유 | CloudWatch 대시보드 이메일 공유          |
| 암호화 키 관리 및 확장           | AWS KMS                         |
| EC2에 원격 접속 (키 없이)       | Systems Manager Session Manager |

### 자주 나오는 오답 패턴

- IAM 정책은 EC2에 직접 연결 불가 → IAM 역할(Role)을 통해 연결
- SCP는 최대 권한 정의 → 계정 관리자라도 SCP 범위 초과 불가
- 루트 사용자는 IAM 정책 적용 안 됨 → S3 객체 잠금 Compliance 모드로만 차단 가능

---

## 13. 콘텐츠 전송 / 글로벌 아키텍처

### 핵심 판단 기준

| 요건                         | 정답                              |
| -------------------------- | ------------------------------- |
| 정적+동적 콘텐츠 글로벌 배포           | CloudFront (S3 + ALB 오리진)       |
| 고정 IP + TCP/UDP + 빠른 장애 조치 | Global Accelerator              |
| 온프레미스 서버를 오리진으로            | CloudFront Custom Origin        |
| S3 직접 접근 차단                | OAI 또는 OAC 설정                   |
| 배포 후에도 예전 캐시가 보일 때         | CloudFront 캐시 무효화(Invalidation) |
| 특정 필드 엔드투엔드 암호화            | CloudFront 필드 수준 암호화            |
| DNS 없는 여러 인스턴스에 트래픽 균등 배포  | Route 53 Multi-Value 라우팅        |
| 사용자 위치 기반 라우팅              | Route 53 Geolocation            |
| 응답 속도 기반 라우팅               | Route 53 Latency                |

### 자주 나오는 오답 패턴

- CloudFront는 고정 IP 없음 → 고정 IP 필요 시 Global Accelerator
- Geolocation ≠ Latency: 위치 기반 vs 속도 기반 (같은 나라도 다른 리전이 더 빠를 수 있음)
- Route 53 Multi-Value는 ELB 대체제가 아님 → 헬스 체크 통과 IP만 반환하는 DNS 기능

---

## 14. 모니터링 / 감사 / 로그

### 핵심 판단 기준

| 요건               | 정답                                       |
| ---------------- | ---------------------------------------- |
| 리소스 구성 변경 이력 추적  | AWS Config                               |
| API 호출 이력 감사     | CloudTrail                               |
| 메트릭 수집 + 알람      | CloudWatch                               |
| 여러 지표 동시 조건 알람   | CloudWatch Composite Alarm               |
| 특정 포트 접속 감지      | VPC Flow Logs + CloudWatch Metric Filter |
| 태그 기반 리소스 전체 조회  | AWS Resource Groups Tag Editor           |
| EC2 RAM 지표 수집    | CloudWatch Unified Agent (기본 지표에 RAM 없음) |
| 비용 분석 (인스턴스 유형별) | Cost Explorer                            |
| 예산 초과 알림         | AWS Budgets                              |

### 자주 나오는 오답 패턴

- Config = 구성 추적/평가만 (예방 불가, 거부 기능 없음)
- CloudTrail 기본 보존 90일 → 장기 보존은 S3 + Athena
- CloudWatch 기본 EC2 지표에 RAM 없음 → Unified Agent로 커스텀 지표 수집 필요

---

## 15. 기타 특수 서비스

### 마케팅 / 커뮤니케이션

|요건|정답|
|---|---|
|양방향 SMS + 마케팅 이벤트 분석|Amazon Pinpoint|
|이메일/SMS 대량 발송 프로그래밍 방식|Amazon SES|
|음성 → 텍스트 (통화 분석)|Amazon Transcribe|
|텍스트 → 음성|Amazon Polly|
|의료 문서에서 PHI 추출|Textract + Comprehend Medical|

### 파일 처리

|요건|정답|
|---|---|
|SFTP 프로토콜로 S3 업로드|AWS Transfer Family|
|PDF/이미지에서 텍스트/테이블 추출|Amazon Textract|
|이미지 부적절 콘텐츠 감지|Amazon Rekognition|
|Java/PHP 웹앱 배포 + 잦은 기능 테스트|Elastic Beanstalk (URL 스와핑)|

### 자주 나오는 오답 패턴

- Transfer Family ≠ DataSync: Transfer Family는 SFTP 프로토콜 유지 / DataSync는 NFS, SMB 마이그레이션
- Pinpoint ≠ SNS: Pinpoint는 마케팅 여정, 양방향 SMS / SNS는 단방향 알림
- Textract는 단순 OCR이 아닌 구조적 데이터(표, 양식 필드)까지 인식