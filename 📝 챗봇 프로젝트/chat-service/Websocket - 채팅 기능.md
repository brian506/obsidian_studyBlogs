#websocket #MSA #project 

# 개요
타입스크립트와 연동하여 **네이티브 웹소켓**을 이용해서 채팅기능을 구현하고자 한다.
사용자의 요청이 **Gateway 를 거쳐서** 채팅 서버로 가기 위한 과정에서 처리해야 할 것이 _key point_ 이다.
___
# 목표

![사진](인프라%20컴포넌트/image/socketconn.png)
# 흐름
1. **채팅방 생성** : 메시지를 보내기 위한 채팅방 먼저 생성
2. **브라우저** : `ws://localhost:19091/chat` 으로 WS 업그레이드 요청(게이트웨이)
3. **게이트웨이** : 라우팅/프록시, CORS 처리 -> Chat-service 로 터널링
4. **Chat-service** : Origin 화이트리스트로 핸드셰이크 허용(101)
5. **클라이언트** : `STOMP CONNECT` (Authorization: Bearer ~) 전송
6. **서버** : `StompHandler` 에서 JWT 검증(setUser(auth))
7. **WebSocketSecurity** : `CONNECT/SUBSCRIBE/SEND` 심사
8. **Subscribe** : `/topic/**` 으로 브로커 목적지 구독
9. **Publish** : `/pub/**` 컨트롤러 핸들러 -> 서비스 로직 -> 브로커로 publish
10. Stomp 설정으로 유지/복구
11. 종료
___
## 게이트웨이에서 Websocket 업그레이드 & 라우팅
1. **브라우저 -> 게이트웨이**
	- `GET /chat`

2. **게이트웨이 라우트 매칭**
	- `ws://chat-service:8081` 로 라우팅
	- 업그레이된 채널을 `chat-service`로 터널링

3. **인증 헤더 유무 체크**
	```java
	// 헤더 존재 여부 확인  
	String token = resolveToken(request);  
	if (token == null) {  
    return onError(exchange, "Token is missing", HttpStatus.UNAUTHORIZED);  
	}  
  
	// 웹소켓 연결은 인증 헤더만 체크 후  다음 필터로 넘어감  
	if(path.startsWith(WEBSOCKET_URL)){  
    log.info(">>> Gateway AuthenticationFilter 웹소켓 연결: {}", path);  
    return chain.filter(exchange);  
	}
	```
	- 게이트웨이에서는 토큰 유무만 확인하고 STOMP 연결에 대한 인증/인가는 `StompHandler` 에서 처리

4. **CORS는 게이트웨이에서만 처리**
```yml
globalcors:  
  add-to-simple-url-handler-mapping: true  
  corsConfigurations:  
    '[/**]':  
      allowedOrigins:  
        - http://127.0.0.1:5500  
      allowedMethods: [ GET, POST, OPTIONS ]  
      allowedHeaders: [ Authorization, Content-Type ]  
      allowCredentials: true  
      maxAge: 3600  
  default-filters:  
    - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
``` 
- 웹소켓에서도 CORS 를 설정하면 충돌이 일어나기 때문에 게이트웨이에서만 설정
- SecurityConfig 에서 CorsConfig 를 이용하지 않고 yml 에서 처리
___
## chat-service의 핸드셰이크 수락

```java
// WebsocketConfig class...

@Override  
public void configureMessageBroker(MessageBrokerRegistry config) {  
    config.enableSimpleBroker(sub); // client 가 구독할 경로  
    config.setApplicationDestinationPrefixes(pub); // client 가 서버로 메시지를 보낼 때 필수 경로 설정(서버가 메시지를 처리하는 경로)  
}  
  
@Override  
public void registerStompEndpoints(StompEndpointRegistry registry) {  
    // todo 웹소켓에 접속허용할 주소 설정  
    registry.addEndpoint(chatUrl)  
            .setAllowedOriginPatterns("http://127.0.0.1:5500"); // cors 검사가 아닌 origin 핸드셰이크 허  
    // sockJS() 안쓰므로 네이티브 웹소켓임  
  
}
```
- 서버에서 CORS 검사가 아닌 최소한의 Origin 허용을 위해서 `setAllowedOriginPatterns("http://127.0.0.1:5500")` 허용
- 허용되면 **101 Switching Protocols** 로 WS 연결 성립

