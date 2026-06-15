# 커넥션 풀과 HikariCP

## 왜 커넥션 풀이 필요한가
DB 커넥션은 만드는 비용이 매우 큼:
1. **TCP 3-way handshake**
2. **DB 인증** (자격 증명 검증)
3. **세션 변수 설정** (autocommit, isolation, charset 등)
→ 매 요청마다 새로 만들면 **수십~수백 ms** 소비

해결: **미리 만들어둔 커넥션을 풀에 보관 → 재사용**

## 커넥션 풀의 동작
```
요청 → 풀에서 빈 커넥션 빌림(borrow)
     → 쿼리 실행
     → 풀에 반납(return)
```
빌릴 수 있는 게 없으면 **대기**, 타임아웃 초과하면 예외.

## HikariCP (Spring Boot 기본)
- 가볍고 빠른 자바 커넥션 풀. **Spring Boot 2.x+ 기본**
- 다른 풀: Apache DBCP, C3P0, Tomcat JDBC

### 핵심 설정 (`application.yml`)
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10          # 풀 최대 크기 ← 가장 중요
      minimum-idle: 10                # 최소 유지 (보통 maximum과 동일 권장)
      connection-timeout: 30000       # 빌리기 대기 한계 (ms)
      idle-timeout: 600000            # 유휴 커넥션 폐기까지
      max-lifetime: 1800000           # 커넥션 최대 수명 (DB의 wait_timeout보다 짧게)
      validation-timeout: 5000
      leak-detection-threshold: 60000 # 누수 의심 임계값
```

## maximum-pool-size 어떻게 정할까
- **무작정 크게 = 더 빠르지 않음**. 오히려 DB 커넥션 수만 늘어 메모리·CPU 부담
- HikariCP 공식 공식 (출처: 그들의 위키):
  ```
  connections = ((core_count * 2) + effective_spindle_count)
  ```
- **현실적 출발점**: 10~20. 부하 테스트로 조정
- **DB 쪽 max_connections 도 같이 확인** (Aurora·RDS는 인스턴스 등급에 따라 다름)

## 자주 발생하는 문제

### 1. Connection Leak
커넥션 반납 누락 → 풀 고갈 → `connection-timeout` 후 예외
- 원인: try-with-resources 미사용, 예외 발생 후 close 안 함, `@Transactional` 미사용 + 수동 관리
- 탐지: `leak-detection-threshold` 설정, 로그 확인

### 2. Pool Exhaustion
요청은 많은데 풀이 작음 → 모두 대기 → 타임아웃 → 503
- 해결: 풀 크기 ↑, 또는 **쿼리 시간 줄이기** (대부분 이쪽이 진짜 원인)

### 3. Stale Connection
DB가 idle 커넥션을 끊었는데 풀은 모르고 빌려줌 → 다음 쿼리 실패
- 해결: `max-lifetime` 을 DB의 `wait_timeout` 보다 짧게

### 4. 트랜잭션 보유 시간 길어짐
`@Transactional` 메서드 안에서 **외부 API 호출 / 무거운 계산** → 커넥션 점유 길어짐 → 풀 고갈
- 해결: 트랜잭션 안에서는 DB 작업만, 외부 호출은 트랜잭션 밖으로

## 모니터링 포인트
- **Active / Idle / Pending 커넥션 수**
- 평균 대기 시간, 최대 대기 시간
- Prometheus: `hikaricp_*` 메트릭
- 그라파나 대시보드로 시각화

## 톰캣 스레드풀과의 관계
```
HTTP 요청 → Tomcat 스레드 (server.tomcat.threads.max=200, 기본)
         → HikariCP 커넥션 (default 10)
```
- Tomcat 스레드 ≫ HikariCP 커넥션이면 → 많은 요청이 DB 대기로 막힘
- 그렇다고 풀을 200으로 늘리면 DB가 죽음
- **핵심**: 쿼리를 빠르게 만드는 게 본질. 풀 크기는 보조 수단

## 면접 빈출 질문
1. **커넥션 풀이 왜 필요한가?** → DB 커넥션 생성 비용(handshake, 인증, 세션)이 비싸 재사용
2. **HikariCP의 maximum-pool-size를 무조건 크게 하면 좋은가?** → 아니오. DB 부담 ↑, 컨텍스트 스위칭 ↑. 적정값(10~20 출발)을 부하 테스트로 결정
3. **Connection Leak을 어떻게 탐지·예방?** → leak-detection-threshold, try-with-resources, `@Transactional` 활용
4. **`@Transactional` 안에서 외부 API를 호출하면?** → 커넥션을 외부 응답 시간 동안 잡고 있어 풀 고갈 위험 → 트랜잭션 외부에서 호출
5. **max-lifetime을 DB의 wait_timeout보다 짧게 설정하는 이유?** → DB가 먼저 끊으면 풀은 유효한 줄 알고 사용 → 다음 쿼리 실패
6. **Tomcat 스레드 풀과 HikariCP 풀의 관계?** → 톰캣 스레드가 HikariCP 커넥션을 빌려 쓰는 구조. 두 값의 균형 + 쿼리 최적화가 핵심

## 프로젝트 연결
- recaring 부하 테스트: K6로 동시 요청 ↑ → HikariCP active/pending 그래프 확인 → pool size·쿼리 튜닝
- 티켓팅 결제 처리: 외부 PG API 호출은 트랜잭션 밖에서 (Outbox 패턴)
