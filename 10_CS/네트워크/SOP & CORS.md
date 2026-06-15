 ## 출처의 정의

> 출처 : 프로토콜  + 호스트 + 포트로 구성되어 있고, 모두 같아야 동일한 출처로 간주

`https://www.domain.com:443`
- 프로토콜 : `https://`
- 호스트 : `www.domain.com`
- 포트 : `443`

## SOP (Same-Origin-Policy, 동일 출처 정책)

> 어떤 출처에서 불러온 문서나 스크립트가 다른 출처에서 가져온 리소스와 상호작용하는 것을 제한하는 보안 방식

-> 출처가 다를 경우, 서로 간의 문서나 스크립트에 접근할 수 없음

## CORS (Cross-Origin-Resource Sharing, 교차 출처 리소스 공유)

> 프론트엔드에서 서버로 엑세스할 때, 한 출처에서 제공하는 스크립트가 다른 출처의 리소스를 요청할 수 있는 방법을 결정한다.

- CORS 정책은 요청/응답 상호 작용에 포함되어야 하는 특정 HTTP 헤더를 정의한다.
- 프론트에서는 Request Header 에 CORS 관련 옵션을 넣어주고, 서버는 Response Header 에 프론트의 요청을 허용한다는 내용을 넣어주면 된다.

## Preflight 요청

### 단순 요청의 조건
> 보안을 생각하지 않고 브라우저는 바로 요청을 보낼 수 있었다.

**조건**
1. 메서드 : GET,HEAD,POST
2. 헤더 : `Accept`, `Accept-Language`, `Content-Language`, `Content-Type`
3. Content-Type 
	- `application/x-www-form-urlencoded`    
	- `multipart/form-data`    
	- `text/plain`

위의 조건을 벗어나는 요청은 보안을 위해 먼저 교차 출처가 가능한 브라우저인지 확인하는 요청 작업이 있따.

![](../../images/스크린샷%202026-04-12%2016.16.18.png)
### Preflight

1. [브라우저 -> 서버] Preflight(OPTIONS) : 헤더만 담아서 보낸다.
	- Origin : 요청을 보내는 내 주소
	- Access-Control-Request-Method : 보낼 메서드 (ex.POST)
	- Access-Control-Request-Headers : 보낼 커스텀 헤더 (ex.Authorization)
2. [서버 -> 브라우저] Response : 서버가 설정된 CORS 정책에 따라 응답
	- Access-Control-Allow-Origin : 허용된 주소
	- Access-Control-Allow-Methods : 허용된 메서드
	- Access-Control-Max-Age : 허락 결과(OPTIONS)를 캐싱할 시간
		- 같은 브라우저에서 오는 요청을 또 검증하지 않게 하기 위해 캐싱할 수 있음
3. 브라우저 검증 후, 진짜 요청을 보내고 응답해줌

## 웹 통신의 전체 계층도 (Flow)

시간 순서대로 나열하면 CORS가 어디쯤 위치하는지 명확해집니다.

1. **L4 - TCP Handshake:** 서버와 클라이언트가 연결 통로를 뚫습니다. (3-Way Handshake)
    
2. **L6 - TLS Handshake:** 그 통로 위에 암호화 장벽을 세웁니다. (HTTPS 완성)
    
3. **L7 - HTTP (CORS Preflight):** 암호화된 통로를 통해 브라우저가 **`OPTIONS`** 편지를 먼저 보냅니다.
    
4. **L7 - HTTP (Actual Request):** 서버가 허락하면 진짜 데이터가 담긴 편지를 보냅니다.