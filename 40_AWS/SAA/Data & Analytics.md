### 1. Amazon Athena ⭐⭐⭐

- S3에 저장된 데이터를 분석하는 **서버리스 SQL 쿼리 서비스** (Presto 엔진 기반)
- 가격: **1TB당 $5**
- CSV, JSON, ORC, Avro, Parquet 지원
- **시험 TIP**: "서버리스 + S3 + SQL 분석" → **Athena**
- 사용 사례: BI, VPC Flow Logs, ELB Logs, CloudTrail 로그 분석, QuickSight와 연동

**성능 향상 방법:**

- **컬럼 기반 포맷** 사용 (Parquet, ORC) → 스캔량 감소, Glue로 변환
- **데이터 압축** (gzip, snappy 등)
- **파티셔닝**: S3 경로를 슬래시로 분할 (예: `year=1991/month=1/day=1/`)
- **큰 파일 사용** (128MB 이상)

**Federated Query**: Lambda 커넥터로 RDS, DynamoDB, CloudWatch Logs 등 다양한 소스를 SQL로 쿼리. 결과는 S3에 저장.

---

### 2. Redshift ⭐⭐

- PostgreSQL 기반이지만 **OLTP 아님 → OLAP** (분석/데이터 웨어하우징)
- **컬럼 기반** 저장, 병렬 쿼리 엔진, PB 단위 확장
- SQL 인터페이스 제공, QuickSight/Tableau 등 BI 도구와 통합
- **Athena와 차이**: 인덱스 덕에 복잡한 쿼리/조인/집계가 더 빠름

**Redshift Cluster 구조:**

- **리더 노드**: 쿼리 계획, 결과 집계
- **컴퓨트 노드**: 실제 쿼리 실행, 리더에 결과 전송
- 노드 크기 미리 프로비저닝 필요, Reserved Instance로 비용 절감

**Snapshots & DR:**

- **Multi-AZ 없음** ⭐
- 스냅샷은 S3에 저장, 증분 방식
- 자동(8시간마다/5GB마다/일정) + 수동(삭제 전까지 보관)
- **다른 리전으로 스냅샷 자동 복사** 가능 → 재해 복구

**Redshift Spectrum:** S3 데이터를 Redshift로 로드하지 않고 직접 쿼리. Redshift 클러스터 필요.

---

### 3. Amazon OpenSearch Service ⭐⭐

- ElasticSearch의 AWS 후속 서비스 (이름 변경됨)
- DynamoDB와 달리 **모든 필드, 부분 일치 검색** 가능
- **서버리스가 아님** — 인스턴스 클러스터 직접 생성 필요
- **SQL 미지원**, 자체 쿼리 언어 사용
- 주로 다른 DB를 **보완**하는 용도 (OpenSearch 단독 사용보다 DynamoDB + OpenSearch 조합)
- 데이터 주입: Kinesis Firehose, IoT, CloudWatch Logs 등
- OpenSearch Dashboard(시각화 도구) 포함

---

### 4. Amazon EMR ⭐

- **Elastic MapReduce** — Hadoop 클러스터를 AWS에서 관리
- Apache Spark, HBase, Presto, Flink 등과 함께 사용
- 수백 개의 EC2 인스턴스로 구성, Spot 인스턴스와 통합
- **사용 사례**: 빅데이터 처리, 머신러닝, 웹 인덱싱

**노드 유형 & 구매 옵션:**

|노드|역할|구매|
|---|---|---|
|**마스터 노드**|클러스터 관리, 상태 모니터링|Reserved (장기 실행)|
|**코어 노드**|태스크 실행 + 데이터 저장|Reserved (장기 실행)|
|**태스크 노드**|작업 실행만 (저장 없음)|**Spot** 인스턴스|
**Hadoop** : 하나의 큰 파일을 여러 컴퓨터에 나눠서 저장하고 천천히 처리할 떄
**Spark** : 메모리에서 처리해서 빠르게 끝낼때

-> 디스크냐 메모리에서 처리하느냐에 따라 다름

---

### 5. Amazon QuickSight ⭐

- **서버리스 ML 기반 BI 서비스**, 대화형 대시보드 생성
- SPICE 엔진으로 인메모리 계산
- 연결 가능: RDS, Aurora, Athena, Redshift, S3 등
- **Enterprise 에디션**: 컬럼 수준 보안(CLS) 가능
- QuickSight 사용자/그룹 ≠ IAM 사용자 (QuickSight 전용)
- 대시보드 공유 전 **먼저 게시(publish)** 해야 함

---

### 6. AWS Glue ⭐⭐

