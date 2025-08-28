#spring개념 

#  JSON 이란?

데이터를 구조화해서 문자열로 표현하는 방식
프론트엔드와 백엔드 사이, 혹은 서버와 서버 사이에서 데이터를 주고받을 때 주로 사용

### 직렬화와 역직렬화

**직렬화**  : 자바 객체 -> JSON 문자열로 변환

> 내 서버에서 클라이언트에게 응답값을 반환할 때 보내는 방식

**역직렬화** : JSON 문자열 -> 자바 객체 

> 클라이언트가 서버로 요청했을 때 서버에서 반응하는 방식

___

## 🔎 Jackson 의 주요 어노테이션
#### @JsonProperty("")

만약 토스 서버에서 보내는 JSON 필드명 형태가 secret-key 라고 하자,
그렇지만 나는 필드명을 secretKey 로 정하면 두 필드명이 매핑되지 않는 상황이 발생하게 된다.
그럴 때 @JsonProperty 괄호 안에 JSON 필드명으로 매핑해준다.

```java
@JsonProperty("secret-key")
private String secretKey;
```

#### @JsonIgonreProperties(ignoreUnknown = true)

외부 api에서 특정 기능을 수행했을 때 응답하는 값들이 정해져 있다.
이 응답값들 중 내가 원하는 몇개의 응답값만 DB 에 저장하고 싶을 때 나머지 내 서버에 존재하지 않는 JSON 필드는 무시하고 역직렬화하도록 해주는 어노테이션이다.

```java
@JsonIgnoreProperties(ignoreUnkown = true)
public class User {
	private String name;
    }
````

___

## 🔎 ObjectMapper

> 역직렬화와 직렬화를 하는 Jackson 라이브러리이다.

평소에 @RequestBody 혹은 dto 를 쓸 때는 자동으로 JSON 을 파싱해주지만 HttpClient 를 직접 다루게 될 때는 ObjectMapper 를 써야한다.

```java
private PaymentResponseErrorCode getPaymentConfirmErrorCode(final ClientHttpResponse response) throws IOException {
        PaymentFailResponse confirmFailResponse = objectMapper.readValue(
                response.getBody(),PaymentFailResponse.class);
        return PaymentResponseErrorCode.findByCode(confirmFailResponse.getCode());
    }
````

위 코드는 토스에서 받은 결제 실패 응답 JSON 을 PaymentFailResponse 객체로 직접 파싱한 것이다. 
응답 본문을 가져와서 PaymentFailResponse 클래스의 객체로 만들어주기 위함이다. 