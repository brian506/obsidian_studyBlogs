# ACID 원리

트랜잭션이 안전하게 처리되기 위한 4가지 특성. **데이터베이스의 신뢰성**의 근간.

## A — Atomicity (원자성)
> "All or Nothing"

트랜잭션 안의 모든 작업이 **전부 성공하거나 전부 실패** 해야 함. 중간에 실패하면 모든 변경을 **롤백**.

- 구현: **Undo Log** — 변경 전 값을 기록해두고 실패 시 되돌림
- 예: 계좌 이체 — 출금만 되고 입금이 실패하면 안 됨

## C — Consistency (일관성)
> 트랜잭션 전후로 **DB 제약 조건과 무결성**이 유지되어야 함

- PK·FK·CHECK·NOT NULL 같은 제약 조건이 위반되면 트랜잭션 실패
- 비즈니스 일관성(잔고 ≥ 0)은 애플리케이션이 함께 책임짐

## I — Isolation (격리성)
> 동시에 실행되는 트랜잭션이 **서로 영향을 주지 않아야 함**

격리 수준이 낮으면 빠르지만 동시성 부작용 발생:
- **Dirty Read**: 커밋 안 된 데이터 읽음
- **Non-Repeatable Read**: 같은 행을 두 번 읽었는데 값이 다름
- **Phantom Read**: 같은 조건으로 다시 읽었더니 행 수가 달라짐

→ 자세한 격리 수준은 [[10_CS/데이터베이스/트랜잭션/MVCC 와 트랜잭션 격리수준 (Transaction Isolation Level)]]

## D — Durability (지속성)
> 트랜잭션이 커밋되면 **시스템 장애가 나도 결과가 보존**되어야 함

- 구현: **WAL (Write-Ahead Log)** — 데이터 변경 전에 **로그를 먼저 디스크에 기록**
- 장애 후 재시작 시 WAL을 재생(replay)해 복구
- MySQL의 **redo log**, PostgreSQL의 **WAL**, Oracle의 **redo log** 모두 같은 개념

## ACID의 구현 — 로그가 핵심
| 보장 | 핵심 메커니즘 |
| --- | --- |
| Atomicity | **Undo Log** (변경 전 값) |
| Durability | **Redo Log / WAL** (변경 후 값) |
| Isolation | **락 + MVCC** |
| Consistency | 제약 조건 + 위 3개의 결과 |

## ACID vs BASE
분산 시스템에서는 ACID를 모두 지키기 어려워 트레이드오프 → **BASE**

| | ACID | BASE |
| --- | --- | --- |
| 일관성 | 즉시 (Strong) | 결국 (Eventual) |
| 가용성 | 트랜잭션 중 잠금 | 항상 응답 |
| 사용 | RDB | NoSQL, MSA |

→ [[10_CS/분산시스템]] 의 CAP 이론과 연결

## Spring `@Transactional` 과의 관계
- `@Transactional` 메서드 진입 → BEGIN
- 정상 종료 → COMMIT, 예외(unchecked) → **ROLLBACK**
- 내부 호출 / proxy / 전파 옵션 주의 → [[10_CS/데이터베이스/트랜잭션/@Transaction 과 스프링 AOP(Aspect-Oriented Programming)]]

## 면접 빈출 질문
1. **ACID 4가지 설명?** → 원자성·일관성·격리성·지속성. 각각 한 줄로 정의
2. **원자성과 지속성은 어떻게 구현?** → Undo Log(롤백)와 Redo Log/WAL(장애 복구)
3. **격리 수준이 낮으면 무슨 문제?** → Dirty Read / Non-Repeatable Read / Phantom Read
4. **분산 시스템에서 ACID가 어려운 이유?** → 노드 간 통신·장애로 강한 일관성 보장 시 가용성·성능 희생 → BASE로 완화
5. **MySQL의 ACID 보장 엔진은?** → InnoDB. MyISAM은 트랜잭션 미지원

## 프로젝트 연결
- 티켓팅 결제: **분산 트랜잭션** 대신 **Outbox 패턴** + 보상 트랜잭션 → ACID의 "강한 일관성"을 포기하고 BASE로 풀어낸 사례
- 위스키 채팅: MongoDB는 단일 도큐먼트 단위 ACID 보장, 멀티 도큐먼트는 트랜잭션 옵션 별도
