#logging 

스프링 부트 애플리케이션의 요청 처리는 크게 두 가지 영역으로 나뉜다.

**서블릿 필터 영역(Web Context)** : DispatcherServlet 에 들어가기 전인 *보안 인증* 단계
**스프링 MVC 영역(Spring Context)** : 비즈니스 로직이 수행되는 컨트롤러/서비스 단계

![](../../images/스크린샷%202026-01-08%2011.05.52.png)

HTTP 요청 -> WAS -> Servlet Filter -> Dispatcher Servlet -> Controller 순으로 거쳐간다.

## Servlet Filter

`chain.doFilter()` 로 토큰에 대한 인증 및 인가에 대한 검증을 한다.
`JwtAuthenticationFilter` 가 수행하는 인증 및 인가 작업을 이 필터에서 수행한다.
모든 인증에 관한 요청, 요청에 대한 예외처리는 `@ControllerAdvice` 가 아닌 이 Filter 에서 처리한다.

인증 요청에 대한 에러 로그를 수집하는 클래스를 한번 살펴보자

#### 1. 사용자가 로그인 시도 및 인증 요청에 대한 예외 처리

```java
public class AuthenticationExceptionHandler {  
  
  // IP추적을 위해 사용자가 접근할 수 있는 모든 경로를 다 탐색한다
    private static final String[] HEADERS = {  
            "X-Forwarded-For",  
            "X-Real-IP",  
            "Proxy-Client-IP",  
            "WL-Proxy-Client-IP",  
            "HTTP_CLIENT_IP",  
            "HTTP_X_FORWARDED_FOR"  
    };  
  
    public void handle(HttpServletRequest request, HttpServletResponse response, AppException exception)  
            throws IOException {  
       // JwtAuthenticationFilter 에서 발생한 에러 처리
    }  
  
    public void handle(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception)  
            throws IOException {  
       // JwtAuthenticationEntryPoint 에서 발생한 에러 처리
    }  
  
    private void writeUnauthorizedResponse(HttpServletRequest request, HttpServletResponse response,  
                                           ApiResponse<Void> body)  
            throws IOException {  
       
       // clientIp : 누가?, method & uri : 무엇을?, userAgent : 어떤 도구로? 
       // 침입하려고 했는지 알기 위해 request 에서 추출하여 로그로 찍어낸다.
        log.warn("[Authentication Exception]: IP={} | Method={} | URI={} | UserAgent={} | errorCode={} | message={}",  
                clientIp, method, uri, userAgent, errorCode, message);  
    }  
}
```

위 클래스는 Filter 영역에서 요청에 대한 인증(토큰 검증 및 유효성)이 실패하였을 때(예외가 터졌을 때) 사용자가 접근한 경로 등 정보를 추출해서 로그로 남기기 위한 클래스이다.

`JwtAuthenticationFilter`: 토큰을 추출해서 유효성을 검증하고, 사용자를 Authentication 에 저장
`JwtAuthenticationEntryPoint` : 비어있는 토큰이 넘어왔을 때 예외처리하는 클래스


## DispatcherServlet

비즈니스 로직에 대한 요청은 앞서 있는 Filter 가 아닌 Dispatcher Servlet 으로 가장 먼저 도달한다.
예외 발생 시 `ControllerAdvice` 를 호출하여 예외를 터뜨린다.

#### 2. 스프링 내에서 발생한 예외처리
```java 
@RestControllerAdvice  
public class ApiControllerAdvice {  
  
    @NullMarked  
    @ExceptionHandler(AppException.class)  
    public ResponseEntity<ApiResponse<@Nullable Object>> handleAppException(AppException e) {  
        printAppExceptionLog(e);  
  // 에러 타입, 데이터, 상태 포멧 응답 처리 
    }  
  
    @NullMarked  
    @ExceptionHandler(Exception.class)  
    public ResponseEntity<ApiResponse<@Nullable Object>> handleException(Exception e) {  
        printExceptionLog(e);  
 
    }  
  
    private void printAppExceptionLog(AppException e) {  
       // 정확한 에러 경로에 대해서 모두 추출하고, 에러 원인에 대해서 3단계로 세분화해서 로그 저장
  
        switch (e.getErrorType().getLogLevel()) {  
            case ERROR ->  
                    log.error("[AppException]: class={} | method={} | line={} | status={} | errorCode={} | message={} | data={}",  
                            className, methodName, lineNumber, status, errorCode, message, data);  
            case WARN ->  
                    log.warn("[AppException]: class={} | method={} | line={} | status={} | errorCode={} | message={} | data={}",  
                            className, methodName, lineNumber, status, errorCode, message, data);  
            default ->  
                    log.info("[AppException]: class={} | method={} | line={} | status={} | errorCode={} | message={} | data={}",  
                            className, methodName, lineNumber, status, errorCode, message, data);  
        }  
    }  
  
    private void printExceptionLog(Exception e) {  
       // 정확한 에러 경로 추출 후 응답
    }  
}
```

