#네트워크 #websocket #spring 

# 개요

많이 쓰이는 HTTP 통신은 요청을 보내야만 응답할 수 있는 **단방향 구조**이다.
HTTP 요청을 보내면 응답이 올때까지 쓰레드는 다른 일을 하지 못하고 기다려야한다.(동기 방식)

이러한 한계를 해결하기 위해  한번의 TCP 연결을 맺고 **양방향 구조**로 요청/응답을 할 수 있는 웹소켓을 쓴다.

채팅 기능을 구현하기 위해서 어떤 방식으로 구현하면 좋을 지 알아보자.
___

## 동작 방식

1. **양방향 통신**
	- 실시간으로 서버와 클라이언트가 동시에 데이터를 보낼 수 있음

2. **연결 유지**
	- 최초 연결 시 한번의 *Handshake 과정* 이후에 TCP 연결 유지

- Handshake 과정 : **HTTP 요청으로 시작**해서 **웹소켓으로 upgrade** 되는 방식
	  1.  _client 요청_ : 일반 HTTP 요청 헤더에 `Upgrade : websocket` 담아서 보냄
	  2. _서버의 응답_ : 서버가 요청을 받고 웹소켓 연결 허용하면 HTTP 상태 코드 `101 Switching Protocols 응답
	  3. _연결 수립_ : 기존의 HTTP 연결은 웹소켓 연결로 전환

___

## 채팅에서의 동작 순서 (WebSocket 으로만 구현)

#### 1. 연결 수립
- 사용자가 채팅방에 입장하면, 클라이언트는 웹소켓 주소(`ws://~`) 로 연결 요청한다.
- 서버는 Handshake 로 연결 수락 후 사용자의 연결 세션을 메모리에 저장해둔다.

#### 2. 메시지 송신
- 사용자가 메시지를 보내면 클라이언트는 JSON 형식의 데이터를 자신의 웹소켓 통로를 통해 서버로 보낸다.
	ex) `{ "roomId": "123", "sender": "user_A", "content": "안녕하세요" }

#### 3. 서버의 수신 및 브로드캐스팅
- 서버는 메시지를 수신한다.
- 서버는 메시지 내용의 **roomId 기반**으로 접속해 있는 다른 사용자들을 찾아서 **사용자들의 웹소켓 통로**를 통해서 메시지를 보낸다. -> **브로드캐스팅**

#### 4. 클라이언트의 수신 및 화면 표시
- 사용자의 클라이언트는 자신의 웹소켓 통로로 서버가 보낸 메시지를 수신한다.

___

# STOMP

- 메시지 전송을 효율적으로 하기 위해 나온 프로토콜의 종류이며 `PUB/SUB` 구조로 **메시지를 송신하고 수신 처리**하는 부분을 쉽게 구현할 수 있다.

![[스크린샷 2025-09-10 오후 5.11.28.png]]


## 코드를 통한 동작 순서 구현

```java
@Configuration  
@EnableWebSocketMessageBroker  
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {  
  
    @Override  
    public void configureMessageBroker(MessageBrokerRegistry config) {  
        config.enableSimpleBroker("/sub");  
        config.setApplicationDestinationPrefixes("/pub");  
    }  
  
    @Override  
    public void registerStompEndpoints(StompEndpointRegistry registry) {  
        // todo 웹소켓에 접속허용할 주소 설정  
        registry.addEndpoint("/chat").setAllowedOrigins("*")  
                .withSockJS();  
    }  
}
```

  - `ws://localhost:19091/chat` : WebSocket **접속 주소** 
	- API Gateway 를 이용해서 모든 요청을 처리하므로 클라이언트가 보낼때는  **`19091` 포트 번호 사용**
	- Chat-server 포트번호 : 8081
	- API Gateway 가 요청을 받으면 **내부적으로 Chat-server 로 라우팅**
- 서버에 접속 후 메시지 전송 주소(Destination) : `/pub/chat/sendMessage`
	- 앞에 http 나 ws 안붙음
- 메시지를 받을 주소 : `/sub/chat/room/123`
### 동작 흐름

#### <구독 단계>

1. 클라이언트가 서버에 `"/topic/chat/room/123"` 와 같은 요청을 보내면 **메시지 브로커가** `enableSimpleBroker("/topic")`에 설정된 `/topic`으로 오는 _요청을 채팅방 ID 에 맞게 자동으로 구독해준다._

```java
@Controller  
@RequiredArgsConstructor  
public class ChatController {  
  
    private final ChatService chatService;  
  
    // = /app/sendMessage  
    // publish / send : 발행 역할  
    @MessageMapping("/sendMessage") // 서버에서 메시지 수신  
    public void sendMessage(Message message){  
        chatService.sendMessage(message);  
    }  
}

@Service  
@RequiredArgsConstructor  
public class ChatService {  
  
    private final MessageRepository messageRepository;  
    private final SimpMessagingTemplate messagingTemplate;  
  
    public void sendMessage(final Message message){  
        // 1. 메시지 객체 생성  
       Message messageToSave = Message.saveMessage(message);  
  
       // 2. 메시지 저장하고 클라이언트들에게 브로드캐스팅  
       messageRepository.save(messageToSave)  
               .doOnSuccess(savedMessage -> {  
                   // 브로드캐스팅 ( 클라이언트들이 메시지를 받기 위해 구독하는 주소 )                   String destination = "/topic/chat/room/" + savedMessage.getRoomId();  
                   messagingTemplate.convertAndSend(destination,savedMessage);  
               })  
               .subscribe();  
    }
```

#### <발행 단계>

2. 해당 채팅방에 구독하고 있는 사용자가 메시지를 보내면 `"/app/sendMessage"`가 호출되어 메시지를 발행한다.
3. `ChatService`는 메시지를 DB에 저장한 후 `messagingTemplate`을 이용해 **`/topic/chat/room/123`**으로 최종 메시지를 보낸다.
4. *메시지 브로커*는 해당 채팅방을 구독하고 있는 사용자들에게 메시지를 **브로드캐스팅해준다.**

