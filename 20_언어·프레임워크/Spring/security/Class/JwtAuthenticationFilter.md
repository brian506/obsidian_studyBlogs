#backend #class #JWT #security #spring 

___

## 개요

- 클라이언트가 로그인 이후에 API 요청을 할 때 보호된 API 에 요청한다.
- 프론트에서 AccessToken 을 꺼내서 요청 헤더에 보내고 [[JwtAuthenticationFilter]] 가 호출된다.

___

## 고려해야할 점

- 요청의 Authorization 항목 확인하고, Bearer 접두사를 떼어낸 AccessToken 을 추출한다.
- 토큰의 **서명을 확인하고 만료 시간**이 지나지 않았는지 검증한다.
- 필터는 유효한 토큰의 내용에 있는 이메일,권한 등의 정보를 추출한다.
- RefreshToken 이 유효하지 않으면 새롭게 재발급한다.
- 이 정보를 바탕으로 Authentication 객체를 생성하여 SecurityContextHolder 에 저장한다.
- 이후  `SecurityConfig` 에 있는 경로와 권한에 맞는지 확인하고 인가 검사를 수행한다.

___

## 코드 구현

### JwtAuthenticationFilter

```java
public class AdminJwtAuthenticationFilter extends OncePerRequestFilter {  
  
    private static final String NO_CHECK_URL = "/v1/api/admin/login";  
  
    private final AdminJwtService jwtService;  
    private final AdminUserRepository adminUserRepository;  
    private final CookieService cookieService;  
  
    @Override  
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {  
  
        if (request.getRequestURI().equals(NO_CHECK_URL)) {  
            filterChain.doFilter(request, response);  
            return;  
        }  
        Optional<String> accessTokenOpt = jwtService.extractAccessToken(request);  
  
        if (accessTokenOpt.isEmpty()) {  
            filterChain.doFilter(request, response);  
            return;  
        }  
        String accessToken = accessTokenOpt.get();  
  
        try {  
            jwtService.verifyToken(accessToken);  
            setAuthentication(accessToken);  
            filterChain.doFilter(request, response);  
  
        } catch (ExpiredJwtException e) {  
            log.info("AccessToken 이 만료되었습니다. RefreshToken 재발급 시도");  
            try {  
                refreshTokensAndContinue(request, response, filterChain);  
            } catch (JwtException | IOException | ServletException refreshException) {  
                log.warn("RefreshToken 재발급 실패: {}", refreshException.getMessage());  
                handleAuthError(response, "로그인 재시도 필요");  
            }  
        }  
    }  
  
    private void refreshTokensAndContinue(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws IOException, ServletException {  
  
        String refreshToken = cookieService.getRefreshTokenFromCookie(request)  
                .orElseThrow(() -> new JwtException("존재하지 않는 cookie"));  
        jwtService.verifyToken(refreshToken);  
  
        AdminUser admin = OptionalUtil.getOrElseThrow(adminUserRepository.findByRefreshToken(refreshToken),"존재하지 않는 관리자입니다.");  
  
        String newAccessToken = jwtService.createAccessToken(  
                new AccessTokenPayload(admin.getEmail(), admin.getRole(), new Date())  
        );  
  
        response.setHeader(HttpHeaders.AUTHORIZATION, "Bearer " + newAccessToken);  
        setAuthentication(newAccessToken);  
        filterChain.doFilter(request, response);  
    }  
  
  
    private void setAuthentication(String accessToken) {  
        Claims claims = jwtService.verifyToken(accessToken);  
        String email = claims.getSubject();  
        String role = claims.get("role", String.class);  
        GrantedAuthority grantedAuthority = new SimpleGrantedAuthority(role);  
        Authentication authentication = new UsernamePasswordAuthenticationToken(email, null, List.of(grantedAuthority));  
        SecurityContextHolder.getContext().setAuthentication(authentication);  
    }  
  
    private void handleAuthError(HttpServletResponse response, String message) throws IOException {  
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);  
        response.setContentType("application/json");  
        response.setCharacterEncoding("UTF-8");  
        response.getWriter().write(String.format("{\"error\": \"%s\"}", message));  
    }  
}
```

1. AccessToken 의 유효성 검증
2. AccessToken 이 유효하지 않거나 없으면 다음 인가 필터로 넘어가서 접근 제한
3. AccessToken 검증하고 유효하면 SecurityContext 에 인증 정보 저장하고 다음 필터로 이동
4. AccessToken 이 만료됐으면 RefreshToken으로 새로운 AccessToken 생성하고 인증정보 저장하고 다음 필터 이동

- 여기서 SecurityContext 는 로그인된 사용자를 저장하는 객체임