## STOMP CONNECT & JWT 검증
1. 연결 성립되면 클라이언트가 **`STOMP CONNECT`프레임 전송**
	- 이 프레임에 JWT 헤더 포함

2. `StompHandler` 가 CONNECT 프레임 가로채서 JWT 검증
	- `accessor.setUser(Authentication)` 으로 STOMP 세션에 인증 주체 지정
	- 이후 모든 메시지 처리는 이 인증 과정을 성공한 사용자들에게만 허용

```java

// StompHandler ...

private static final String USER_ID_KEY = "stompUserId";  
    
@Override  
public Message<?> preSend(Message<?> message, MessageChannel channel) {  
    StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);  
  
    // stomp connect 명령어 일때만  
    if (StompCommand.CONNECT.equals(accessor.getCommand())) {  
        // stomp 헤더에서 jwt 추출 - 클라이언트가 메시지 전송 시 헤더에 토큰 받아서 보내야됨  
        log.info("CONNECT 요청 수신. Authorization 헤더: {}", accessor.getFirstNativeHeader("Authorization"));  
        String authHeader = accessor.getFirstNativeHeader("Authorization");  
  
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {  
            throw new IllegalArgumentException("토큰이 없습니다.");  
        }  
        String token = authHeader.substring(7);  
  
        Authentication authentication = jwtUtil.getAuthentication(token);  
        var principal = (StompPrincipal) authentication.getPrincipal();  
        accessor.setUser(principal);  
        // 다른 프레임에서도 해당 인증 객체를 사용하기 위해 세션에 저장해둠  
        accessor.getSessionAttributes().put(USER_ID_KEY, principal.getUserId());  
  
        accessor.setLeaveMutable(true);  
        log.info("✅ STOMP CONNECT 인증 성공: user={}", authentication.getName());  
    }  
    return message;  
}
```
- 다른 프레임에서도 인증 권한을 불러오기 위해 세션에 KEY 를 이용하여 따로 저장
___
## 메시지 권한 심사

1. **WebsocketConfig** 클래스에 `StompHandler`지정
```java
// WebSocketConfig class...

// stomp 메시지 처리 전, 인증을 위한 인터셉터 등록  
@Override  
public void configureClientInboundChannel(ChannelRegistration registration) {  
    registration.interceptors(stompHandler);  
}
```
- 모든 STOMP 프레임은 `clientInboundChannel(WebSocketConfig)` → `ChannelSecurityInterceptor(StompHandler)`를 거치며  **`AuthorizationManager<Message<?>>(WebSocketSecurity)**` 규칙으로 허용/차단**

### WebSocketSecurityConfig
```java
@Bean  
public AuthorizationManager<Message<?>> messageAuthorizationManager() {  
    // ★ 빈 주입 대신 직접 생성  
    MessageMatcherDelegatingAuthorizationManager.Builder messages =  
            MessageMatcherDelegatingAuthorizationManager.builder();  
  
    return messages  
            .nullDestMatcher().authenticated()                 // CONNECT 등 목적지 없는 프레임  
            .simpDestMatchers("/pub/**").hasAuthority("ROLE_GENERAL")      // 클라 → 서버 전송  
            .simpSubscribeDestMatchers("/topic/**").hasAuthority("ROLE_GENERAL")  
            .simpTypeMatchers(  
                    SimpMessageType.CONNECT,  
                    SimpMessageType.DISCONNECT,  
                    SimpMessageType.HEARTBEAT,  
                    SimpMessageType.OTHER  
            ).permitAll()                                      // 기술 프레임 허용  
            .anyMessage().denyAll()  
            .build();  
}
```
___
## 구독 및 발행
1. *채팅방 입장* : 인증 성공한 사용자가 `{roomId}`의 채팅방으로 입장
2. *SUBSCRIBE* :  메시지 전송 처리 이전에 먼저 **특정 채널 `topic/chat/room/{roomId}` 을 구독** - 프론트의 역할
	- 해당 세션을 '구독자 목록'에 등록
	- 이후 메시지가 발행되면 세션으로 전달
	- 로그 : `User subscribed to destination: /topic/chat/room/68d56d654889797df6d82006`

