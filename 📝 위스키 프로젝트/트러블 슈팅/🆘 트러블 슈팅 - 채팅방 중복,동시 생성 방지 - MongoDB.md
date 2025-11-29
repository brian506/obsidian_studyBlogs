#NoSQL #트러블슈팅 #mongoTemplate 

# 개요
질문자 기준에서 상대방을 조회해서 상대방의 `userId`를 가져온 값으로 `RequestBody`에 targetId(상대방Id)로 채팅방을 생성한다.

영민과 철수가 채팅을 한다고 가정했을 때,
**영민이 질문자일 때**
영민(질문자) <=> 철수(답변자)의 역할을 가진 채팅방이 영민과 철수에게 채팅방이 하나씩 생성되고,
**철수가 질문자일 때**
철수(질문자) <=> 영민(답변자)의 역할을 가진 채팅방도 하나씩 생성될 수 있다.
왜냐하면 익명성 보장을 위해서 **질문자일 때는 상대방의 실명**이 보이고, **답변자일 때는 상대방의 닉네임**이 보이기 때문이다.

채팅방의 중복,동시 생성을 방지하기 위해 *unique 인덱스*를 사용한다.
똑같은 문자열(roomKey)이 발생하는 것을 방지하기 위해 해당 필드값에 `UNIQUE` 특성을 걸어 DB 단에서 중복을 방지한다.
## 구현

### Room
```java
@Indexed(unique = true)  
@Field(name = "room_key")  
private String roomKey; // 중복 생성 방지

public static Room ofPrivateRoom(Participant asker, Participant answerer) {  
    return Room.builder()  
            .roomType(RoomType.PRIVATE)  
            .participants(List.of(asker, answerer))  
            .roomKey(directionalKey(asker.userId(),answerer.userId()))  
            .createdAt(LocalDateTime.now())  
            .build();  
}

// 질문자 -> 답변자 순서 고정 생성  
public static String directionalKey(String askerId,String answererId) {  
    if (answererId == null || askerId == null) throw new DataNotFoundException("질문자 혹은 답변자가 존재하지 않습니다.");  
    return askerId + "->" + answererId; // 방향성 유지!  
}
```
- 채팅방이 생성될 때 `{askerId} -> {answererId}` 가 `roomKey`에 저장된다.
- 방향성 있는 고유한 문자열이 UNIQUE 한 특성을 지닌 상태를 갖게 된다.
- 같은 두 사용자라도 사용자의 역할(질문자 or 답변자)에 따라 채팅방이 4개까지 만들어진다.

### RoomService
```java

// 채팅방 최종 생성
public RoomUserResponse createPrivateRoom(String myUserId, String targetUserId) {  
    Room room = createOrGetRoom(myUserId, targetUserId);  
    String view = viewByRole(myUserId, room);  
  
    // 채팅방 발행  
    CreateRoomEvent roomEvent = CreateRoomEvent.of(room.getId(),myUserId,targetUserId);  
    publishService.publishRoomCreated(roomEvent);  
    return RoomUserResponse.of(room.getId(), view);  

// ㅊ
public Room createOrGetRoom(String askerId, String answererId) {  
    String roomKey = Room.directionalKey(askerId,answererId);  
  
    return roomRepository.findByRoomKey(roomKey).orElseGet(() -> {  
        ChatUser askerUser   = OptionalUtil.getOrElseThrow(userRepository.findById(askerId),    "존재하지 않는 사용자입니다.");  
        ChatUser answererUser = OptionalUtil.getOrElseThrow(userRepository.findById(answererId),"존재하지 않는 사용자입니다.");  
        // dto 변환  
        Participant asker    = Participant.asker(askerUser.getUserId(),   askerUser.getNickname(),   askerUser.getUsername());  
        Participant answerer = Participant.answerer(answererUser.getUserId(), answererUser.getNickname(), answererUser.getUsername());  
  
        Room newRoom = Room.ofPrivateRoom(asker,answerer);  
        return roomRepository.save(newRoom);  
    });  
}  
  
// 요청자 관점에서 상대 표시 (ASKER=실명, ANSWERER=닉네임)  
// 채팅방 안에서 참여자들 중에 내가 질문자인지, 답변자인지에 따른 Participant 객체 반환  
private String viewByRole(String requesterUserId, Room room) {  
  
    Participant askerUser = room.getParticipants().stream()  
            .filter(p -> p.userId().equals(requesterUserId))  
            .findFirst().orElseThrow(() -> new DataNotFoundException("채팅방 참여자 정보가 없습니다."));  
  
    Participant answererUser = room.getParticipants().stream()  
            .filter(p -> !p.userId().equals(requesterUserId))  
            .findFirst().orElseThrow(() -> new DataNotFoundException("상대방 정보가 없습니다."));  
  
    if (askerUser.userType() == UserType.ASKER) {  
        return answererUser.username();  
    }  
    return answererUser.nickname();  
}    
}
```

> `createOrGetRoom()`
1. `askerId`와 `answererId`를 가지고 먼저 `roomKey`를 생성한다.
2. `roomKey` 가 Repository 에 없으면 Room 에 있는 채팅방 참여자 객체인 `ParticipantDTO`로 변환해서 채팅방을 저장한다.
> `viewByRole()`
3. 질문자 입장에서는 상대방의 이름이, 답변자 입장에서는 상대방의 별명이 보여야 하므로 역할을 구분한 상태로 User 객체를 반환한다.
4. 채팅방에 사용자가 두명인 것을 전제로 p(= 나), !p(= 상대방) 객체로 할당하여 내가 질문자일 때는 상대방의 실명, 내가 답변자일 때는 상대방의 별명을 반환한다.
> `createPrivateRoom()`
5. 앞에서의 메서드로 채팅방을 생성하고
6. 두 사용자에게 채팅방 생성의 실시간성을 보장하기 위해 `PUBLISH`한다.


