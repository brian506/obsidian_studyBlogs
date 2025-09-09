#JWT #security #redis 

## 개요

JWT 를 구현할 때 로그인 할때마다 AccessToken 과 RefreshToken 을 같이 발급한다.
하지만 로그인 이후에 AccessToken은 만료기간이 짧기 때문에 토큰의 탈취 위험을 방지하기 위해 RefreshToken 을 이용해서 AccessToken 을 재발급한다.

보통 AccessToken 만료기간은 30분 ~ 2시간 , RefreshToken 만료기간은 1주 ~ 1개월 정도로 지정한다.

많은 사용자가 한번에 로그인 요청을 할 때 RefreshToken 을 MySQL 에 저장하게 되면 Disk I/O 의 낮은 성능, 사용 가능한 DB Connection 개수가 제한되어 있고 토큰 데이터가 쌓이게 되면 삭제하는 쿼리 등 병목을 발생시킬 것이다.

Redis 를 이용하여 데이터를 영구적으로 저장할 수는 없지만, TTL(Time To Live) 을 통해 데이터의 유효기간을 정하면 자동으로 만료된 토큰을 삭제할 수 있다.

AccessToken 만료 시 RefreshToken 재발급 API 요청(Oauth 2 라이브러리 이용) 하는 것이 아닌 **Filter 내에서 처리 하는 방식**으로 구현해 볼 것이다.

인증 Filter 를 사용하는 것이 서버 내에서 토큰 유효성 등을 해결하므로 클라이언트에게 에러 응답을 보낼 필요도 없기 때문에 **지연시간이 짧아진다.**

___
### Redis Hash

```java
@RedisHash(value = "refresh_token",timeToLive = 604800) // 7일  
@AllArgsConstructor  
@Getter  
@ToString  
public class Token  {  
  
    @Id  
    private String email;  
  
    private String refreshToken;  
  
    private Role role;  
  
}

@Repository  
public interface TokenRepository extends CrudRepository<Token,Long> {  
  
    Optional<Token> findByEmail(String email);  
  
    Optional<Token> findByRefreshToken(String refreshToken);  
}
```

- RedisRepository 를 이용해서 Token 엔티티에 사용자 정보값이 들어간다.

___

### Redis 데이터 보는 방법

1. Redis 서버 접속 : `redis-cli -h {host} -p {포트번호} -a {password}` 
	- `localhost:6379>`
2. 저장된 키 조회 : `keys refresh_token:*`
	- `1) "refresh_token:brianchoi506@gmail.com"
3) 데이터 조회 : `HGETALL refresh_token:user@example.com`

```yml
1) "refreshToken"
2) "eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJicmlhbmNob2k1MDZAZ21haWwuY29tIiwiaXNzIjoiYXV0aC1zZXJ2aWNlIiwiaWF0IjoxNzU3MjQ5Nzg2LCJleHAiOjE3NTg0NTkzODd9.P7AiIwig4nKJ-QCP4To26L-Azp2s9FSFIQbKW-afm2cGHq3SQKW1LW7Mru6_Wngan5RNeYzM0FnQ2TxBO7Eo1w"
3) "email"
4) "brianchoi506@gmail.com"
5) "role"
6) "GENERAL"
7) "_class"
8) "org.auth.domain.entity.Token"
```

___

### 두 방식의 동작 비교

#### 1. 필터에서 재발급하는 방식

2. **[Request]** 클라이언트 → 서버: 만료된 Access Token으로 API 요청 (50ms 소요)
3. **[Server Process]** 서버 필터:
- ExpiredJwtException 감지
- 쿠키에서 Refresh Token 추출
- Redis 조회 및 검증
- **새로운 Access Token 생성**
- 원래 요청된 비즈니스 로직 처리
2. **[Response]** 서버 → 클라이언트: **최종 API 결과와 새로운 Access Token을 한 번에 응답**
- **총 네트워크 왕복 횟수**: **1회**
- **클라이언트가 느끼는 총 소요 시간**: (서버 처리 시간) + **50ms**
  
#### 2. API를 호출하여 재발급하는 방식 (컨트롤러)
1. **[Request 1]** 클라이언트 → 서버: 만료된 Access Token으로 API 요청 (50ms 소요)
2. **[Response 1]** 서버 → 클라이언트: 401 Unauthorized 에러 응답 (50ms 소요)
3. **[Request 2]** 클라이언트: 401 에러를 인지하고, /api/auth/reissue 재발급 API 호출 (50ms 소요)
4. **[Server Process]** TokenController:
- 쿠키에서 Refresh Token 추출
- Redis 조회 및 검증
- **새로운 Access Token 생성**
1. **[Response 2]** 서버 → 클라이언트: 새로운 Access Token 응답 (50ms 소요)
- 여기까지 토큰을 새로 받기 위해 추가로 **100ms**가 더 소요되었습니다.
1. **[Request 3]** 클라이언트: 새로 받은 Access Token으로 **원래 API를 다시 요청** (50ms 소요)
2. **[Response 3]** 서버 → 클라이언트: 최종 API 결과 응답
- **총 네트워크 왕복 횟수**: **2회 이상** (재발급 1회 + 원래 요청 재시도 1회)
- **클라이언트가 느끼는 총 소요 시간**: (서버 처리 시간 x 2) + **최소 200ms 이상**

##### 성능 차이 비교

|   |   |   |   |
|---|---|---|---|
|항목|필터 방식|API 호출 방식|비고|
|**서버** **처리** **시간**|거의 동일|거의 동일|핵심 연산(Redis 조회, JWT 생성)은 동일|
|**네트워크** **오버헤드**|**낮음** (1번 왕복)|**높음** (추가 왕복 필요)|**가장** **큰** **차이점**|
|**클라이언트** **체감** **속도**|**빠름**|**느림**|네트워크 왕복 시간만큼의 차이 발생|
