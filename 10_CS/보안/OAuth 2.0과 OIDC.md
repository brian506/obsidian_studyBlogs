# OAuth 2.0과 OIDC

## 한 줄 정의
- **OAuth 2.0**: 사용자 자원에 **제3자 앱이 접근할 권한을 위임** 받는 **인가(Authorization)** 프로토콜
- **OIDC (OpenID Connect)**: OAuth 2.0 위에 **인증(Authentication)** 표준을 얹은 것 → "이 사용자가 누구인지" 검증 가능

요약하면 **OAuth = 권한 위임**, **OIDC = OAuth + 로그인(누구인지)**.

## 등장 배경
"제3자 앱에 내 비밀번호를 줘야 했던" 문제 해결.
→ 비밀번호 대신 **Access Token**을 발급해 권한만 위임.

## 4가지 역할
- **Resource Owner**: 사용자 (자원의 주인)
- **Client**: 제3자 앱 (예: 우리 서비스)
- **Authorization Server**: 토큰 발급 (Google·Kakao 인증 서버)
- **Resource Server**: 보호된 API (Google Drive, Kakao Profile 등)

## 4가지 Grant Type
| Grant Type | 용도 | 추천 여부 |
| --- | --- | --- |
| **Authorization Code** | 서버 백엔드가 있는 웹 앱 | ✅ 표준 |
| **Authorization Code + PKCE** | SPA·모바일 (client_secret 보관 불가) | ✅ SPA/모바일 표준 |
| Implicit | 과거 SPA용 (토큰을 URL 프래그먼트로) | ❌ Deprecated |
| Resource Owner Password | 사용자 비번 직접 받음 | ❌ 피할 것 |
| **Client Credentials** | 서버 ↔ 서버 (사용자 없음) | ✅ M2M 용 |

→ 사람의 로그인이라면 사실상 **Authorization Code (+PKCE)** 만 기억하면 됨.

## Authorization Code Flow (가장 많이 쓰는 흐름)

```
사용자          Client(우리 서버)        Auth Server(Google)        Resource Server
  │
  │  1. 로그인 요청
  ├────────────────►│
  │                 │ 2. /authorize 로 리다이렉트
  │◄────────────────┤   (client_id, redirect_uri, scope, state)
  │
  │ 3. Google 로그인 + 동의
  ├──────────────────────────────────►│
  │                                   │
  │ 4. redirect_uri?code=AUTH_CODE&state=...
  │◄──────────────────────────────────┤
  │                 │
  │ 5. code 전달
  ├────────────────►│
  │                 │ 6. /token  (code + client_secret)
  │                 ├──────────────────►│
  │                 │                   │
  │                 │ 7. Access Token (+ Refresh + id_token)
  │                 │◄──────────────────┤
  │                 │
  │                 │ 8. Bearer Access Token
  │                 ├──────────────────────────────────────►│
  │                 │                                       │ 9. 사용자 정보
  │                 │◄──────────────────────────────────────┤
```

핵심: **Authorization Code**는 짧고 한 번만 사용 가능. **client_secret + code 교환**으로 토큰을 받는 것이 안전한 이유 = 토큰이 URL/브라우저에 직접 노출되지 않음.

## PKCE (Proof Key for Code Exchange)
- SPA·모바일은 **client_secret을 안전하게 보관 불가** → 대신 PKCE
- 동작:
  1. 클라이언트가 랜덤 `code_verifier` 생성
  2. `code_challenge = SHA256(code_verifier)` 를 `/authorize` 요청에 첨부
  3. 토큰 교환 시 `code_verifier` 원본을 함께 전송 → 서버가 SHA256 비교
- 효과: 인가 코드가 탈취돼도 verifier 없이는 토큰 못 받음

## OIDC가 추가하는 것
- **id_token**: 사용자 정보가 담긴 **JWT** (sub, email, name 등) → "누가 로그인했는지" 검증
- **UserInfo 엔드포인트**: 추가 사용자 정보 조회
- 표준화된 **scope**: `openid`, `profile`, `email`

→ OAuth 2.0만으로는 "이 토큰을 가진 사람이 누구인지" 모름. OIDC는 그 정보를 표준화해서 줌.

## state 파라미터 (CSRF 방어)
- 요청 시 랜덤 `state` 생성 → 콜백에서 같은 값인지 확인
- 다른 사이트가 임의로 콜백 URL을 호출해 사용자를 다른 계정으로 연결하는 공격 방어

## scope (권한 범위)
- 클라이언트가 요청하는 권한의 종류 (`profile`, `email`, `drive.readonly` 등)
- **최소 권한 원칙**: 필요한 것만

## Refresh Token 처리
- OAuth 표준엔 Refresh Token이 있음
- Google·Kakao는 Refresh Token을 처음 동의 시에만 줌 → 안전하게 저장 필요

## 면접 빈출 질문
1. **OAuth 2.0이 풀려는 문제?** → 제3자 앱에 비밀번호를 주지 않고 제한된 자원 접근 권한 위임
2. **OAuth 2.0과 OIDC 차이?** → OAuth=인가, OIDC=OAuth + 인증(누구인지). OIDC는 id_token 발급
3. **왜 Authorization Code 방식이 안전한가?** → 토큰이 브라우저 URL에 노출되지 X, code → token 교환은 서버 간 통신
4. **PKCE가 왜 필요한가?** → SPA·모바일은 client_secret 보관 불가 → code 탈취 시에도 토큰 받지 못하게 함
5. **state 파라미터의 역할?** → CSRF 방지 (콜백이 우리가 시작한 요청인지 확인)
6. **Authorization Code와 Access Token의 차이?** → 코드는 짧고 한번만 쓰는 교환용 / 토큰은 실제 API 호출용
7. **소셜 로그인은 OAuth인가 OIDC인가?** → "로그인" 목적이라면 OIDC가 정답. 단순 권한 위임은 OAuth

## Spring Security와 연결
- `spring-boot-starter-oauth2-client`: 클라이언트 (소셜 로그인) 구현
- `spring-boot-starter-oauth2-resource-server`: JWT 검증으로 보호된 API 노출
- 우리 서버는 보통 **인증 = OIDC로 사용자 식별 → 우리 JWT 발급** 구조

## 프로젝트 연결
- 위스키 프로젝트: Google/Kakao OAuth 로그인 + 추가 정보 입력 단계 → OIDC로 식별 후 자체 JWT 발급
