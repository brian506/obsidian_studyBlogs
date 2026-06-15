#kafka #spring #project  

# 📌 Producer

### 메시지 흐름

 Kafka 를 **사용자와 pg 사 호출 사이**에 두려고 했다. 
 예약 정보를 producer 가 받아서 Kafka 내에서 분산,병렬 처리를 통해 PG 사 호출 전 병목 현상을 완화시키려고 했다.
 
하지만 토스는 결제 API 를 프론트단에서 결제 위젯을 이용해 결제 서버를 호출하기 때문에 프론트와 통신을 지원하지 않는 Kafka 를 이용해서 위와 같은 구조를 구현할 수 없었다.

PG 사 결제 승인 요청까지 마무리 된 paymentResponse 값을 PaymentMessage 로 변환 후 Producer 가 전달할 것이다.
PaymentMessage dto 안에는 예약정보까지 포함되어있다.

**메시지들을 배치 형태로 정해진 크기를 Buffer 에 모아서 한번에 전송해줄 것이다.**

1. send() 메서드로 메시지들이 Producer 내부에 있는 __Buffer__ 에 쌓임
2. Producer 에서 메시지 __Batch 형태__ 로 만들어서 해당 Broker 의 **partition**으로 전송(partition : Batch)

___
###   application-local.yml 파일

```yml
kafka:  
  admin:  
    properties:  
      bootstrap.servers: localhost:9092  
  topic-config:  
    payment:  
      name: payment-topic  
      partitions: 3  
      replicas: 1
      
  producer:  
  bootstrap-servers: localhost:9092  
  key-serializer: org.apache.kafka.common.serialization.StringSerializer  
  value-serializer: org.springframework.kafka.support.serializer.JsonSerializer  
  retry: 2
  
```

- local 환경이기 때문에 bootstrap-servers 주소를 localhost:9092 로 설정
### application-docker.yml 파일

```yml
kafka:  
  admin:  
    properties:  
      bootstrap.servers: kafka:9092  
  topic-config:  
    payment:  
      name: payment-topic  
      partitions: 3  
      replicas: 1
      
producer:  
  bootstrap-servers: kafka:9092  
  key-serializer: org.apache.kafka.common.serialization.StringSerializer  
  value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
  properties:
	  max.block.ms: 5000 # 버퍼가 가득 찼을 때 대기 시간(5초)
	  linger.mns: 20 # 추가 메시지 기다리는 최대시간(20ms)
  buffer-memory: 67108864 # 버퍼 크기(67MB)  
  retry: 2
```

- docker 환경에서 kafka 를 띄우기 때문에 kafka:9092 로 설정
- producer 에서는 Batch 메시지 크기를 **용량**으로 정해서 보냄

___

### ProducerConfig

예약과 결제 공통으로 사용되는 ProducerConfig 이다.

```java
@Configuration
public class KafkaProducerConfig{
	@Value("${kafka.producer.bootstrap-servers}")

	@Bean
	public Map<String,Object> producerConfigs() {
	Map<String,Object> props = new HashMap<>();
	props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,bootstrapServers);
	props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,StringSerializer.class);
	props.put(ProducerConfig.VALUE_SERIALZER_CLASS_CONFIG,JsonSerializer.class);
	props.put(JsonSerializer.ADD_TYPE_INFO_HEADERS,true); // 역직렬화 충돌방지
	return props;
	}

	// producer 인스턴스 생성 factory 객체
	@Bean
	public ProducerFactory<String,Object> producerFactory(){
		return new DefaultKafkaProducerFactory<>(producerConfigs());
		}

	// kafka 메시지 전송 담당 객체
	@Bean
	public KafkaTemplate<String,Object> kafkaTemplate(){
		return new KafkaTemplate<>(producerFactory());
		}
	
}
```

- Producer 에서 Json 형식의 PaymentMessage 객체를 받을 것이므로 JsonSerializer.class 를 사용한다.

### PaymentProducer

```java
@Service
@RequiredArgsConstructor
public class PaymentProducer {
	
	@Value("${kafka.topic-config.payment.name}")
	private String topic;

	private final KafkaTemplate<String,Object> kafkaTemplate;

	public void sendPayment(final PaymentConfirmResponse response,final PaymentConfirmRequest request) {
		PaymentMessage message = response.fromResponse(response,request);
		KafkaTemplate.send(topic,String.valueOf(message.getTicketTypeId()),message);
		}
	}


// PaymentService 클래스
// 결제 승인 요청  
public void confirmPayment(PaymentConfirmRequest paymentConfirmRequest) {  
    PaymentConfirmResponse paymentConfirmResponse = paymentGateway.requestPaymentConfirm(paymentConfirmRequest);  
    paymentProducer.sendPayment(paymentConfirmResponse);  
}
```

- 결제 정보만 담겨 있는 PaymentConfirmResponse 를 받아서 PaymentMessage 로 변환해준다.
- 이 메시지를 payment-topic 으로 보내준다.
- **ticketTypeId 를 기반으로 key 값을 정해서 라운드-로빈 방식이 아닌 key 값을 기준으로 전달한다.**
- 같은 ticketTypeId 는 무조건 같은 파티션으로 할당된다.(Id 값이 달라도 같은 파티션에 할당될 수 있음)


- paymentService 에서 send() 메서드로 토스 **서버에서 받은 응답을 producer 로 전송**한다.



### Consumer Group 특징

- **Consumer Group**: 논리적인 개념
- **확장성**: 메시지를 여러 서비스로 처리 가능
- **병렬 처리**: 여러 Consumer가 동시에 메시지 처리












