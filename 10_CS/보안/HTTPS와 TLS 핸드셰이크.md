# HTTPS와 TLS 핸드셰이크

## HTTP vs HTTPS
- **HTTP**: 평문 통신 → 도청·변조·신원 위조 가능
- **HTTPS**: HTTP + **TLS(Transport Layer Security)** → 기밀성 + 무결성 + 인증

TLS는 **SSL의 후속**. 흔히 SSL/TLS라 부르지만, 실제 사용 중인 건 TLS 1.2 / 1.3.

## TLS가 제공하는 것
1. **기밀성(Confidentiality)** — 대칭 암호화로 도청 방지
2. **무결성(Integrity)** — MAC으로 변조 감지
3. **인증(Authentication)** — 인증서로 서버(필요 시 클라이언트) 신원 확인

## TLS 1.2 핸드셰이크 (요약)
1. **ClientHello** — 클라이언트 → 서버: 지원 TLS 버전, 암호 스위트, 클라이언트 랜덤
2. **ServerHello** — 서버 → 클라이언트: 선택한 암호 스위트, 서버 랜덤, **서버 인증서**(공개키 포함)
3. **인증서 검증** — 클라이언트가 CA 체인 검증 → 신뢰 가능한지
4. **키 교환** — 클라이언트가 **Pre-Master Secret**을 서버 공개키로 암호화해 전송 (또는 ECDHE 키 교환)
5. **세션키 생성** — 양쪽 모두 랜덤 + Pre-Master로 **대칭 세션키** 생성
6. **Finished** — 이후 모든 통신은 세션키로 대칭 암호화

→ **2-RTT** (왕복 2번)

## TLS 1.3의 개선
- 핸드셰이크 **1-RTT** (재방문 시 **0-RTT**까지 가능)
- 취약한 알고리즘(RSA 키 교환, RC4, MD5 등) 제거
- **Forward Secrecy 강제**: 서버 개인키가 유출돼도 과거 트래픽 복호화 불가 (ECDHE 키 교환 사용)

## 인증서 (X.509)
- 서버의 **공개키 + 도메인 + 발급자(CA) 서명**이 담긴 문서
- **CA(Certificate Authority)** 가 신뢰의 뿌리. 브라우저/OS에 루트 CA 인증서가 내장
- 체인: **루트 CA → 중간 CA → 서버 인증서** — 클라이언트가 위쪽으로 검증

### 인증서 종류
- **DV (Domain Validated)**: 도메인 소유만 확인 (Let's Encrypt)
- **OV (Organization Validated)**: 조직 확인
- **EV (Extended Validation)**: 가장 엄격, 주소창에 회사명 표시

## Let's Encrypt / Certbot
- 무료 DV 인증서. 90일 만료 → **자동 갱신** 필요
- recaring 프로젝트의 Nginx-Certbot 자동 갱신 참고

## Forward Secrecy (전방향 안전성)
- 세션마다 **임시 키 쌍**을 만들어 교환 (ECDHE)
- 장기 개인키가 나중에 유출돼도 과거 세션을 복호화할 수 없음

## HSTS (HTTP Strict Transport Security)
- 응답 헤더: `Strict-Transport-Security: max-age=31536000`
- 브라우저가 이후 해당 도메인은 무조건 HTTPS로 접속 → HTTP 다운그레이드 공격 방어

## 면접 빈출 질문
1. **HTTP와 HTTPS의 차이?** → 평문 vs TLS 암호화. 기밀성·무결성·인증 제공
2. **TLS 핸드셰이크 과정?** → ClientHello → ServerHello+인증서 → 키 교환 → 세션키 합의 → 대칭 통신
3. **왜 대칭과 비대칭을 둘 다 쓰나?** → 비대칭은 키 교환만 (안전), 본 데이터는 빠른 대칭으로
4. **인증서의 역할? CA란?** → 서버 신원과 공개키를 보증하는 문서. CA가 서명, 브라우저가 루트 CA 신뢰
5. **TLS 1.3이 1.2보다 좋은 점?** → 1-RTT, 0-RTT, 취약 알고리즘 제거, Forward Secrecy 강제
6. **Forward Secrecy가 무엇이고 왜 중요?** → 임시 키 사용. 장기 개인키 유출돼도 과거 통신 안전
7. **HSTS의 역할?** → 브라우저에게 HTTPS 강제 → SSL Strip 방어

## 프로젝트 연결
- recaring/위스키 프로젝트: Nginx + Certbot 으로 Let's Encrypt 인증서 자동 갱신
- ALB + ACM: AWS에서 인증서 발급·갱신 자동화
