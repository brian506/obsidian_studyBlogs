#backend #class #JWT #security #spring 

____

## 고려해야 할 부분들

- 필터에서 인증 성공된 이후에 사용자의 Credentials 를 토대로 AccessToken 과 RefreshToken 을 생성한다.
- JwtService 에는 __토큰 검증, 토큰 생성, 토큰 추출__ 메서드를 포함한다.

___

## 코드 구현

### JwtService

```java
public class AdminJwtService {  
  
    private final SecretKey secretKey;  
  
    @Value("${spring.application.name}")  
    private String issuer;  
  
    @Value("${jwt.access.expiration}")  
    private long accessKeyExpiration; // 30분  
  
    @Value("${jwt.refresh.expiration}")  
    private long refreshKeyExpiration; // 14일  
  
  
    public AdminJwtService(@Value("${jwt.secret-key}") String secretKey) {  
        this.secretKey = Keys.hmacShaKeyFor(Decoders.BASE64.decode(secretKey));  
    }  
  
    // 커스텀 예외 추가  
    public Claims verifyToken(final String token) throws JwtValidationException {  
        return Jwts.parserBuilder()  
                .setSigningKey(secretKey)  
                .build()  
                .parseClaimsJws(token)  
                .getBody();  
    }  
  
    public String createAccessToken(final AccessTokenPayload payload) {  
        return Jwts.builder()  
                .setSubject(payload.email())  
                .claim("role", payload.role().getKey())  
                .setIssuer(issuer)  
                .setIssuedAt(payload.date())  
                .setExpiration(new Date(payload.date().getTime() + accessKeyExpiration * 1000L))  
                .signWith(SignatureAlgorithm.HS512, secretKey)  
                .compact();  
    }  
  
    public String createRefreshToken(final RefreshTokenPayload payload) {  
        return Jwts.builder()  
                .setSubject(payload.email())  
                .setIssuer(issuer)  
                .setIssuedAt(payload.date())  
                .setExpiration(new Date(payload.date().getTime() + refreshKeyExpiration + 1000L))  
                .signWith(SignatureAlgorithm.HS512, secretKey)  
                .compact();  
    }  
  
  
    public Optional<String> extractAccessToken(HttpServletRequest request) {  
        return Optional.ofNullable(request.getHeader("Authorization"))  
                .filter(accessToken -> accessToken.startsWith("Bearer "))  
                .map(accessToken -> accessToken.replace("Bearer ", ""));  
    }
```

___

### CookieService

```java
public class CookieService {  
  
    @Value("${jwt.refresh.expiration}")  
    private long refreshKeyExpiration;  
  
    private final String REFRESH_TOKEN = "refreshToken";  
  
    public Optional<String> getRefreshTokenFromCookie(HttpServletRequest request) {  
        Cookie[] cookies = request.getCookies();  
  
        if (cookies == null || cookies.length == 0) {  
            return Optional.empty();  
        }  
  
        for (Cookie cookie : cookies) {  
            if (cookie.getName().equals(REFRESH_TOKEN)) {  
                return Optional.ofNullable(cookie.getValue());  
            }  
        }  
        return Optional.empty();  
    }  
  
  
    public ResponseCookie createRefreshTokenCookie(final String refreshToken) {  
        return ResponseCookie.from("refreshToken",refreshToken)  
                .httpOnly(true)  
                .secure(false)  
                .maxAge(refreshKeyExpiration)  
                .sameSite("Strict")  
                .path("/")  
                .build();  
    }
```

- `httpOnly(true)` : XSS 공격 방지
- `secure(false)` : true 로 하면 HTTPS 연결을 통해서만 전송. false 시 HTTP 도 가능
- `samsite("Strict")` : CSRF 방지
- `path("/")` : 쿠키가 적용될 URL 경로 지정

___

### LoginService

```java
public class AdminLoginService {  
  
    @Value("${jwt.access.expiration}")  
    private long accessKeyExpiration; // 30분  
  
    @Value("${jwt.refresh.expiration}")  
    private long refreshKeyExpiration; // 14일  
  
    private final AdminUserRepository adminUserRepository;  
    private final CookieService cookieService;  
    private final AdminJwtService jwtService;  
    private final RefreshTokenRepository refreshTokenRepository;  
  
    @Transactional  
    public LoginResponse login(final String email){  
        AdminUser admin = OptionalUtil.getOrElseThrow(adminUserRepository.findByEmail(email),"존재하지 않는 관리자 이메일입니다.");  
  
        String refreshToken = jwtService.createRefreshToken(new RefreshTokenPayload(admin.getEmail(), new Date()));  
        updateRefreshToken(admin,refreshToken);  
  
        String accessToken = jwtService.createAccessToken(new AccessTokenPayload(admin.getEmail(),admin.getRole(),new Date()));  
        ResponseCookie responseCookie = cookieService.createRefreshTokenCookie(refreshToken);  
        return new LoginResponse(admin.getRole(),accessToken,responseCookie);  
  
    }  
  
    public void updateRefreshToken(AdminUser admin,String token){  
        refreshTokenRepository.findByAdmin(admin).ifPresent(refreshTokenRepository::delete);  
        refreshTokenRepository.save(RefreshToken.builder()  
                .token(token)  
                .createdAt(LocalDateTime.now())  
                .expiredAt(LocalDateTime.now().plus(Duration.ofMillis(refreshKeyExpiration)))  
                .build());  
    }
```

- 기간이 지난 리프레시 토큰은 재발급하여 저장