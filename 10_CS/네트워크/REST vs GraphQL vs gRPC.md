# REST vs GraphQL vs gRPC

## 한 줄 비교

| | REST | GraphQL | gRPC |
| --- | --- | --- | --- |
| 스타일 | 리소스 중심 (URL + HTTP 메서드) | 쿼리 언어 | RPC (함수 호출) |
| 전송 | HTTP/1.1, 2 | HTTP/1.1, 2 (보통 POST) | **HTTP/2** |
| 포맷 | JSON, XML | JSON | **Protobuf** (바이너리) |
| 스키마 | OpenAPI(선택) | **내장 SDL** | **.proto 파일 강제** |
| 캐싱 | 표준 (HTTP 캐시) | 어려움 (POST + 단일 엔드포인트) | 직접 구현 |
| 스트리밍 | SSE/WebSocket 별도 | Subscription | **내장 (단/양방향)** |
| 적합 | 일반 웹 API, 외부 공개 | 클라이언트가 필요 데이터 골라야 할 때 | **서비스 간 통신 (MSA)** |

## REST (REpresentational State Transfer)

### 핵심 원칙
- **리소스**를 URL로 식별: `/users/123/orders`
- HTTP 메서드로 행위 표현: GET / POST / PUT / PATCH / DELETE
- **무상태(Stateless)**: 요청에 필요한 모든 정보 포함
- **HATEOAS** (선택): 응답에 다음 액션 링크 포함 (실무엔 거의 미사용)

### 장점
- 단순·표준·도구 풍부 (Swagger, Postman)
- HTTP 캐시 활용 (GET + 캐시 헤더)
- 모든 언어·도구 지원

### 단점
- **Over-fetching**: 필요 없는 필드까지 받음
- **Under-fetching**: 한 화면을 위해 여러 endpoint 호출 (1+N 호출)
- 버전 관리 부담 (`/v1`, `/v2`)

## GraphQL

### 핵심
- **하나의 엔드포인트** (`/graphql`) + 쿼리 언어
- 클라이언트가 **필요한 필드만** 지정해서 요청
- 스키마 강제 (타입 시스템)

```graphql
query {
  user(id: 1) {
    name
    orders(limit: 5) { id, total }
  }
}
```

### 장점
- Over/Under-fetching 해결
- 단일 요청으로 복합 데이터 (REST의 N번 호출 → 1번)
- 강한 타입 + 스키마 자동 문서
- BFF (Backend for Frontend) 패턴에 최적

### 단점
- **HTTP 캐싱 어려움** (POST + 단일 endpoint) → 별도 캐시 레이어
- **N+1 쿼리 문제** 빈발 → **DataLoader** 패턴으로 배치 로딩
- 쿼리 복잡도 폭주 가능 → 깊이 제한·복잡도 분석 필요
- 학습 곡선·서버 구현 복잡

## gRPC (Google RPC)

### 핵심
- **Protocol Buffers (.proto)** 로 스키마 정의 → 다양한 언어로 코드 자동 생성
- HTTP/2 위 바이너리 통신 (작고 빠름)
- 4가지 통신 패턴:
  - Unary (1 요청 → 1 응답)
  - Server Streaming
  - Client Streaming
  - **Bidirectional Streaming**

```protobuf
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc StreamOrders(GetUserRequest) returns (stream Order);
}
```

### 장점
- **빠름·작음**: 바이너리 + HTTP/2 멀티플렉싱
- **타입 안전 + 코드 생성** (계약 우선 개발)
- 양방향 스트리밍 내장
- MSA의 서비스 간 통신에 최적 (Netflix, Google)

### 단점
- 브라우저 직접 호출 어려움 (gRPC-Web으로 우회)
- 사람이 읽기 어려운 바이너리 → 디버깅 도구 필요
- 방화벽·프록시가 HTTP/2 trailers 지원 필요

## 선택 가이드

| 상황 | 추천 |
| --- | --- |
| **외부 공개 API, 단순함** | **REST** |
| **모바일·SPA가 화면별로 다른 데이터 조합 필요** | **GraphQL** |
| **MSA 내부 서비스 통신, 성능 중요** | **gRPC** |
| **실시간 양방향** | gRPC (스트리밍) 또는 WebSocket |
| **공개 API + 일부 복잡 화면** | REST + GraphQL 하이브리드 |

## REST API 잘 설계하기 (자주 묻는 부분)

### 1. URL은 명사 + 복수형
```
좋음:  GET  /users/123/orders
       POST /users/123/orders
       DELETE /orders/456

나쁨:  GET  /getUser?id=123
       POST /deleteOrder
```

### 2. HTTP 상태 코드 정확히
- 200 OK / 201 Created / 204 No Content
- 400 Bad Request / 401 Unauthorized / 403 Forbidden / 404 Not Found / 409 Conflict
- 500 Internal Server Error / 503 Service Unavailable

### 3. PUT vs PATCH
- **PUT**: 리소스 **전체 교체** (멱등)
- **PATCH**: 일부 수정

### 4. 멱등성 (Idempotency)
- 여러 번 호출해도 결과가 같음
- GET, PUT, DELETE: 멱등 / POST, PATCH: 비멱등 (보통)
- 결제·주문 같은 비멱등 API는 **Idempotency-Key 헤더**로 중복 방어

## 면접 빈출 질문
1. **REST의 핵심 원칙?** → 리소스 + URL + HTTP 메서드 + 무상태. (HATEOAS는 선택)
2. **GraphQL이 REST보다 좋은 점?** → Over/Under-fetching 해결, 단일 요청으로 복합 데이터, 강타입 스키마
3. **GraphQL의 단점?** → HTTP 캐시 어려움, N+1 쿼리, 학습 곡선, 쿼리 복잡도
4. **gRPC가 REST보다 빠른 이유?** → Protobuf 바이너리 + HTTP/2 멀티플렉싱 + 헤더 압축
5. **gRPC는 언제 쓰나?** → MSA 내부 서비스 간 통신, 실시간 스트리밍
6. **PUT과 PATCH 차이?** → 전체 교체 vs 부분 수정
7. **멱등성이란? 왜 중요?** → 같은 요청을 N번 보내도 결과 동일. 네트워크 재시도 안전성

## 프로젝트 연결
- 외부 클라이언트용 = REST + JSON (대부분의 프로젝트)
- MSA 내부 서비스 간 = gRPC 후보 (위스키 프로젝트의 서비스 분리)
- 모바일이 복잡한 데이터 조합을 원하면 GraphQL BFF
