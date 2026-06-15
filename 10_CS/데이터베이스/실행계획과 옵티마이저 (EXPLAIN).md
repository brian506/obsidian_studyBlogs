# 실행계획과 옵티마이저 (EXPLAIN)

## 옵티마이저란
SQL을 어떻게 실행할지 (어떤 인덱스, 어떤 조인 알고리즘, 어떤 순서) **실행 계획(Plan)** 을 만드는 DB의 두뇌.

- **규칙 기반(RBO)**: 정해진 규칙에 따라 — 옛날 방식
- **비용 기반(CBO)**: **통계 정보**로 비용을 예측 → 가장 싼 계획 선택 (현대 DBMS 표준)

→ 통계 정보가 부정확하면 좋은 계획이 안 나옴. **ANALYZE TABLE** 로 통계 갱신.

## EXPLAIN 사용
```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 100;

-- 실제 실행 + 계획
EXPLAIN ANALYZE SELECT ...;  -- PostgreSQL / MySQL 8.0.18+
```

## MySQL EXPLAIN 주요 컬럼

| 컬럼 | 의미 |
| --- | --- |
| **id** | 쿼리 실행 순서 (서브쿼리·UNION 구분) |
| **select_type** | SIMPLE / PRIMARY / SUBQUERY / DERIVED 등 |
| **table** | 접근 대상 테이블 |
| **type** | **접근 방식** ← 가장 중요. 아래 표 참고 |
| **possible_keys** | 사용 가능한 인덱스 후보 |
| **key** | 실제 사용된 인덱스 |
| **key_len** | 사용된 인덱스 바이트 |
| **rows** | 읽을 행 추정치 |
| **filtered** | rows 중 WHERE로 통과할 비율 (%) |
| **Extra** | Using index / Using filesort / Using temporary 등 |

## type 컬럼 — 좋은 순서
```
system > const > eq_ref > ref > range > index > ALL
```
- **const**: 기본키/유니크 인덱스로 한 행 (가장 빠름)
- **eq_ref**: JOIN에서 한 행씩 PK/유니크로 매칭
- **ref**: 인덱스로 여러 행 매칭 (일반적인 인덱스 활용)
- **range**: 인덱스로 범위 스캔 (BETWEEN, >, <)
- **index**: **인덱스 풀스캔** — 인덱스를 다 읽음 (느릴 수 있음)
- **ALL**: **테이블 풀스캔** — 최악. 인덱스 미사용
→ **ALL이 보이면 가장 먼저 의심해야 할 신호**

## Extra 컬럼 — 자주 보는 키워드
- `Using index` — **커버링 인덱스** (인덱스만으로 결과 만족, 테이블 접근 X). 매우 좋음
- `Using where` — WHERE로 필터링
- `Using filesort` — **정렬을 위해 임시 정렬 수행** → ORDER BY가 인덱스를 못 탔다. 인덱스 추가 검토
- `Using temporary` — **임시 테이블 생성** → GROUP BY·DISTINCT·UNION에서 자주. 비싸므로 회피
- `Using join buffer` — 인덱스 없는 조인 → 인덱스 추가
- `Impossible WHERE` — WHERE가 항상 false

## 자주 발생하는 안티 패턴
1. **인덱스 컬럼에 함수 적용**: `WHERE DATE(created_at) = '2024-01-01'` → 인덱스 못 탐
   → `WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02'`
2. **묵시적 형변환**: 컬럼 타입과 다른 값 비교 (`WHERE phone = 01012345678` 인데 컬럼은 VARCHAR)
3. **LIKE의 와일드카드 시작**: `LIKE '%abc'` → 인덱스 못 탐. `LIKE 'abc%'` 는 가능
4. **OR**: 인덱스 한쪽만 가능하면 풀스캔 → UNION ALL 로 분리 검토
5. **복합 인덱스 순서 오해**: `idx(a, b, c)` 에서 b만 검색하면 인덱스 못 탐. **선두 컬럼 필수**
6. **SELECT \*** : 커버링 인덱스 무력화

## 옵티마이저 힌트 (필요 시 강제)
```sql
SELECT /*+ INDEX(orders idx_user_id) */ * FROM orders WHERE user_id = 100;
```
→ 옵티마이저가 잘못된 선택을 할 때만 마지막 수단. **통계 갱신·인덱스 정비가 우선**.

## 면접 빈출 질문
1. **EXPLAIN을 왜 보나?** → 옵티마이저가 어떤 인덱스·접근 방식·조인 알고리즘을 골랐는지 확인 → 튜닝의 시작
2. **type=ALL이 나오면?** → 풀스캔. 인덱스가 없거나 옵티마이저가 인덱스를 무시함. 인덱스 추가·SQL 수정 검토
3. **커버링 인덱스란?** → SELECT·WHERE의 모든 컬럼이 인덱스 안에 있어서 테이블 접근 없이 끝나는 케이스 → `Using index`
4. **인덱스가 있어도 못 타는 경우?** → 함수 적용, 묵시적 형변환, LIKE '%xxx', OR, 선두 컬럼 미사용, 카디널리티 낮음
5. **Using filesort는 왜 나쁜가?** → ORDER BY가 인덱스 정렬을 못 써서 추가 정렬 수행 → 인덱스 순서 맞추기
6. **옵티마이저 통계가 부정확하면?** → 비용 추정 오류 → 잘못된 계획. ANALYZE TABLE로 통계 갱신

## 프로젝트 연결
- 냉장고 프로젝트: 유통기한 임박 식재료 조회 쿼리 개선 → EXPLAIN으로 type/Extra 확인 후 복합 인덱스
- 위스키: 피드 인덱스 → 카디널리티·정렬 컬럼 고려해 복합 인덱스 설계
- 페이징 cursor 방식이 offset보다 좋은 이유도 결국 인덱스 range scan 활용
