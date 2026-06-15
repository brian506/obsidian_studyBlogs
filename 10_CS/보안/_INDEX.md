# 보안 — 면접 대비 인덱스

## 핵심 노트
- [[인증과 인가 (Authentication vs Authorization)]] — 세션 vs JWT, RBAC, 쿠키 옵션
- [[암호화와 해시 (대칭·비대칭·솔트)]] — AES/RSA, bcrypt, 디지털 서명
- [[HTTPS와 TLS 핸드셰이크]] — TLS 1.2/1.3, 인증서, Forward Secrecy
- [[웹 취약점 (XSS, CSRF, SQL Injection)]] — OWASP 3종 + 방어
- [[JWT 보안 이슈와 대응]] — Access+Refresh, Redis 화이트리스트, alg=none
- [[OAuth 2.0과 OIDC]] — Authorization Code + PKCE, state, id_token

## 빈출 주제 우선순위
1. **JWT 보안 이슈와 대응** (Refresh Token 저장 위치, 무효화 전략)
2. **XSS / CSRF 차이와 방어** (HttpOnly / SameSite가 각각 막는 것)
3. **OAuth 2.0 흐름** (Authorization Code, PKCE)
4. **비밀번호 저장** (bcrypt와 솔트)
5. **HTTPS / TLS 핸드셰이크 흐름**
6. **SQL Injection 방어** (Prepared Statement)

## 연결되는 다른 영역
- **Spring Security 구현** → [[20_언어·프레임워크/Spring/security]]
- **RefreshToken Redis 저장** → [[20_언어·프레임워크/Spring/security/oauth2.0/RefreshToken 을 Redis 에 저장하는 이유와 방법]]
- **인증서 자동 갱신** → [[30_인프라·운영/Nginx-SSL/Certbot (https 인증서)]]
- **HTTPS 개념** → [[10_CS/네트워크/HTTP VS HTTPS]]
- **CORS** → [[10_CS/네트워크/SOP & CORS]]