- **서버리스 ETL 서비스** (Extract, Transform, Load)
- 추출 -> 변환(가공) -> 적재 작업용도
- S3 → Parquet 변환, Glue Data Catalog로 메타데이터 관리
- 주로 **배치 작업**에서 사용

**추가 기능:**

- **Job Bookmark**: 이미 처리한 데이터 재처리 방지
- **Glue Elastic Views**: SQL로 여러 저장소 데이터 결합, 가상 테이블(materialized view)
- **Glue DataBrew**: 노코드 데이터 정리/정규화
- **Glue Studio**: ETL 작업 GUI
- **Glue Streaming ETL**: Kinesis, Kafka, MSK와 호환

---

### 7. AWS Lake Formation ⭐

- **데이터 레이크** = 분석용 중앙 집중식 데이터 저장소
- 수개월 걸리던 데이터 레이크 구축을 **며칠** 만에 가능
- 정형 + 비정형 데이터 결합 가능
- **행/열 수준 세분화된 액세스 제어** ← 핵심 기능
- AWS Glue 위에 빌드되지만 Glue와 직접 상호작용하지 않음
- 소스 블루프린트: S3, RDS, NoSQL DB 등

---

### 8. Kinesis Data Analytics ⭐⭐

**SQL 애플리케이션용:**

- Kinesis Data Streams + **Firehose** 모두 입력으로 사용 가능
- 완전 관리형, 오토 스케일링, 사용량 기반 과금
- 출력: Kinesis Data Streams 또는 Firehose
- 사용 사례: 시계열 분석, 실시간 대시보드, 실시간 지표

**Apache Flink용:**

- Java, Scala, SQL로 스트리밍 데이터 처리
- **⭐ Flink는 Firehose 데이터 읽기 불가** — Firehose 읽기가 필요하면 SQL 애플리케이션용 사용
- 체크포인트/스냅샷으로 백업

---

### 9. Amazon MSK ⭐

- **Managed Streaming for Apache Kafka** — Kinesis의 대안
- 완전 관리형 Kafka 클러스터. Broker + Zookeeper 노드 자동 관리
- VPC, 최대 3개 Multi-AZ 배포, 자동 장애 복구
- **EBS에 원하는 기간 동안** 데이터 저장 (Kinesis는 최대 365일 제한)
- **MSK Serverless**: 용량 관리 불필요, 자동 프로비저닝

**Kinesis vs MSK:**

![](../../images/스크린샷%202026-04-12%2020.49.22.png)

---

### 10. Big Data Ingestion Pipeline ⭐⭐ (아키텍처 시험 단골)

```
IoT Core → Kinesis Data Streams → Kinesis Firehose (최소 1분 간격)
         → Lambda (데이터 변환)
         → S3 (원시 데이터)
         → SQS → Lambda
         → Athena (서버리스 SQL 쿼리)
         → S3 (분석 결과)
         → QuickSight / Redshift (대시보드)
```

---

| **키워드**                             | **추천 서비스**          | **특징**                |
| ----------------------------------- | ------------------- | --------------------- |
| **S3 직접 쿼리, 서버리스, SQL**             | **Amazon Athena**   | 가장 빠르고 간편함, 쿼리당 비용    |
| **데이터 웨어하우스, 대규모 복잡한 분석**           | **Amazon Redshift** | 데이터 로드 필요, 클러스터 관리 필요 |
| **ETL(추출/변환/로드), 데이터 카탈로그**         | **AWS Glue**        | 데이터를 분류하고 정리할 때 사용    |
| **Big Data, Hadoop, Spark, 복잡한 연산** | **Amazon EMR**      | 성능은 강력하지만 운영이 매우 힘듦   |

### 시험 TIP 최종 정리

- **S3 서버리스 SQL 분석** → **Athena** ($5/TB)
- **대용량 데이터 웨어하우스 OLAP** → **Redshift** (Multi-AZ 없음!)
- **텍스트/부분 검색** → **OpenSearch** (서버리스 아님, SQL 미지원)
- **Hadoop 빅데이터 클러스터** → **EMR** (태스크 노드 = Spot)
- **BI 대시보드 서버리스** → **QuickSight** (SPICE, 컬럼 보안은 Enterprise)
- **서버리스 ETL** → **Glue** (Job Bookmark = 재처리 방지)
- **행/열 수준 데이터 접근 제어** → **Lake Formation**
- **Flink는 Firehose 못 읽음** → SQL 애플리케이션용 Kinesis Analytics 사용
- **기존 Kafka 환경 마이그레이션** → **MSK** (데이터 보존 기간 무제한, 메시지 최대 10MB)