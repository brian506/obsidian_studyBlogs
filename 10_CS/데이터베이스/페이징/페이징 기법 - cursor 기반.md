#database #mongoTemplate 
# 개요
offset 기반과 다르게 cursor 는 마지막으로 본 항목의 고유한 ID 값으로 책갈피와 비슷한 역할을 하는 cursor 를 이용해서 많은 데이터를 조회할 수 있다.

쉽게 말해 갯수가 아니라 위치 기반 페이징 기법이다.

커서가 되기 위한 조건으로는
- 고유한(Unique) 값
- 정렬 가능해야 하는 값

## 구현

채팅방에서 새로고침해도 대화 내용이 사라지지 않게 이전 대화 내용의 메시지들을 무한 스크롤이 가능한 커서 기반으로 조회하는 예제이다.
메시지 DB 는 mongoDB 를 사용하므로 동적 쿼리를 위해 MongoTemplate 을 이용해서 구현해볼 것이다.

### Repository
```java
@RequiredArgsConstructor  
public class MessageRepositoryCustomImpl implements MessageRepositoryCustom{  
  
    private final MongoTemplate mongoTemplate;
    
    @Override
    public List<Message> getMessagesFromRoomId(String roomId,String cursor){
	    Query query = new Query();
	    // 고정 조건 room_id = roomId 일 때
	    query.addCriteria(Criteria.where("room_id").is(roomId));
	    // 동적 조건 _id = message 엔티티의 @Id
	    if(cursor != null && !cursor.isEmpty()){
		    query.addCriteria(Criteria.where("_id").lt(new ObjectId(cursor)));
	    }
	    // 가장 최신 데이터를 10씩 조회
	    query.with(Sort.by(Sort.Direction.DESC,"_id"))
			 .limit(10);
	    return mongoTemplate.find(query,Message.class);
    }
```
- message 의 "`_id`" 값을 cursor 로 설정
- ObjectId 는 값 자체가 **Unique 하고 타임스탬프가 포함**되어있어 cursor 로 적합
- 사용자가 위로 올리면서 조회를 할 때 가장 최신 데이터가 나와야하므로 내림차순 정렬로 최신 데이터를 가져옴

### Service
```java
public MessageListResponse getMessagesFromRoom(final String roomId, final String cursor){
	List<Message> messages = messageRepository.getMessagesFromRoomId(roomId,cursor);
	String nextCursor = messages.get(messages.size() - 1).getId();
	
	Collections.reverse(messages); 
	
	List<MessageResponse> messageResponses = messages.stream()  
        .map(Message::toDtoFromRoom)  
        .toList();  
	return new MessageListResponse(messageResponses,nextCursor);
}
```
- 현재 Message 리스트 값이 최신~나중 순으로 정렬되어 있음. ex) 10 9 8 7 ... 1
- 다음 커서는 가장 나중 값으로 설정해야 하기 때문에 `message.size()-1` 로 배열의 끝 인덱스로 설정해야 위로 스크롤해서 다음 데이터를 볼 때 책갈피 역할을 할 수 있음 ex) cursor : 1
- `Collections.reverse(messages)` : 사용자가 볼때는 최신 메시지가 아래부터 보여야 하기 때문에 원래 정렬된 데이터를 거꾸로 뒤집는다. - 최신순이므로 늦게 생성된 순으로 데이터가 정렬되어있기 때문
-> 이렇게 되면 대화방 위쪽이 옛날 메시지 ~ 아래쪽이 최근 메시지
- 다음 커서를 받기 위해 새로운 DTO 객체 MessageListResponse 생성

### Controller
```java
@GetMapping("/{roomId}")  
@PreAuthorize("isAuthenticated()")  
public ResponseEntity<?> getMessagesFromRoom(@PathVariable String roomId,  
                                             @RequestParam(required = false) String cursor){  
    MessageListResponse messageResponses = messageService.getMessagesFromRoomCursor(roomId,cursor);  
    SuccessResponse response = new SuccessResponse(true,"채팅방 메시지 불러오기 성공",messageResponses);  
    return new ResponseEntity<>(response,HttpStatus.OK);  
}
```
- 프론트에 cursor 값을 넘겨준다.