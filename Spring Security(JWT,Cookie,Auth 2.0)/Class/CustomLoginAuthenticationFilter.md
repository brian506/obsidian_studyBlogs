#backend #security #JWT #spring #class

___
## 주요 개념

사용자가 로그인 요청을 하면 먼저 제일 처음에 [[CustomLoginAuthenticationFilter]] 에서 인증을 가로챈다.

**✅ 로그인 요청은 Controller 에서 처리하지 않고 Filter 에서 처리한다.**
#### 이유 : 
- 관심사의 분리(인증/인가, 비즈니스 로직 분리)
- Filter Chain 기반 동작
- 보안 강화
##### Filter Chain 이란? 
-> 웹 요청이 최종 목적지(컨트롤러)에 도달하기 전에 거쳐야하는 연속된 줄(Chain)

___

## 고려해야 할 점

1. 사용자 확인 (attemptAuthentication)
2. `UsernamePasswordAuthenticationToken` 인증 객체 반환
3. `SecurityConfig` 에서 Manager 가 Provider 에게 비밀번호 비교 로직 위임
4. Manager 에게 인증 성공 여부 데이터를 받은 Filter 가 핸들러 호출
5. 로그인 시에만 **서버는 body 에 AccessToken 을 담아서 클라이언트에게 보낸다.**

___

### 구조
![[image/스크린샷 2025-08-26 오후 4.53.37.png]]

### 코드 구현

#### 사용자 확인 및 인증 객체 반환

```java
  
public class CustomLoginAuthenticationFilter extends AbstractAuthenticationProcessingFilter {  
  
    private static final String DEFAULT_LOGIN_REQUEST_URL = "/v1/api/admin/login";  
    private static final String HTTP_METHOD = "POST";  
    private static final String CONTENT_TYPE = "application/json";  
    private static final String USERNAME_KEY = "email";  
    private static final String PASSWORD_KEY = "password";  
    private static final AntPathRequestMatcher DEFAULT_LOGIN_PATH_REQUEST_MATCHER = new AntPathRequestMatcher(DEFAULT_LOGIN_REQUEST_URL, HTTP_METHOD);  
  
    private final ObjectMapper objectMapper;  
  
    public CustomLoginFilter(ObjectMapper objectMapper) {  
        super(DEFAULT_LOGIN_PATH_REQUEST_MATCHER);  
        this.objectMapper = objectMapper;  
    }  
  
    @Override  
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException, IOException {  
	    // application/json 이 아니면 예외 발생
        if (request.getContentType() == null || (!request.getContentType().equals(CONTENT_TYPE) && !request.getContentType().startsWith(CONTENT_TYPE))) {  
            throw new AuthenticationServiceException("Authentication Content-Type not supported: " + request.getContentType());  
        }  
        String messageBody = StreamUtils.copyToString(request.getInputStream(), StandardCharsets.UTF_8);  
        Map<String, String> usernamePasswordMap = objectMapper.readValue(messageBody, Map.class); 
        // 요청 받은 messageBody 는 json 형식이므로 ObjectMapper 로 email 과 password 추출 
        String email = usernamePasswordMap.get(USERNAME_KEY);  
        String password = usernamePasswordMap.get(PASSWORD_KEY);  
        // email 과 password 로 인증처리
        UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(email, password);  
        return this.getAuthenticationManager().authenticate(authRequest);  
    }  
}
```

- 로그인 경로 설정을 위 필터에서 설정한다.(POST, Content-Type 등)
- email 과 password 로 인증처리를 하고 __`UsernamePasswordAuthenticationToken` 객체를 만들어  Manager 에게 반환한다.__

#### Provider 에게 비밀번호 비교 위임

```java
public class SecurityConfig {
	@Bean  
    public AuthenticationManager authenticationManager(PasswordEncoder passwordEncoder) {  
        DaoAuthenticationProvider authenticationProvider = new DaoAuthenticationProvider();  
        authenticationProvider.setUserDetailsService(adminUserDetailsService); // 사용자 정보 가져와서  
        authenticationProvider.setPasswordEncoder(passwordEncoder); // provider 안에서 암호화된 비밀변호 db 와 비교  
        return new ProviderManager(authenticationProvider);  
    }  
  
    @Bean  
    public CustomLoginFilter customLoginFilter() {  
        CustomLoginFilter filter = new CustomLoginFilter(objectMapper); // ObjectMapper는 DI 받아야 함  
        AuthenticationManager authenticationManager = authenticationManager(passwordEncoder());  
        filter.setAuthenticationManager(authenticationManager);  
        filter.setAuthenticationSuccessHandler(adminLoginSuccessHandler);  
        filter.setAuthenticationFailureHandler(adminLoginFailureHandler);  
        return filter;  
    }  
  
    @Bean  
    public PasswordEncoder passwordEncoder() {  
        return new BCryptPasswordEncoder();  
    }  
}
}
```

- Manager 는 Provider 에게 객체를 반환하여 Provider 는 __(1) [[UserDetailsService]] 에서 email 의 사용자 정보를 DB 에서 찾아오고__, (2) 찾아온 암호화된 비밀번호를 __`PasswordEncoder` 로 비교한다.__

#### LoginSuccessHandler / LoginFailureHandler

```java

// 성공 핸들러
public class AdminLoginSuccessHandler extends SimpleUrlAuthenticationSuccessHandler {  
  
    private final AdminLoginService adminLoginService;  
    private final ObjectMapper objectMapper;  
  
    @Override  
    @Transactional    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,  
                                        Authentication authentication) throws IOException {  
        String email = extractUsername(authentication);  
        LoginResponse loginResponse = adminLoginService.login(email);  
        sendLoginSuccessResponse(response, loginResponse);  
  
        log.info("로그인에 성공한 관리자 정보 : " + email);  
    }  
  
    private void sendLoginSuccessResponse(HttpServletResponse response, LoginResponse loginResponse) throws IOException {  
        response.setStatus(HttpServletResponse.SC_OK);  
        response.setContentType("application/json");  
        response.setCharacterEncoding("UTF-8");  
  
        response.addHeader("Set-Cookie", loginResponse.getRefreshTokenCookie().toString());  
  
        objectMapper.writeValue(response.getWriter(),  
                Map.of("role", loginResponse.getRole(), "accessToken", loginResponse.getAccessToken()));  
    }  
  
    private String extractUsername(Authentication authentication) {  
        CustomUserDetails userDetails = (CustomUserDetails) authentication.getPrincipal();  
        return userDetails.getUsername();  
    }

// 실패 핸들러
@Component
public class AdminLoginFailureHandler extends SimpleUrlAuthenticationFailureHandler {  
  
    @Override  
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response,  
                                        AuthenticationException exception){  
        throw new AuthException(ErrorResponseCode.FAIL_LOGIN, "로그인 실패");  
    }  
}
```

- Manager 는 Provider 에게 비밀번호 일치 여부에 대한 데이터를 받고 Filter 에게 전달하여 Filter 는 인증 성공 시 `LoginSuccessHandler`를 호출하고, 인증 실패 시 `LoginFailureHandler`가 호출된다.
- 인증에 성공한 사용자는 [[LoginService]]를 호출한다.
- 서버는 생성된 사용자의 **AccessToken 은 body** 에 담아서 보내고, Cookie 는 Header 에 담아서 보낸다.
- 이후 **API 요청에서 클라이언트는 AccessToken 을 Header에 담아서 요청**한다.
- 핸들러들은 Bean 으로 등록되어야 하기 때문에 @Component 어노테이션을 추가한다.