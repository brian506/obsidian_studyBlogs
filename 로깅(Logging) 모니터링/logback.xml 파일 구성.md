
로그를 어디에 , 어떤 모양으로, 얼마나 남길지를 설정하는 파일

`<appender>` : 변수 정의
`<logger>` : 특정 라이브러리의 로그를 얼마나 자세하게 볼지 레벨 설정
`<appender>` : 로그를 어디로 보낼지 정의
`<root>` : 위에서 만든 배송지들을 실제로 켜는 스위치

### 변수 정의

```xml
<property name="APP_NAME" value="ticketon"/>  
<property name="LOG_PATH" value="/var/log/${APP_NAME}"/>  
<property name="LOG_FILE" value="${APP_NAME}"/>  
<property name="ERR_LOG_FILE" value="err-${APP_NAME}"/>  
  
<property name="CONSOLE_PATTERN"  
          value="%d{yyyy-MM-dd HH:mm:ss.SSS} %magenta([%thread]) %highlight([%-3level]) TraceID[%X{traceId:-NoTraceId}] %logger{36} - %msg%n"/>  
<property name="FILE_PATTERN"  
          value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] [%-5level] TraceID[%X{traceId:-NoTraceId}] %logger{36} - %msg%n"/>
```

`APP_NAME`: 어디서 온 건지 출처를 알기 위해 꼬리표(label)를 달때 사용
`LOG_PATH` : 로그가 저장되는 경로(컨테이너 내부의 가상 경로)
`FILE_PATTERN`: 로그가 찍히는 모양 결정
	- `%d`: 날짜/시간
	- `%thread` : 어떤 스레드인지
	- `&-5level`: 로그 레벨
	- `%X{traceId}` : 분산 추적 ID


### 로그 레벨 조절

```xml
<logger name="org.springframework" level="INFO" />  
<logger name="org.springframework.boot" level="INFO" />  
<logger name="org.hibernate" level="WARN" />
```

`hibernate` 를 WARN 으로 둔 이유
- SQL 실행 마다 모든 로그가 찍히기 때문에 로그 레벨 조절

보통 dev 환경에서 로그 레벨 제한하고 prod 레벨에서는 


### appender (로그를 어디에, 어떻게 출력할지)

```xml
<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">  
    <encoder>        
    <pattern>${CONSOLE_PATTERN}</pattern>  
    </encoder>
</appender>  
  
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">  
    <file>${LOG_PATH}/${LOG_FILE}.log</file>  
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">  
        <fileNamePattern>${LOG_PATH}/${LOG_FILE}.%d{yyyy-MM-dd}.log</fileNamePattern>  
        <maxHistory>30</maxHistory>  
        <totalSizeCap>1GB</totalSizeCap>  
    </rollingPolicy>  
      <encoder>        
      <pattern>${FILE_PATTERN}</pattern>  
    </encoder>
    </appender>  
  
<appender name="ERROR" class="ch.qos.logback.core.rolling.RollingFileAppender">  
    <filter class="ch.qos.logback.classic.filter.LevelFilter">  
        <level>error</level>  
        <onMatch>ACCEPT</onMatch>  
        <onMismatch>DENY</onMismatch>  
    </filter>    
    <file>${LOG_PATH}/${ERR_LOG_FILE}.log</file>  
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">  
        <fileNamePattern>${LOG_PATH}/${ERR_LOG_FILE}.%d{yyyy-MM-dd}.log</fileNamePattern>  
        <maxHistory>30</maxHistory>  
        <totalSizeCap>1GB</totalSizeCap>  
    </rollingPolicy>   
     <encoder>        
     <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>  
    </encoder>
    </appender>
```

`ConsoleAppender` : 로그를 가로채서 터미널에 나타냄
`RollingFileAppender` : 로그를 파일에 저장
`Rolling` : 기존 파일을 백업하고 새 파일에 기록 하는 과정

- `<TimeBaseRollingPolicy>` 
	- 현재 날짜가 경로 뒤에 있으므로, 날짜가 바뀌는 자정이 되면 `Rolling` 발생
	- `<maxHistory>` : 30일 기준으로 오래된 로그 파일은 자동 삭제
	- `<totalSizeCap>`: 1GB 가 차면 가장 오래된 파일부터 삭제

- `<filter>`
	- `error`레벨 제외하고는 거절
	- `error` 레벨의 로그만 저장