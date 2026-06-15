# 네트워크 — 면접 대비 인덱스

## 기존 노트
- [[네트워크 기초]] — 처리량, 지연, 토폴로지
- [[OSI 7계층 모델]]
- [[TCP - IP 4계층]] — 3-way / 4-way handshake
- [[꼭 알아야 할 네트워크 개념]] — 공인/사설 IP, 서브넷, NAT, LAN/WAN/VPN
- [[DNS 통신]]
- [[HTTP VS HTTPS]]
- [[SOP & CORS]]
- [[쿠키와 세션]]
- [[로드밸런서]]
- [[프록시 서버]]
- [[CDN?]]
- [[웹소켓(WebSocket)과 STOMP]]

## 새로 추가 (면접 빈출 + 프로젝트 활용)
- [[TCP vs UDP]] — 비교표, UDP 사용 사례, QUIC 연결
- [[TCP 신뢰성·흐름제어·혼잡제어]] — Sliding Window, Slow Start, HOL Blocking
- [[HTTP 1.1 vs 2 vs 3 (QUIC)]] — 멀티플렉싱, 0-RTT, HOL 해소
- [[REST vs GraphQL vs gRPC]] — 선택 기준, REST 설계, 멱등성
- [[WebSocket vs SSE vs Polling]] — 실시간 통신 4종 비교

## 빈출 주제 우선순위
1. **HTTP / HTTPS 차이 + TLS 핸드셰이크** (보안 노트와 연결)
2. **TCP 3-way / 4-way handshake** (TIME-WAIT 이유 등)
3. **TCP vs UDP + 각 사용 사례**
4. **HTTP 1.1 vs 2 vs 3** (멀티플렉싱, HOL Blocking, QUIC)
5. **CORS / SOP** (브라우저 동작 원리, preflight)
6. **REST API 설계 + 멱등성**
7. **WebSocket vs SSE vs Polling** (선택 기준)
8. **로드밸런서·프록시·CDN 차이**

## 자주 묻는 시나리오
- "브라우저 주소창 URL 입력 후 화면 표시까지?" → DNS → TCP → TLS → HTTP 요청·응답 → 렌더링
- "API 호출했더니 CORS 에러" → SOP, preflight(OPTIONS), Access-Control 헤더
- "실시간 알림 어떻게 구현?" → SSE / WebSocket 선택 기준
- "MSA 서비스 간 통신" → REST vs gRPC

## 연결되는 다른 영역
- **TLS·HTTPS·인증서** → [[10_CS/보안/HTTPS와 TLS 핸드셰이크]]
- **XSS·CSRF·SQLi** → [[10_CS/보안/웹 취약점 (XSS, CSRF, SQL Injection)]]
- **로드밸런서 구현** → [[30_인프라·운영/Nginx-SSL]], [[40_AWS/Load Balancing & Auto Scaling Groups]]
- **WebSocket 구현** → 위스키 프로젝트 chat-service
- **SSE 부하 테스트** → recaring 프로젝트
