#MSA 

# 개요
MSA 아키텍처에서 사용자가 서버에 요청을 보내면 먼저 Gateway 가 이 요청을 받아서 처리하게 된다. 
Gateway 는 받은 요청을 해당 서비스로 보내야 하는데, 여기서 **서비스를 찾기 위해서 Eureka Server 가 필요**로 한다.
___
## 특징
![사진](image/eureka.png)

- **Serivce Registry(서비스 주소록)** : Eureka 는 각 서비스의 IP / PORT / InstanceId 를 가지고 있고 Rest 기반으로 작동
- **Client-Server 구조** : Eureka Server에 등록된 서비스들은 Eureka Client 로 불림
- **Client끼리 통신** : Eureka Client 끼리 통신할 때 Eureka Server 에서 조회한 정보로 통신
- **Server-Client TTL** 
	- 1. Client -> Server : 30초마다 HeartBeats 요청
	- 2. Server : 90 초 안에 HeartBeats 가 도착하지 않으면 Client 제거

## 구현

### Eureka Server
- Eureka-Server 의존성 추가 (build.gradle)
- 메인 클래스에 `@EnableEurekaServer` 추가

```yml
spring:  
  application:  
    name: eureka-server  
  
server:  
  port: 8761  
  
eureka:  
  client:  
    # false: 나는 서버이므로, 나 자신을 다른 유레카 서버에 클라이언트로 등록하지 않음  
    register-with-eureka: false  
  
    # false: 나는 서버이므로, 다른 유레카 서버로부터 서비스 목록을 가져오지 않음  
    fetch-registry: false
```

### Eureka Client
- Eureka-Client 의존성 추가 (build.gradle)
- 메인 클래스에 `@EnableDiscoveryClient` 추가

```yml
server:  
  port: 8081
eureka:  
  client:  
    register-with-eureka: true  
    fetch-registry: true  
    service-url:  
      defaultZone: http://eureka-server:8761/eureka/
```

- 각 Client Service 들은 yml 파일의 자신의 포트번호를 지정
	- Eureka Server 는 이 포트번호로 Service 찾음

이후 http://localhost:8761 에 들어가서 접속해 있는 서비스들 모니터링 가능

