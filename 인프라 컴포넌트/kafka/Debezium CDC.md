#kafka 

![[cdc.png]]

## CDC(Change Data Capture) 란?

DB 에서 일어나는 실시간 변경사항(INSERT,UPDATE,DELETE)들을 감지하여 다른 시스템에 전달하는 기술이다.

## 동작 흐름

1. DB 에 `INSERT`
2. CDC 엔진이 binlog(트랜잭션 로그)를 감시하다가 감지하여
3. 메시징 큐(kafka)로 전달

## 장점

- Kafka 의 producer 를 따로 정의하지 않고 CDC 는 binlog 을 구독하여 CDC의 Debizium은 실시간으로 읽고 Kafka 로 전달한다.
- DB 에 어떠한 쿼리를 날리는 게 아니라 DB의 트랜잭션 로그만 읽어오기 때문에 DB 와 CDC 는 완전히 분리되어 있다.
- Debezium 은 트랜잭션 로그를 어디까지 읽었는지 offset 을 kafka 에 같이 보내기 때문에 중간에 데이터 유실이 있어도 이어서 작업할 수 있다.

## CDC KAFKA 세팅

CDC 는 외부 기술이므로 현재 사용하고 있는 docker 환경에서 새로운 컨테이너를 띄워서 실행한다.

### docker-compose

```yml
kafka-connect:  
  image: debezium/connect:2.6  
  container_name: kafka-connect  
  ports:  
    - "8083:8083"  
  environment:  
    BOOTSTRAP_SERVERS: "kafka:9092"  
    GROUP_ID: "connect-cluster"  
    CONFIG_STORAGE_TOPIC: "connect_configs"  
    OFFSET_STORAGE_TOPIC: "connect_offsets"  
    STATUS_STORAGE_TOPIC: "connect_status"  
    # (싱글 브로커이므로 리플리케이션 팩터는 1로 설정)  
    CONFIG_STORAGE_REPLICATION_FACTOR: "1"  
    OFFSET_STORAGE_REPLICATION_FACTOR: "1"  
    STATUS_STORAGE_REPLICATION_FACTOR: "1"  
    # JSON 컨버터 사용  
    KEY_CONVERTER: "org.apache.kafka.connect.json.JsonConverter"  
    VALUE_CONVERTER: "org.apache.kafka.connect.json.JsonConverter"  
    # JDBC 드라이버 (Debezium 이미지에 포함되어 있음)  
    CONNECT_PLUGIN_PATH: "/kafka/connect"  
  networks:  
    - monitoring  
  depends_on:  
    - kafka  
    - mysql
```

### register-debezium.json.template

```json
{  
  "name": "ticket-mysql-connector",  
  "config": {  
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    // DB 설정...
    "database.hostname": "ticket-mysql", // docker 내의 mysql 컨테이너 이름
    "database.port": "3306",  
    "database.user": "root",  
    "database.password": "wind6298",  
    "database.server.id": "184054",  
    "database.server.name": "ticket-mysql", 
    // 모니터링 대상 설정 
    "database.include.list": "ticket_on", // DB 스키마 이름
    "table.include.list": "ticket_on.outbox_message",  // 감시할 테이블
    "include.schema.changes": "false",  // ALTER 같은 DDL 명령어는 무시(false)
    // Debezium 동작
    "snapshot.mode": "initial", // 커넥터가 처음 시작될 때 모든 기존 데이터 읽어옴, 이후에 변경만 감지 
    "topic.prefix": "ticket",  
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",  
    "schema.history.internal.kafka.topic": "schema-changes.ticket_on",  
	// 아웃박스 패턴 변환
    "transforms": "outbox",  
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",  
    "transforms.outbox.table.field.event.id": "outbox_message_id",  // 테이블 칼럼을 Debezium 내부 필드에 매핑
    "transforms.outbox.table.field.event.key": "outbox_message_id",  
    "transforms.outbox.table.field.event.payload": "payload", // 아웃박스의 payload 컬럼을 내용물로 사용
    "transforms.outbox.table.expand.json.payload": "true",  
    "transforms.outbox.route.by.field": "topic", // 카프카의 topic 으로 전달 
    "transforms.outbox.route.topic.replacement": "ticket-confirm",  // topic 이름
    "transforms.outbox.route.key.field": "outbox_message_id",  
  
    "topic.creation.default.partitions": "3",  
    "topic.creation.default.replication.factor": "1",  
    "transaction.recovery.enabled": "true"  
  }  
}
```

`"transforms.outbox.table.expand.json.payload": "true"`
- String 인 payload 를 JSON 객체로 파싱
- 이 파싱된 객체를 EventRouter 가 만드는 payload 필드에 넣음

## 데이터는 어떻게 전달되는가?

1. **DTO -> JSON String 변환**
```java
String jsonPayload = objectMapper.writeValueAsString(event.message());  
OutboxMessage message = OutboxMessage.toEntityFromTicket(jsonPayload);
```
- event 로 감싸진 PaymentMessage 를 JSON String으로 변환
- JSON String로 변환된 payload 를 OutboxMessage 에 저장

형식 예시 )`"{\"ticketId\":123,\"userId\":\"abc\"}"`

2. **JSON String 파싱**
	1. Debezium 이 테이블 `INSERT` 를 감지하고
	2. 커넥터 설정에 따라 `transforms.outbox.expand.json.payload: "true"` JSON String 으로 파싱
	3. 파싱된 객체를 Debezium의 payload 라는 필드로 감싸서 Kafka 토픽으로 전달
	- 최종 발행 메시지 형태 : `"schema":{...}, "payload":{"ticketId":123,...}}`

3. **Consumer -> String deserializer**
	1. Consumer 는 StringDeserializer 를 사용하여 Debezium 이 발행한 JSON 메시지 전체를 String 타입으로 받는다.

4. **String -> DTO 파싱**

	```java
	public void consumePayment(String message, Acknowledgment ack) {  
    try {  
        // Debezium 이 보낸 전체 JSON 문자열을 JsonNode 로 파싱  
        JsonNode rootNode = objectMapper.readTree(message);  
        // payload 에 있는 실제 메시지(PaymentMessage) 추출  
        JsonNode payload = rootNode.get("payload");  
        PaymentMessage payment = objectMapper.treeToValue(payload, PaymentMessage.class);  
        ticketService.issueTicket(payment);  
        ack.acknowledge();
	```
	- 문자열을 `JsonNode` 로 파싱하여 .get("payload") 로 원하는 PaymentMessage Dto 의 필드값들이 담겨져 있는 payload 부분만 가져온다.
	- `objectMapper` 를 이용하여 DTO 객체로 변환한다.
- `JsonNode` : JSON 데이터를 담는 객체

### 왜 JsonDeserializer(PaymentMessage) 이 아닌 StringDeserializer 으로 받을까?

**이유** : Debezium 이 메시지를 `{"payload": ...}` 구조로 한번 감싸서 보내기 때문

만약 JsonDeserializer 로 받게 되면 DTO 필드값에는 없는 `scheme, payload` 이 있어서 변환에 실패하여 모든 객체가 NULL 인 상태로 받게 된다.

Debezium CDC의 표준 출력을 처리하는 방식으로는 일단 String 으로 다 받고, Consumer 가 직접 payload 를 파싱해서 써야 한다.