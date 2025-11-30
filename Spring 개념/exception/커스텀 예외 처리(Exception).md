#spring #exception #backend 

## 개요

스프링에서 주어지는 에러 응답을 보고 서버에서나 프론트쪽에서 에러를 정확하게 식별하기 어렵다. 
그래서 Exception 을 직접 커스텀해서 어떤 계층 또는 메서드에서 Exception 이 발생했는 지 정확하게 알기 위해서 예외처리를 커스텀해서 구현하면 유지보수 또는 문제가 있는 곳을 찾기 더 쉬울 것이다.
___

## 순서

예외가 발생하는 순서를 먼저 파악해보자.

![[../../images/exec.jpeg]]

티켓팅 프로젝트에서 **서버 내에서 발생하는 에러(TicketException)** 와  **PG 사 서버(PaymentException)에서** 발생하는 에러를 로컬 환경에서 로그로 볼 수 있는 커스텀 에러를 작성해보자.

TicketException 예시로 InvalidTicketCancelException, NotFoundDataException 등 상황에 맞는 커스텀 예외를 설정해놓은 상태이다.

🔥 문제 발생
1. TicketException 또는 PaymentException 예외 호출
2. 해당 예외는 ExceptionBase 를 상속중
3. ExceptionResolver 에서 ExceptionBase 탐지
4. ErrorCode 에서 예외에 맞는 상태코드, 메시지 꺼내서 응답

# 구성 클래스

### ✅ [[ExceptionResolver]]

### ✅  ExceptionBase

```java
@Getter  
public abstract class ExceptionBase extends RuntimeException {  
  
    protected String errorMessage;  
    protected ErrorCode errorCode;  
    public abstract HttpStatus getHttpStatus(); // 구현 클래스에 위임  
}
```

ExceptionBase 를 공통된 응답값을 정의하고 하위 클래스가 세부 구현을 다르게 하도록 추상클래스로 만든다.

___

### ✅  TicketException / PaymentException (커스텀 예외 처리) 

#### Ex) TicketAlreadyCancelledException

```java
public class TicketAlreadyCancelledException extends ExceptionBase {  
  
    public TicketAlreadyCancelledException() {  
        this.errorCode = ErrorResponseCode.TICKET_ALREADY_CANCELLED;  
        this.errorMessage = "이미 취소된 티켓입니다.";  
    }  
  
    @Override  
    public HttpStatus getHttpStatus() {  
        return HttpStatus.CONFLICT;  
    }  
}

 // 티켓 중복 건에 대한 예외발생 메서드
 private void validateCancelable() {  
    if (this.ticketStatus.equals(TicketStatus.CANCELLED)) {  
        throw new TicketAlreadyCancelledException();  
    }  
}

```

에러 응답 시 errorCode, errorMessage, httpStatus 를 각 Exception 클래스에서 값을 정한다.

#### Ex) PaymentConfirmException

```java
@Getter
@AllArgsConstructor
public class PaymentConfirmException extends RuntimeException {  
	private HttpStatusCode code;
	private String errorMessage;  
}
```

PaymentResponseErrorCode 안에 여러 예외 상수들을 모아놓았다. 
위 클래스 안에서 해당 code 를 찾아서 에러 응답값을 반환한다.

___

### ✅ ErrorResponseCode 

이제 해당 에러가 터졌을 때 어떤 코드 번호를 응답할지 담아 놓은 enum 클래스를 구현해보자.

#### ErrorResponseCode

```java
@Getter  
@RequiredArgsConstructor  
public enum ErrorResponseCode implements ErrorCode{  
    EXCEEDED_TICKET_QUANTITY(4001),  
    INVALID_TICKET_CANCELLATION(4002),  
    NOT_VALID_TOKEN(4031),  
    DATA_NOT_FOUND(4041),  
    TICKET_ALREADY_CANCELLED(4091);  
      
    private final int code;  
}
```

위 클래스는 내 서버에서 발생하는 티켓 예약 에러에 대한 code 값을 쉽게 식별하기 위해 만든 커스텀 enum 클래스이다.

상수값을 이용해서 해당 로직에 대한 예외가 터졌을 때, 특정 코드값을 반환하기 위함이다.

___
### ✅ Toss 호출 WebClient

```java
@Override  
public PaymentConfirmResponse requestPaymentConfirm(PaymentConfirmRequest request) {  
    TossConfirmRequest tossRequest = request.toTossConfirmRequest();  
    return restClient.method(HttpMethod.POST)  
            .uri(paymentProperties.getConfirmUrl())  
            .contentType(MediaType.APPLICATION_JSON)  
            .body(tossRequest)  
            .retrieve()  
            .onStatus(HttpStatusCode::isError, (req, res) -> {
	            String errorBody = getErrorBody(res);
        System.out.println("Toss error response: " + errorBody);
        throw new PaymentConfirmException(res.getStatusCode(), errorBody);
                })
                .body(PaymentConfirmResponse.class);
    }

public String getErrorBody(ClientHttpRepsonse res){
	return String errorBody 
	= new BufferedReader(new InputStreamReader(res.getBody()))
							        .lines().collect(Collectors.joining("\n"));
}


```

- 토스 서버에서 응답해주는 에러 코드를 로그로 반환받을 수 있다.
- 정확한 에러 정보를 식별하기 위해서는 토스 개발자 센터에서도 확인할 수 있다.