SSE 실시간 이동 경로 분석에서 SSE 연결을 맺고 있을 때 Nginx 설정할 때 고려해야 할 것들이 뭐가 있는지 살펴보자.

## proxy_buffering

Nginx는 기본적으로 **응답을 버퍼링하기 때문에**, SSE 이벤트가 클라이언트로 실시간 전달되지 않고 버퍼에 쌓였다가 한꺼번에 나가거나 지연된다. 
이렇게 되면 실시간으로 스트림해야 하는 SSE 연결 구조상, 끊임없이 전달되지 안되는 문제가 발생한다.

따라서 별도의 location 블록을 추가해서 SSE 스트림하는 API는 따로 처리를 해야 한다.

```
 location /api/v1/location/stream/ {
      include /etc/nginx/conf.d/service-url.inc;
      proxy_pass $service_url;

      # SSE 필수
      proxy_http_version 1.1;
      proxy_set_header Connection "";      # upstream keepalive 허용 (Connection 헤더 비움)
      proxy_buffering   off;               # ★ 버퍼링 끄기 — 이게 핵심
      proxy_cache       off;
      proxy_read_timeout 3600s;            # 서버 SSE 타임아웃(5분)보다 충분히 길게

      proxy_set_header Host              $host;
      proxy_set_header X-Real-IP         $remote_addr;
      proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
  }
```

`proxy_http_version 1.1`
- 왜 HTTP/1.1인가
	- chunked streaming / keep alive에 필요
	- **주의** : 동시 연결은 최대 6개 병렬이 가능하므로, **보호자가 6명 이상이면 안됨**
- **왜 HTTP/2.2는 안했나**
	- 일단 정책상 보호자 6명 이상안됨. 최대 5명
	- HTTP/2.2는 연결 하나에 멀티플렉싱이 가능해 연결이 다수가 가능함.
		-> 이건 나중에 정책이 바꾸거나 병목이 생길때 2.2로 전환하자

`proxy_set_header Connection "";`
**설정 이유**
- 클라이언트 -> Nginx : Keepalive 정상임
- Nginx -> 웹 서버 : HTTP/1.0 사용하는데, Keepalive을 보장안함. 
	-> SSE 연결이 Nginx에서 웹서버에서도 **Keepalive(연결 파이프라인 유지) 지속하기** 위해서 설정

` proxy_buffering   off` : 버퍼링 끔

#### upstream?
> Nginx 뒤에 연결된 서버를 부르는 말

여기서는 Nginx 뒤에 App이 있으므로 App이 upstream이다.