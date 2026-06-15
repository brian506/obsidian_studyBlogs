### 1. 핵심 용어 ⭐⭐

- **RPO (Recovery Point Objective)**: 얼마나 과거 시점으로 복구할 수 있는가 → **데이터 손실 허용 범위**
- **RTO (Recovery Time Objective)**: 재해 발생 후 복구까지 걸리는 시간 → **서비스 중단 허용 시간**
- RPO↓ + RTO↓ = 비용↑

---

### 2. DR 전략 4가지 ⭐⭐⭐

비용과 복구 속도 기준으로 4단계:

|전략|RPO|RTO|비용|핵심|
|---|---|---|---|---|
|**Backup & Restore**|높음 (시간~일 단위)|높음|가장 저렴|주기적 백업, 재해 시 복원|
|**Pilot Light**|중간|중간|저렴|핵심 DB만 상시 실행, 나머지는 재해 시 시작|
|**Warm Standby**|낮음|낮음|중간|최소 규모로 전체 시스템 상시 실행, 재해 시 확장|
|**Multi-Site / Hot Site**|최저|최저 (분~초)|가장 비쌈|풀 프로덕션 규모를 두 곳에서 동시 실행 (active-active)|

**각 전략 상세:**

- **Backup & Restore**: S3/Glacier/EBS 스냅샷/RDS 백업. 재해 시 AMI로 EC2 재생성. 쉽고 저렴하나 복구 시간 오래 걸림
- **Pilot Light**: RDS는 항상 복제 중 (DB만 준비). EC2는 재해 시 시작. Backup보다 RTO 낮음
- **Warm Standby**: EC2 ASG 최소 용량으로 가동 중. ELB 준비됨. 재해 시 Route 53으로 ELB에 장애 조치 + 스케일 업
- **Hot Site**: 온프레미스 + AWS 동시에 full 프로덕션. Route 53이 두 곳 모두에 요청 라우팅 (active-active)

---

### 3. DR Tips ⭐

**백업:** EBS 스냅샷, RDS 자동 백업, S3/S3-IA/Glacier, 교차 리전 복제, Snowball/Storage Gateway

**고가용성:** Route 53 다른 리전 페일오버, RDS Multi-AZ, ElastiCache Multi-AZ, EFS, S3, Direct Connect 장애 시 Site-to-Site VPN 백업

**복제:** RDS 교차 리전 복제, Aurora Global DB, Storage Gateway

**자동화:** CloudFormation/Elastic Beanstalk으로 환경 재생성, CloudWatch 알람으로 EC2 자동 복구

---

### 4. DMS (Database Migration Service) ⭐⭐

- 마이그레이션 중에도 **원본 DB 계속 사용 가능**
- 동종 마이그레이션: Oracle → Oracle / 이기종: SQL Server → Aurora
- CDC(Continuous Data Replication)로 지속적 복제 가능
- DMS 실행을 위해 **EC2 인스턴스 생성 필요**

**AWS Schema Conversion Tool (SCT):**

- 다른 엔진으로 마이그레이션 시 스키마 변환 필요
- **같은 엔진이면 SCT 불필요** (예: 온프레미스 PostgreSQL → RDS PostgreSQL)
- OLTP: SQL Server/Oracle → MySQL, PostgreSQL, Aurora
- OLAP: Teradata/Oracle → Redshift

---

### 5. RDS & Aurora 마이그레이션 ⭐

**RDS MySQL → Aurora MySQL:**

- 옵션 1: RDS 스냅샷 → Aurora 복원 (**다운타임 발생**)
- 옵션 2: Aurora 읽기 전용 복제본 생성 → 복제 지연 0이 되면 독립 클러스터로 승격 (시간+비용)
- 두 DB 동시 가동 중: **DMS 사용**

**외부 MySQL → Aurora MySQL:**

- 옵션 1: Percona XtraBackup → S3 → Aurora 생성 (빠름)
- 옵션 2: mysqldump로 직접 마이그레이션 (느림)

---

### 6. AWS Backup ⭐

완전 관리형 백업 서비스. 스크립트 없이 여러 AWS 서비스 백업을 중앙 관리.

지원 서비스: EC2, EBS, S3, RDS, Aurora, DynamoDB, DocumentDB, Neptune, EFS, FSx, Storage Gateway

- 리전 간/계정 간 백업 지원
- PITR 지원, 온디맨드/예약 백업
- 태그 기반 백업 정책

**Backup Vault Lock:**

- **WORM (Write Once Read Many)** 강제화
- 루트 사용자조차 삭제 불가
- 백업을 악의적 삭제로부터 보호

---

### 7. AWS Application Migration Service (MGN) ⭐

- SMS(Server Migration Service) 대체 서비스 (CloudEndure Migration의 후속)
- **Lift-and-shift (리호스팅)** 방식으로 온프레미스 → AWS 마이그레이션
- 물리/가상/클라우드 서버 모두 지원
- 최소 다운타임으로 마이그레이션

**AWS Application Discovery Service:**

- 온프레미스 서버 스캔 → 마이그레이션 계획 수립
- **Agentless**: VM 인벤토리, CPU/메모리/디스크 사용량
- **Agent-based**: 프로세스, 네트워크 연결 세부 정보
- 수집 데이터는 **AWS Migration Hub**에서 확인

---

### 8. 대용량 데이터 전송 비교 ⭐

200TB 데이터 전송 (100Mbps 인터넷) 기준:

|방법|소요 시간|특징|
|---|---|---|
|공용 인터넷/Site-to-Site VPN|**185일**|빠른 설치, 느린 전송|
|Direct Connect 1Gbps|**18.5일**|초기 설치 1개월 이상|
|Snowball (2~3개 병렬)|**약 1주일**|가장 빠름|

**지속적 복제/전송:** Site-to-Site VPN or Direct Connect + DMS or DataSync 조합

---

### 9. VMware Cloud on AWS

기존 VMware 환경을 그대로 유지하면서 AWS 확장성 활용. 사용 사례: VMware 워크로드 AWS 마이그레이션, 하이브리드 클라우드, DR 전략.

---

### 시험 TIP 최종 정리

- **RPO**: 데이터 손실 허용 범위 / **RTO**: 복구 시간 목표
- **가장 저렴한 DR** → Backup & Restore (RPO/RTO 높음)
- **가장 빠른 DR** → Hot Site / Multi-Site (비용 가장 높음)
- **DB만 미리 준비** → Pilot Light (RDS 상시 복제)
- **전체 최소 규모로 대기** → Warm Standby
- **DMS**: 마이그레이션 중 원본 DB 사용 가능, EC2 인스턴스 필요
- **다른 엔진으로 마이그레이션** → SCT 필요 / **같은 엔진이면 SCT 불필요**
- **Backup Vault Lock**: WORM, 루트도 삭제 불가
- **대용량 일회성 전송** → Snowball (1주일) > Direct Connect (18일) > VPN (185일)
- **온프레미스 → AWS Lift-and-shift** → **AWS MGN (Application Migration Service)**