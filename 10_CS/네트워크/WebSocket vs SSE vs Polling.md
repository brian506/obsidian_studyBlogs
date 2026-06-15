# WebSocket vs SSE vs Polling

서버 → 클라이언트로 **실시간 데이터를 보내는 4가지 방식**의 비교.

## 0. 왜 필요한가
HTTP는 기본적으로 **클라이언트가 요청해야** 서버가 응답하는 단방향 구조. 서버가 먼저 알려주려면 우회 방법 필요.

## 1. Polling (단기 폴링)
일정 주기로 클라이언트가 서버에 묻기.
```
[every 5s] GET /messages
```
- 장점: 단순, HTTP 그대로
- 단점: **지연(주기만큼) + 빈 응답이 많아 낭비**
- 적합: 변경 빈도 낮은 데이터 (예: 5분마다 갱신)

## 2. Long Polling
서버가 **변경이 있을 때까지** 응답을 미루고, 응답하면 클라이언트가 즉시 다시 요청.
```
GET /messages    → (서버 대기) → 응답 → 즉시 다시 GET
```
- 장점: 실시간성 ↑, 일반 HTTP라 호환성 좋음
- 단점: 연결당 서버 자원 점유, 요청·응답 반복 오버헤드

## 3. SSE (Server-Sent Events)
**서버 → 클라이언트 단방향** 스트리밍. HTTP/1.1 위에서 `text/event-stream` 컨텐츠 타입으로 연결을 열어둠.

```
GET /events HTTP/1.1
Accept: text/event-stream

← data: {"id":1, "msg":"hi"}\n\n
← data: {"id":2, "msg":"bye"}\n\n
```

- 장점:
  - **표준 HTTP** — 프록시·방화벽 친화
  - 브라우저 자동 재연결 + Last-Event-ID로 이어받기
  - 구현 단순
- 단점:
  - **단방향** (서버 → 클라이언트만)
  - 바이너리 미지원 (텍스트만)
  - HTTP/1.1은 도메인당 6개 연결 제한 → HTTP/2면 해결

## 4. WebSocket
HTTP `Upgrade` 로 **TCP 연결을 양방향 채널로 승격**. 이후엔 HTTP가 아닌 **ws://** 프로토콜로 작은 프레임을 주고받음.

```
Client: GET /ws HTTP/1.1
        Upgrade: websocket
        Connection: Upgrade
        Sec-WebSocket-Key: ...

Server: 101 Switching Protocols
        Upgrade: websocket
```

- 장점:
  - **양방향, 저지연, 저오버헤드**
  - 텍스트 + 바이너리
  - 게임·채팅·협업 도구에 표준
- 단점:
  - 일반 HTTP 캐시·프록시 미적용 (별도 설정)
  - 재연결·하트비트 등 **상태 관리 직접 구현**
  - 로드밸런서 sticky session 또는 메시지 브로커(Pub/Sub) 필요
  - 일부 환경에서 차단

## 비교표

| | Polling | Long Polling | SSE | WebSocket |
| --- | --- | --- | --- | --- |
| 방향 | 클→서 (재요청) | 클→서 (대기) | **서→클** | **양방향** |
| 프로토콜 | HTTP | HTTP | HTTP | **WS (Upgrade)** |
| 실시간성 | 낮음 (주기 지연) | 중간 | **높음** | **높음** |
| 서버 부하 | 낮음 | 중간 | 낮음 | 낮음 (긴 연결 유지) |
| 양방향 | X | X | X | **O** |
| 바이너리 | O | O | X | **O** |
| 자동 재연결 | 자동 (다음 요청) | 자동 | **내장** | 직접 구현 |
| 브라우저 호환 | 모두 | 모두 | 모던 브라우저 | 모던 브라우저 |
| 적합 | 낮은 빈도 갱신 | 호환성 우선 실시간 | **알림·스트림 (단방향)** | **채팅·게임·협업 (양방향)** |

## 선택 기준
1. **클라이언트가 서버로도 자주 보내야 한다** → WebSocket
2. **서버 → 클라이언트만, HTTP 친화 환경 (CDN·프록시)** → SSE
3. **호환성·단순함이 중요, 실시간성은 적당히** → Long Polling
4. **변경 빈도 낮음** → Polling

## STOMP
- WebSocket 위의 **메시지 프로토콜** (subscribe/send/topic 같은 추상화)
- Spring의 `@MessageMapping` 으로 채팅·실시간 알림 구현
- WebSocket 자체는 메시지 라우팅이 없음 → STOMP가 그 위에 표준 메시징을 얹는 것
- 자세히는 [[웹소켓(WebSocket)과 STOMP]] 참고

## SSE의 부활
- 한동안 WebSocket이 압도적이었으나, 최근 **AI 스트리밍 응답**(ChatGPT 등)으로 SSE 재조명
- 단방향이면 SSE가 더 가볍고 단순. HTTP 인프라(인증·캐시·프록시) 그대로 활용 가능

## 면접 빈출 질문
1. **Polling, Long Polling, SSE, WebSocket 차이?** → 방향·실시간성·프로토콜 기준으로 비교
2. **WebSocket이 HTTP와 다른 점?** → Upgrade로 양방향 TCP 채널 승격, 이후엔 HTTP 아닌 프레임 통신
3. **SSE를 쓰는 이유?** → 단방향이면 충분하고 HTTP 인프라 친화 + 자동 재연결 + 단순함
4. **WebSocket 다중 서버 환경의 어려움?** → 한 사용자가 어느 서버에 연결됐는지 모름 → **Redis Pub/Sub, Kafka** 같은 메시지 브로커로 메시지 라우팅
5. **STOMP가 무엇?** → WebSocket 위의 메시지 프로토콜. subscribe/send/topic 추상화
6. **AI 스트리밍은 왜 SSE인가?** → 서버→클라이언트 단방향 + HTTP 인프라 활용 + 단순. 양방향 필요 없음

## 프로젝트 연결
- 위스키 채팅 (WebSocket + STOMP + Redis Pub/Sub) → 다중 서버 메시지 라우팅
- recaring의 SSE 부하 테스트 → 단방향 알림은 SSE로 충분, 동시 연결 수가 핵심 지표
