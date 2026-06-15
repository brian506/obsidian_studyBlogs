# 인증과 인가 (Authentication vs Authorization)

## 한 줄 정의
- **인증(Authentication, AuthN)**: 너 **누구**야? — 신원 확인
- **인가(Authorization, AuthZ)**: 너 이거 **해도 돼**? — 권한 확인

→ 인증이 먼저, 그 다음 인가. 호텔에 비유하면 **체크인(인증) → 객실 키로 특정 층 출입(인가)**

## 인증 방식

### 1. 세션 기반 (Session-based)
1. 로그인 성공 시 서버가 세션 생성 후 **Session ID**를 쿠키로 전달
2. 이후 요청마다 쿠키의 Session ID로 서버 메모리/DB에서 사용자 조회
- 장점: 서버에서 즉시 무효화 가능 (로그아웃, 강제 종료)
- 단점: **서버에 상태 저장** → 수평 확장 시 세션 공유 필요 (Sticky Session, Redis 세션 스토어)

### 2. 토큰 기반 (Token-based, JWT)
1. 로그인 성공 시 서버가 JWT 발급
2. 클라이언트는 매 요청 `Authorization: Bearer <token>` 헤더로 전송
3. 서버는 **서명만 검증** (DB 조회 X)
- 장점: **Stateless** → 수평 확장 용이, MSA 적합
- 단점: 발급된 토큰을 만료 전 무효화 어려움 (블랙리스트·짧은 만료 + Refresh Token)

### 3. OAuth 2.0 / OIDC
- 외부 제공자(Google, Kakao)에게 인증을 위임받음. → 별도 노트

## 인가 방식

### 1. RBAC (Role-Based Access Control)
- 사용자 → **역할(Role)** → 권한(Permission)
- 예: USER, ADMIN. 대부분의 Spring Security 예제가 이 방식

### 2. ABAC (Attribute-Based Access Control)
- 사용자/리소스/환경 **속성**으로 정책 평가 (예: "본인이 작성한 게시물만 수정")
- 더 유연하지만 정책 관리 복잡

### 3. ACL (Access Control List)
- 리소스별 허용 사용자 목록 (예: 파일별 권한)

## Spring Security에서의 매핑
- 인증: `AuthenticationManager`, `UsernamePasswordAuthenticationFilter`
- 인가: `SecurityFilterChain`의 `.authorizeHttpRequests()`, `@PreAuthorize`
- `SecurityContextHolder` ← 인증된 사용자 정보 저장 (스레드 로컬)

## 쿠키 보안 옵션
- **Secure**: HTTPS에서만 전송
- **HttpOnly**: JS에서 `document.cookie` 접근 차단 → XSS 방지
- **SameSite**:
  - `Strict`: 동일 사이트만 (가장 안전, UX 제약)
  - `Lax`: 톱레벨 navigation은 허용 (기본값) — **CSRF 방지**
  - `None`: 모든 크로스사이트 허용 (`Secure` 필수)

## 면접 빈출 질문
1. **인증과 인가의 차이?** → 신원 확인 vs 권한 확인. 인증이 먼저.
2. **세션과 JWT 중 무엇이 좋은가?** → 트레이드오프. 즉시 무효화·보안성=세션, Stateless·확장성=JWT
3. **JWT를 쓰면서 로그아웃은 어떻게?** → 짧은 Access Token + Redis 블랙리스트 + Refresh Token rotation
4. **Refresh Token을 어디에 저장?** → HttpOnly·Secure·SameSite 쿠키 (XSS·CSRF 동시 방어) / 서버 측은 Redis에 화이트리스트
5. **Spring Security의 인증 흐름?** → Filter → AuthenticationManager → AuthenticationProvider → UserDetailsService → SecurityContextHolder
6. **HttpOnly가 막는 공격은?** → XSS로 인한 토큰 탈취
7. **SameSite가 막는 공격은?** → CSRF

## 프로젝트 연결
- 위스키/티켓팅의 로그인: JWT + Refresh Token Redis 저장 → 무효화 가능성 확보
- OAuth 2.0 로그인 흐름은 별도 노트 참고
