#spring #exception #error-handling 

# 개요

스프링에서 발생하는 모든 API 예외 처리를 효율적으로 하기 위해 ExceptionResolver 클래스를 이용할 것이다.

목표 :
- ExceptionBase 를 상속한 모든 예외 감지
- 예외 발생 시, 클라이언트에게 일관된 JSON 응답 반환
- 에러 로그 출력

### ✅ ExceptionResolver

```java
@ControllerAdvice  
public class ExceptionResolver {  
  
    private static final Logger logger = LogManager.getLogger(ExceptionResolver.class);  
  
    @ExceptionHandler({ExceptionBase.class}) 
    @ResponseBody  
    public ResponseEntity<SuccessResponse> handleClientException(HttpServletRequest request, Exception exception) {  
        ExceptionBase ex = (ExceptionBase) exception;  
        logException(request, ex);  
        SuccessResponse response = new SuccessResponse(false, ex.getErrorMessage(), ex.getErrorCode().getCode());  
        return ResponseEntity  
                .status(ex.getHttpStatus())  
                .body(response);  
    }  
 
```


```json
{
  "success": false,
  "message": "[VIP(10000)] 발급된 티켓이 없습니다.",
  "code": 4002
}

```


1. WAS 에서 받은 요청을 통해서 예외를 ExceptionBase 로 캐스팅한다.
2. ExceptionBase.Class 하위 클래스의 예외들을 호출한다.
3. 호출한 응답값을 @ResponseBody 를 통해서 JSON 형태로 반환한다.


```java
private void logException(HttpServletRequest request, ExceptionBase exception) {  
    String requestBody = extractRequestBody(request);  
    logger.error("Request URL: {}, Method: {}, Exception: {}, Request Body: {}",  
            request.getRequestURL(),  
            request.getMethod(),  
            exception.getErrorMessage(),  
            requestBody);  
}  
  
private String extractRequestBody(HttpServletRequest request) {  
    if (request instanceof ContentCachingRequestWrapper wrapper) {  
        byte[] content = wrapper.getContentAsByteArray();  
        if (content.length > 0) {  
            return new String(content, StandardCharsets.UTF_8);  
        }  
    }  
    return "N/A";  
}
```

- 에러를 log 에 나타내기 위해 request 의 url, method(POST,GET 등), errorMessage, 본문을 추출해 log 에 남긴다.

___

#### 어노테이션 설명
### @ControllerAdvice
[[커스텀 예외 처리(Exception)]]
- 전역예외 처리를 할 수 있도록 하는 어노테이션이다.

### @ExceptionHandler(XXX.class)
[[커스텀 예외 처리(Exception)]]
- XXX.class 안에 있는 메서드들을 호출해 예외를 처리한다.
