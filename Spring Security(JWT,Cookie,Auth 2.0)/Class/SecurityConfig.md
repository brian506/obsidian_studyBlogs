#backend #class #JWT #security #spring 

___

## 개요

- SpringSecurity 의 전반적인 모든 설정을 담당하는 Config 설정 클래스

___

### SecurityConfig

[[JwtAuthenticationFilter]] 이후의 다음 Filter(authorizeHttpRequests()) 동작

```java
@Bean  
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {  
    http.csrf(AbstractHttpConfigurer::disable)  
            .cors(cors -> cors.configurationSource(corsConfig.corsConfigurationSource()))  
            .formLogin(AbstractHttpConfigurer::disable)  
            .httpBasic(AbstractHttpConfigurer::disable)  
            .addFilterAt(customLoginFilter(), UsernamePasswordAuthenticationFilter.class)  
            .addFilterAfter(jwtAuthenticationFilter, BasicAuthenticationFilter.class)  
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))  
            .authorizeHttpRequests(authorize -> authorize  
                    //  OPTIONS 요청은 인증 없이 항상 최우선으로 허용  
                    .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()  
                    .requestMatchers("/v1/api/admin/login").permitAll()  
                    .requestMatchers("/v1/api/admin/**").hasAuthority("ROLE_ADMIN")  
                    .anyRequest().authenticated()  
            );  
    return http.build();  
}
```

- `.csrf(disable)` : CSRF 공격 방어 기능 비활성화 (토큰 방식(Stateless이므로 안전)
- `.formLogin(disable)` : form 기반 로그인 비활성화 (REST API 이용하므로)
- `.cors(corsConfig)` : CORS 설정 활성화
- `.sessionManagement()` : 세션 관리 정책 'STATELESS' 로 설정
	- 서버가 클라이언트의 상태를 기억하지 않을 것이라는 의미
	- 독립적으로 JWT 가 포함된 요청에 한해서만 인증

- **.hasRole("ADMIN")** 코드는 **"ROLE_ADMIN"** 로 Role 에 저장됨
	- .claim("role",payload.role().getKey())
		-> 여기서 .getKey() 로 받아야 ROLE_ADMIN 으로 저장

___
#### 주의할 점
  
>Securing OPTIONS /v1/api/admin/users/delete/...
  Securing DELETE /v1/api/admin/users/delete/...

-> 다른 출처(Cross-origin)로 위험할 수 있는 요청을 보내기 전에 먼저 HttpMethod.OPTIONS 로 서버에게 안전한 요청인지 물어보는 느낌임

1. 동일 출처 정책(SOP)

- 도메인 주소가 같은 서버에서의 요청만을 같은 출처로 봄

2. SOP 의 예외

- CORS 정책으로 교차 출처 리소스 공유하는 것이고 요청을 두개로 나눈다.
- 단순 요청 : 데이터를 바꾸지 않는 GET 요청
- Preflight request : 데이터를 바꾸는 요청(DELETE,POST 등 권한 헤더가 포함되어있는)

  **DELETE 요청**은 서버의 데이터를 삭제하는 **'위험할 수 있는 요청'**에 해당한다. 따라서 브라우저는 보안상의 이유로 반드시 OPTIONS 사전 요청을 먼저 보내 서버의 허락을 받아야만** 한다.
	-> config 에서 맨앞에 HttpMethod.Options 허용해줌