1. **handleAppException** : 예상한 에러에 대한 비즈니스 예외를 처리한다.
2. **handleException** : 예상하지 못한 시스템 버그에 대한 에러를 처리한다.


#### 3. 비즈니스 로직에 대한 로그(AOP 활용)

```java
@Aspect  
@Component  
public class SignAspect {  
  
    @Pointcut("@within(org.springframework.web.bind.annotation.RestController) || " +  
            "@within(org.springframework.stereotype.Controller)")  
    private void controllers() {  
    }  
  
    @Around("controllers()")  
    public Object apiExchange(ProceedingJoinPoint joinPoint) throws Throwable {  
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();  
  
        if (attributes == null) {  
            return joinPoint.proceed();  
        }  
  
        HttpServletRequest request = attributes.getRequest();  
        String clientIp = NetworkUtils.getClientIp(request);  
        String method = request.getMethod();  
        String uri = request.getRequestURI();  
        String userAgent = request.getHeader("User-Agent");  
        log.info("[REQUEST API]: IP={} | Method={} | URI={} | UserAgent={}", clientIp, method, uri, userAgent);  
  
        Object result = joinPoint.proceed();  
  
        HttpServletResponse response = attributes.getResponse();  
        int status = resolveStatus(response, result);  
        log.info("[RESPONSE API]: status={}", status);  
  
        return result;  
    }  
  
    private int resolveStatus(HttpServletResponse resp, Object result) {  
        if (result instanceof ResponseEntity<?> re) {  
            return re.getStatusCode().value();  
        }  
  
        if (resp != null) {  
            return resp.getStatus();  
        }  
  
        return 200;  
    }  
  
    @Around("@annotation(com.foodkeeper.foodkeeperserver.common.aspect.annotation.SignInLog)")  
    public Object signInLog(ProceedingJoinPoint joinPoint) throws Throwable {  
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();  
  
        if (attributes != null) {  
            HttpServletRequest request = attributes.getRequest();  
            String clientIp = NetworkUtils.getClientIp(request);  
            String method = request.getMethod();  
            String uri = request.getRequestURI();  
            String userAgent = request.getHeader("User-Agent");  
            log.info("[SIGN IN]: IP={} | Method={} | URI={} | UserAgent={}", clientIp, method, uri, userAgent);  
        }  
  
        return joinPoint.proceed();  
    }  
    
    
    // SignInLog 어노테이션
    // 로그인 컨트롤러 메서드 위에 달려있음
@Target(ElementType.METHOD)  
@Retention(RetentionPolicy.RUNTIME)  
public @interface SignInLog {  
}
}
```

`@Aspect` : AOP 의 감시자 역할을 하는 클래스임을 지정
`@Pointcut`: 모든 컨트롤러 클래스의 요청, 응답에 대해서 감시하는 목표를 지정
`JoinPoint` : 하나의 요청에 대하여 AOP 가 가로챈 메서드 지점
`@Around` : () 안에 있는 객체를 감싸고 있는 역할. 
	- 요청이 들어오면 () 안에 있는 객체보다 먼저 만나고, 응답이 나갈때도 마지막에 배웅해준다.

`RequestContextHolder` 라는 전역 보관함에 `ServletRequestAttributes` 라는 포장지를 뜯어서 `HttpServletRequest` 객체를 꺼내서 해당 요청에 대한 정보를 추출한다.
	- AOP 라는 제3의 공간에서는 직접 `HttpServletRequest` 를 받지 못하므로 

### `apiExchange()`동작 순서

1. 사용자의 요청이 들어온다. 아직 실제 컨트롤러에는 도달하지 못했다.
2. 컨트롤러 메서드 진입 전에 해당 `HttpServletRequest` 를 추출하여 로그로 남긴다.
3. `joinPoint.proceed()` 가 호출되면 해당 컨트롤러의 메서드가 실행되어 `ResponseEntity`를 리턴하여 `Object` 로 감싸서 `result` 에 담는다.
4. 컨트롤러는 일이 끝났고, 응답 받은 `result` 에 담긴 상태 코드를 확인해서 반환한다.

### `signInLog` 동작 순서

1. 런타임시에만 동작하는 `SignInLog` 어노테이션이 로그인 메서드에 달려있다.
2. 로그인 성공 로그는 별도로 처리해야 하므로 `SignInLog` 어노테이션이 달린 메서드를 대상으로 요청을 가로채서 정보를 추출하고 로그인 시에 들어온 정보를 로그에 저장한다.