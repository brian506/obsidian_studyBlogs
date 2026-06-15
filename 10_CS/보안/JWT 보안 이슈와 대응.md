# JWT 보안 이슈와 대응

## JWT 구조 (복습)
`xxxxx.yyyyy.zzzzz` — 점으로 구분된 3부분, Base64URL 인코딩

1. **Header**: 알고리즘(alg), 토큰 타입(typ)
2. **Payload**: Claims (sub, exp, iat, 커스텀 데이터)
3. **Signature**: `HMAC(header.payload, secret)` 또는 RSA/ECDSA 서명

핵심: **서명만 검증하면 위변조 여부를 알 수 있음** → 서버는 별도 저장 없이 신뢰 가능 (Stateless)

> ⚠️ **Payload는 암호화가 아니라 인코딩**. 누구나 디코딩해 내용을 볼 수 있음 → 민감 정보 금지

## 주요 보안 이슈

### 1. 토큰 탈취 (Token Theft)
한 번 발급된 토큰은 만료 전엔 그 자체로 인증 수단 → 탈취되면 그대로 악용.

**원인**
- localStorage 저장 → XSS로 탈취
- HTTPS 미적용 → 중간자 탈취
- 로그·URL에 노출

**대응**
- **HttpOnly + Secure + SameSite 쿠키**에 Refresh Token 보관 → XSS·CSRF 동시 방어
- Access Token은 짧게 (5~30분)
- HTTPS 강제 (HSTS)
- 로그에 토큰 절대 안 찍기

### 2. Stateless의 함정 — 무효화 어려움
서버에 상태가 없으니 발급한 토큰을 **즉시 폐기**하기 어려움. 로그아웃해도 토큰은 만료 전까지 유효.

**대응 패턴**
- **짧은 Access + Refresh Token** — 탈취돼도 노출 시간 짧음
- **Refresh Token을 Redis에 화이트리스트로 저장** — 로그아웃 시 삭제 → 즉시 무효화
- **블랙리스트** — 폐기된 Access Token 목록을 Redis에 (만료 시간까지만)
- **Refresh Token Rotation** — 사용할 때마다 새 Refresh Token 발급, 이전 것 폐기

### 3. alg=none 공격
일부 라이브러리는 `alg: none` 토큰을 검증 없이 신뢰하는 버그가 있었음.
- **대응**: 서버에서 허용 알고리즘 명시 (예: HS256만)

### 4. 약한 Secret
HS256은 secret이 짧으면 무차별 대입으로 깨질 수 있음.
- **대응**: 충분히 긴 secret (256bit 이상 권장), 환경변수/Secret Manager로 관리

### 5. HS256 vs RS256 선택
- **HS256 (대칭)**: 같은 secret으로 발급·검증 → 발급자=검증자 단일 서비스에 적합
- **RS256 (비대칭)**: 개인키로 발급, 공개키로 검증 → **MSA**나 외부 서비스가 검증할 때 유리 (공개키만 배포)

## 권장 아키텍처: Access + Refresh 전략

```
[로그인]
  서버 → Access(15분) + Refresh(7일, Redis 저장)
  Access: 응답 body 또는 Authorization 헤더
  Refresh: HttpOnly + Secure + SameSite=Strict 쿠키

[API 호출]
  Authorization: Bearer <access>
  서버: 서명만 검증 (DB 조회 X)

[Access 만료]
  401 → 클라이언트가 /refresh 호출
  서버: 쿠키의 Refresh 검증 + Redis 화이트리스트 확인
  → 새 Access (+ 새 Refresh, 기존 Refresh 폐기 = Rotation)

[로그아웃]
  Redis에서 Refresh 삭제 → 더 이상 재발급 불가
  Access는 만료까지 살아있음 (필요 시 블랙리스트)
```

## Payload에 넣을 것 / 넣지 말 것
- ✅ user_id, role, exp, iat
- ❌ 비밀번호, 주민번호, 결제 정보, 개인 식별 정보(PII)

## 면접 빈출 질문
1. **JWT의 장단점?** → Stateless·확장성 좋지만 즉시 무효화 어려움, payload 노출
2. **왜 Access + Refresh로 나누나?** → 탈취 시 피해 시간 단축 + Refresh로 사용자 경험 유지
3. **Refresh Token을 어디에 저장?** → 클라이언트: HttpOnly·Secure 쿠키 / 서버: Redis 화이트리스트
4. **로그아웃은 어떻게 구현?** → Refresh 삭제(재발급 차단) + 필요시 Access 블랙리스트
5. **JWT를 localStorage에 저장하면 안 되는 이유?** → XSS로 JS가 탈취 가능 (HttpOnly 쿠키와 비교)
6. **HS256과 RS256 차이?** → 대칭 vs 비대칭, MSA·외부 검증은 RS256이 유리
7. **JWT가 정말 stateless인가?** → Refresh·블랙리스트를 쓰는 순간 부분 stateful. 완전 stateless는 즉시 무효화 포기

## 프로젝트 연결
- 위스키/티켓팅: Refresh Token을 Redis에 저장 → 강제 로그아웃·중복 로그인 제어 가능
- MSA 환경에서는 인증 서버에서 RS256으로 서명, 각 서비스는 공개키로 검증
