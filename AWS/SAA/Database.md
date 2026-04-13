
### 1. DB 유형 분류 ⭐⭐⭐

|유형|서비스|키워드|
|---|---|---|
|**RDBMS (SQL/OLTP)**|RDS, Aurora|Join, 트랜잭션|
|**NoSQL**|DynamoDB (JSON), ElastiCache (키/값), Neptune (그래프), DocumentDB (MongoDB), Keyspaces (Cassandra)|조인 없음, 유연한 스키마|
|**객체 스토어**|S3, Glacier|대용량 파일, 아카이브|
|**데이터 웨어하우스**|Redshift (OLAP), Athena, EMR|SQL 분석, BI|
|**검색**|OpenSearch|비정형 텍스트, JSON 검색|
|**그래프**|Neptune|관계 데이터, 소셜 네트워크|
|**원장**|QLDB|불변 금융 거래 기록|
|**시계열**|Timestream|IoT, 실시간 분석|

---

### 2. RDS ⭐

- 관리형 RDBMS (PostgreSQL, MySQL, Oracle, SQL Server, MariaDB)
- EBS 볼륨 + 인스턴스 크기 직접 프로비저닝 필요
- 읽기 전용 복제본 + Multi-AZ 지원
- 자동 백업 최대 35일, 수동 스냅샷 장기 보관
- RDS Proxy로 IAM 인증 강제, Secrets Manager 통합
- **RDS Custom**: Oracle, SQL Server만 해당, OS/인스턴스 직접 접근 가능
- **사용 사례**: OLTP, SQL 쿼리, 트랜잭션

---

### 3. Aurora ⭐⭐

- PostgreSQL/MySQL 호환, 컴퓨팅/스토리지 분리
- 3 AZ × 6개 복제본, 자가 복구, 오토스케일링
- **Aurora Serverless**: 예측 불가 워크로드, 용량 계획 불필요
- **Aurora Multi-Master**: 쓰기 고가용성 (여러 인스턴스가 동시 쓰기)
- **Aurora Global**: 리전 간 복제 1초 미만, 리전당 최대 16개 읽기 인스턴스
- **Aurora Cloning**: 스냅샷보다 빠른 클러스터 복제
- **사용 사례**: RDS와 동일하되 더 높은 성능/가용성 필요 시

---

### 4. ElastiCache ⭐⭐

- 관리형 Redis/Memcached, 인메모리, 1ms 미만 지연
- EC2 인스턴스 유형 직접 프로비저닝 필요
- **⭐ 핵심 주의사항**: **앱 코드 수정 필요**. "코드 변경 없는 캐싱" 문제에서는 정답이 아님!
- **사용 사례**: 세션 데이터, DB 쿼리 캐싱, 키-값 스토어, 고읽기/저쓰기 워크로드

---

### 5. DynamoDB ⭐⭐

- 서버리스 NoSQL, 밀리초 지연, SQL 불가
- Provisioned(예측 가능 워크로드) / On-demand(예측 불가) 모드
- DAX로 마이크로초 읽기 캐시
- DynamoDB Streams → Lambda/Kinesis 이벤트 처리
- Global Tables: active-active 멀티 리전 읽기/쓰기
- PITR 최대 35일, S3 내보내기/가져오기 지원
- **사용 사례**: 400KB 미만 서버리스 앱, 빠른 스키마 변경, 서버리스 세션 저장

---

### 6. 특수 목적 DB들 ⭐⭐ (각 키워드 연결이 중요)

**DocumentDB** — "MongoDB를 AWS에서"

- MongoDB 호환 NoSQL, JSON 데이터
- Aurora와 유사한 배포 (3 AZ 복제), 10GB씩 자동 확장 최대 64TB
- 초당 수백만 요청 처리 가능

**Neptune** — "그래프 데이터베이스"

- 소셜 네트워크, 지식 그래프, 사기 탐지, 추천 엔진
- 3 AZ, 최대 15개 읽기 복제본
- 수십억 관계를 밀리초 지연으로 쿼리

**Keyspaces** — "Apache Cassandra를 AWS에서"

- CQL(Cassandra Query Language) 사용
- 서버리스, 3 AZ 복제, 10ms 미만 지연
- **사용 사례**: IoT 디바이스 정보, 시계열 데이터

**QLDB** — "불변 금융 원장"

- 모든 변경 이력 추적, 수정/삭제 불가 (Immutable)
- 서버리스, 3 AZ, SQL 사용 가능
- Managed Blockchain과 차이: **탈중앙화 없음**, 중앙 관리 금융 규정 준수용

**Timestream** — "시계열 데이터베이스"

- 완전 관리형 서버리스, 하루 수조 건 이벤트
- 관계형 DB 대비 **1000배 빠르고 비용 1/10**
- 최근 데이터는 메모리, 과거 데이터는 저비용 스토리지
- **사용 사례**: IoT, 운영 모니터링, 실시간 분석

---

### 시험 TIP — 문제 유형별 정답 패턴 ⭐⭐⭐

|상황|정답|
|---|---|
|SQL, 트랜잭션, 조인 필요|RDS 또는 Aurora|
|고가용성 + 높은 성능 관계형 DB|Aurora|
|예측 불가 서버리스 워크로드|Aurora Serverless|
|인메모리 캐시 (코드 수정 OK)|ElastiCache|
|코드 수정 없는 캐싱|ElastiCache ❌ → 다른 방법 고려|
|NoSQL, 서버리스, 빠른 스키마|DynamoDB|
|MongoDB 워크로드 마이그레이션|DocumentDB|
|소셜 네트워크, 관계 그래프|Neptune|
|Apache Cassandra 마이그레이션|Keyspaces|
|금융 거래 불변 기록|QLDB|
|IoT 시계열 데이터 분석|Timestream|
|정적 파일, 대용량 객체|S3|
|SQL 기반 데이터 분석/BI|Redshift, Athena|
|텍스트 검색, 비정형 데이터|OpenSearch|