3. *SEND* : 사용자가 '전송'을 누르면 `SEND` 프레임을 `/pub/send/message` 주소로 전송

4. *createMessage()* : 전송하는 메시지를 DB 에 저장

5. *MESSAGE* : `SEND` 프레임으로 들어온 메시지를 받아서 **해당 채널을 `SUBSCRIBE`한 세션들에게 `MESSAGE`프레임을(messagingTemplate.convertAndSend()) 전송**
	- `topic/rooms/123` 주소로 메시지 보냄

6. *Broadcast* : STOMP 브로커가 `topic/rooms/123` 주소를 구독하고 있는 사용자들에게 전송

```java
// PublishService class ....

private final SimpMessagingTemplate messagingTemplate;  
private final String MESSAGE_DEST_URL = "/topic/chat/room/";  
private final String ROOM_DEST_URL = "/topic/rooms/";
  
// 메시지 발행  
public void publishMessage(final MessageBroadcastResponse response){  
    messagingTemplate.convertAndSend(MESSAGE_DEST_URL + response.roomId(),response);  
}

// MessageService ....
// 메시지 전송  
public void createMessage(final SendMessageEvent message,final String senderId){  
    Message messageToSave = Message.saveMessage(message, senderId);  
    Message savedMessage = messageRepository.save(messageToSave);  
    MessageBroadcastResponse response = Message.toBroadCastResponse(savedMessage,senderId);  
    publishService.publishMessage(response);  
}
```
___
# SockJS 가 아닌 네이티브 WebSocket 을 쓴 이유
CORS 정책은 Gateway의 프록시 장점을 살리기 위해서 어떻게든 Gateway 에서 처리하고 싶었다.
SockJS() 를 사용하기 되면 폴백 HTTP 기반이므로 게이트웨이 CORS 정책과 계속해서 충돌 및 인증 이슈가 발생했다. 

네이티브 WebSocket 으로 하면 CORS 설정없이 핸드셰이크의 Origin 체크가 핵심이므로 게이트웨이에서만 CORS 를 관리하게 하는 구현이 쉬워졌다.
다만 주의해야 할 점이 **폴백이 없어서 WebSocket 을 반드시 허용해줘야 한다.(방화벽 등 설정)**

### 폴백 HTTP? (SockJS)
**폴링** :  클라이언트가 일정 주기마다 `HTTP 요청` 을 보내서 새로운 메시지가 있는 지 확인하는 방식
- *장점* : 평범한 HTTP 요청이므로 **방화벽/프록시 환경에서도 거의 다 동작**
- *단점* : 요청/응답이 자주 일어나 **네트워크 오버헤드, 지연 큼**

### 네이티브 WebSocket 
: 첫 HTTP 요청 이후에 TCP 지속적으로 연결

 - *장점* : **낮은 지연과 오버헤드**로 많은 메시지를 빠르게 주고 받을 수 있음
 
	클라이언트당 1개씩 연결되므로 서버 스케일링 필요
 
-  *단점* : 방화벽이나 프록시는 지속적으로 연결되어 있으면 **비정상 트래픽으로 간주**
 
	 방화벽에서 `Websocket Upgrade`를 허용해주고
	 AWS 에서 80/443 포트를 열어두고 `HTTP/HTTPS over WebSocket` 으로 허용해야 함