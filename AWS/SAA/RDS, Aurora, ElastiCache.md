### 1. RDS 기본

관리형 관계형 DB 서비스. 지원 엔진: PostgreSQL, MySQL, MariaDB, Oracle, MS SQL Server, Aurora.

**EC2에 직접 DB 설치 vs RDS 차이**: RDS는 OS 패치, 백업, 모니터링, 스케일링, 다중 AZ를 AWS가 대신 관리해줌. 단, **SSH 접속 불가** (RDS Custom 제외).

**Storage Auto Scaling**: 스토리지 자동 확장. 남은 공간 10% 미만 + 5분 이상 지속 + 마지막 수정 후 6시간 경과 시 자동 확장. 최대 임계값 설정 필수.

---

### 2. RDS Read Replicas vs Multi-AZ ⭐⭐⭐ (초빈출)

이 둘의 차이는 시험에서 거의 매번 나옵니다.

![](../../images/스크린샷%202026-04-12%2020.17.12.png)

> **시험 TIP**: "재해 복구", "failover" → Multi-AZ / "읽기 부하 분산", "분석 쿼리" → Read Replica

**Single-AZ → Multi-AZ 전환**: 다운타임 없이 가능. 수정 버튼 클릭만 하면 내부적으로 스냅샷 생성 → 새 AZ에 복원 → 동기화 자동 설정.

---

### 3. RDS Custom

Oracle, MS SQL Server 전용. 기저 EC2와 OS에 직접 접근 가능 (SSH/SSM). 사용 전 자동화 끄는 것 권장, 스냅샷 미리 생성 권장.

---

### 4. Aurora ⭐⭐

AWS 독자 기술. MySQL/PostgreSQL 호환.

**핵심 스펙 — 숫자 외워야 함:**

- MySQL 대비 **5배**, PostgreSQL 대비 **3배** 성능
- 스토리지 자동 확장, 최대 **128TB**, 10GB 단위 증가
- **3개 AZ에 6개 복사본** 저장
    - 쓰기: 6개 중 **4개** 필요
    - 읽기: 6개 중 **3개** 필요
- 복제본 최대 **15개**, 복제 지연 **10ms 미만**
- Failover **30초 이내**
- RDS 대비 비용 **20% 높음**

**Aurora DB Cluster 엔드포인트:**

- **Writer Endpoint**: 항상 마스터를 가리킴
- **Reader Endpoint**: 읽기 복제본들에 로드 밸런싱 (connection level)
- **Custom Endpoint**: 특정 복제본 서브셋으로 라우팅 (예: 분석용 고사양 복제본)

**Aurora 특수 기능들:**

|기능|설명|
|---|---|
|**Aurora Serverless**|간헐적/예측 불가 워크로드. 자동 인스턴스화, 초당 과금|
|**Aurora Multi-Master**|모든 노드에서 쓰기 가능. 쓰기 고가용성 필요 시|
|**Global Aurora**|최대 5개 읽기 전용 리전, 복제 지연 **1초 미만**, RTO **1분 미만**|
|**Backtrack**|스냅샷 없이 특정 시점으로 데이터 복원|
|**DB Cloning**|스냅샷보다 빠름. 프로덕션 영향 없이 스테이징 DB 생성|

---

### 5. 백업 비교 ⭐

![](../../images/스크린샷%202026-04-12%2020.17.34.png)


> **팁**: 중지된 RDS도 스토리지 비용 계속 발생. 장기 미사용 시 스냅샷 → 삭제 → 나중에 복원이 경제적.

**S3에서 복원 시:** MySQL RDS는 일반 백업 파일 사용, Aurora는 **Percona XtraBackup** 소프트웨어 사용.

---

### 6. RDS & Aurora 보안

- **저장 암호화**: KMS 사용. 처음 생성 시 정의. 마스터가 미암호화면 Read Replica도 암호화 불가
- **전송 암호화**: 기본 TLS
- **IAM 인증**: 사용자명/패스워드 대신 IAM Role로 연결 가능
- **SSH 불가** (RDS Custom 제외)
- 감사 로그 → CloudWatch Logs로 장기 보관 가능

---

### 7. RDS Proxy ⭐⭐

Lambda + RDS 조합 문제에서 자주 등장합니다.

- **연결 풀링**: 앱이 프록시에 연결 → 프록시가 DB 연결 관리. CPU/RAM 부하 감소
- Failover 시간 **66% 단축**
- 완전 서버리스, Multi-AZ 지원
- IAM 인증 강제 가능, 자격증명은 **Secrets Manager** 저장
- **퍼블릭 액세스 절대 불가 → VPC 내부에서만 접근**
- 지원: MySQL, PostgreSQL, MariaDB, MS SQL Server, Aurora(MySQL/PostgreSQL)

---

### 8. ElastiCache ⭐⭐

인메모리 캐시 서비스. Redis 또는 Memcached 관리형.

**주요 사용 패턴:**

- **DB Cache**: 앱 → ElastiCache 조회 (캐시 히트) → 없으면 RDS 조회 후 캐시 저장
- **Session Store**: 로그인 세션을 ElastiCache에 저장 → 다른 인스턴스로 이동해도 재로그인 불필요

---

### 9. Redis vs Memcached ⭐⭐⭐

![](../../images/스크린샷%202026-04-12%2020.17.50.png)
> **한 줄 요약**: 고가용성/내구성/복잡한 구조 → Redis / 단순 캐시/멀티스레드/샤딩 → Memcached

**보안:**

- ElastiCache는 **IAM 인증 미지원** (IAM 정책은 AWS API 레벨만)
- Redis: **Redis AUTH** (비밀번호/토큰) + SSL
- Memcached: **SASL** 기반 인증

---

### 10. 캐싱 패턴 ⭐

|패턴|동작|장점|단점|
|---|---|---|---|
|**Lazy Loading**|읽을 때만 캐시에 저장|필요한 것만 캐싱|캐시 미스 시 3번 네트워크 호출, stale 데이터 가능|
|**Write Through**|DB 쓸 때 캐시도 동시에 업데이트|stale 데이터 없음, 읽기 빠름|처음엔 데이터 없음, churn 발생 가능|
|**Session Store**|TTL 설정한 세션 임시 저장|stateless 앱 구현|-|

**TTL (Time-to-live)**: 캐시 자동 만료 시간. 리더보드, 댓글, 활동 스트림 등에 유용. 메모리 부족으로 제거가 많이 일어나면 스케일 업/아웃 필요.

---

### 11. 포트 번호 — 시험에 그냥 나옴

|서비스|포트|
|---|---|
|PostgreSQL / Aurora(PG)|**5432**|
|MySQL / MariaDB / Aurora(MySQL)|**3306**|
|Oracle|**1521**|
|MS SQL Server|**1433**|
|SSH / SFTP|22|
|HTTP / HTTPS|80 / 443|

---

### 시험 TIP 최종 정리

- "읽기 부하 분산" → Read Replica / "재해 복구, failover" → Multi-AZ
- "Aurora 숫자": 3AZ, 6복사본, 15복제본, 128TB, 30초 failover, 글로벌 복제 1초 미만
- "Lambda가 RDS 연결 폭주" → **RDS Proxy**
- "고가용성 캐시, 백업, 정렬" → **Redis** / "단순 분산 캐시, 멀티스레드" → **Memcached**
- "예측 불가 워크로드 DB" → **Aurora Serverless**
- "프로덕션 DB 복사해서 테스트" → **Aurora Cloning**