# 데이터베이스 — 면접 대비 인덱스

## 기존 노트
### 인덱스
- [[인덱스 구조 (B-Tree 자료구조)]]
- [[인덱스 기본 사용법]]
- [[인덱스 사용]]
- [[인덱스 스캔 과정과 인덱스 스캔 효율]]
- [[가상 컬럼 (Virtual Column)]]

### 트랜잭션·동시성
- [[트랜잭션 생명주기]]
- [[트랜잭션 전파 (Propagation)]]
- [[MVCC 와 트랜잭션 격리수준 (Transaction Isolation Level)]]
- [[스토리지 엔진 수준의 락 (레코드 락, 갭 락, 넥스트 키 락)]]
- [[DB 차원에서의 동시성 제어]]
- [[@Transaction 과 스프링 AOP(Aspect-Oriented Programming)]]
- [[Synchronized-비관적 락-낙관적 락-레디스 분산락]]
- [[Redis 에서 동시성을 제어하는 방법]]

### 기타
- [[데이터베이스 저장 구조 및 조회 방식]]
- [[벌크 연산 시 주의할 점(execute, clear)]]
- [[postgre 개념들]]
- NoSQL/[[MongoDB]], [[MongoDB 인덱스 활용]]
- 페이징/[[페이징 기법 - cursor 기반]], [[페이징 기법 - offset 기반]]
- 동적 쿼리/[[queryDSL - 동적쿼리]]

## 새로 추가 (면접 빈출 + 프로젝트 활용)
- [[정규화와 반정규화]] — 1NF~BCNF, 반정규화 트레이드오프
- [[ACID 원리]] — Undo/Redo Log, BASE 비교
- [[조인 알고리즘과 조인 종류]] — NLJ/Hash/Sort-Merge, N+1
- [[실행계획과 옵티마이저 (EXPLAIN)]] — type/Extra 컬럼, 안티패턴
- [[샤딩·파티셔닝·리플리케이션]] — 헷갈리지 않게 구분
- [[커넥션 풀과 HikariCP]] — 풀 사이즈, 누수, Tomcat 스레드 관계

## 빈출 주제 우선순위
1. **인덱스 동작 원리** (B+Tree, 복합 인덱스 선두 컬럼)
2. **트랜잭션 격리수준 + MVCC**
3. **N+1 문제와 해결** (fetch join, EntityGraph)
4. **EXPLAIN 읽기** (type=ALL 안 좋은 이유, 커버링 인덱스)
5. **ACID 4가지 + 구현 메커니즘**
6. **비관적/낙관적 락 + 분산 락**
7. **샤딩 vs 파티셔닝 vs 리플리케이션 구분**
8. **HikariCP 풀 사이즈 결정 기준**

## 자주 묻는 시나리오
- "이 쿼리가 느려요. 어떻게 튜닝?" → EXPLAIN → type/Extra → 인덱스 추가 → 커버링 인덱스 → 쿼리 수정
- "동시에 같은 자원 업데이트, 어떻게?" → 비관적 락 / 낙관적 락 / 분산 락 트레이드오프
- "Replica로 읽었더니 방금 쓴 게 없어요" → 복제 지연, Read-Your-Writes, 캐시 활용
