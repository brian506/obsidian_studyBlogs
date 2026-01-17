
어플리케이션에서 발생한 **로그와 매트릭**을 수집하는 모니터링 도구이다.
원래는 로그는 Promtail에서 수집, 매트릭은 바로 Prometheus 에서 수집하는데,
Promtail 지원을 중단해서 Alloy 를 이용해서 로그와 매트랙을 수집하려고 한다.

수집한 로그는 **Loki** 로 전송하고,
수집한 매트릭은 **Prometheus** 로 전송한다.

![](../images/스크린샷%202026-01-08%2014.42.41.png)
# Alloy, Prometheus 설정하기

## config.alloy

### 매트릭 수집

```yml
// [Metrics 수집 설정]
prometheus.scrape "{임의로 설정하는 앱 이름}" {  
  job_name = "{임의로 설정하는 앱 이름}"
  targets = [  
    { "__address__" = "{spring 컨테이너 이름}:8080" },  # 같은 네트워크이므로 내부포트로 바로 접속
  ]  
  metrics_path = "/actuator/prometheus"  # spring actuator 경로
  forward_to   = [prometheus.remote_write.default.receiver]  
}

// 수집한 메트릭을 Prometheus로 전송  
prometheus.remote_write "default" {  
  endpoint {  
    url = "http://prometheus:9090/api/v1/write"  
  }  
}

```

`prometheus.scrape`: 매트릭을 긁어오는 출처
`metrics_path` : target의 어디에서 매트릭을 가져올것인가 - `pull`방식
`forward_to` : 수집한 데이터를 alloy 내부의 어느 컴포넌트로 전송할 것인가
	- `prometheus.remote_write.default` 로 전송

Prometheus 가 Alloy 가 수집한 매트릭을 가져오는 방식(PULL)
	- 스프링이 `/actuator/prometheus` 경로에 매트릭 상태를 노출하고 대기
	- Prometheus 가 `remote_write` 로 데이터 긁어오는 형식
	

**스프링의 `/actuator` 경로 설정**

```yml
management:
  endpoints: 
    web: 
      exposure: 
	    include: prometheus # /actuator/prometheus 활성화
```

- Spring Security 에서 `/actuator` 경로 허용 해야함

**`build.gradlew`설정**

```yml
implementation 'org.springframework.boot:spring-boot-starter-actuator'
implementation 'io.micrometer:micrometer-registry-prometheus'
```
### 로그 수집

```go
// [Logs 수집 설정]

// 1. 파일 찾음
local.file_match "app_logs" {  
  path_targets = [{"__path__" = "/var/log/ticketon/*.log"}]  
}  
// 2. 파일 읽음
loki.source.file "app_logs_read" {  
  targets    = local.file_match.app_logs.targets  
  forward_to = [loki.process.multiline_detect.receiver]  
}  
// 3. 에러 로그 깨짐 방지(가공 단계)
loki.process "app_labels" {  
  forward_to = [loki.write.default.receiver]  
  
  stage.multiline {  
    firstline = "^\\d{4}-\\d{2}-\\d{2} \\d{2}:\\d{2}:\\d{2}\\.\\d{3}"  
    max_wait_time = "3s"  
  }  
  
  stage.regex {  
    expression = ".*(?P<level>INFO|WARN|ERROR|DEBUG|TRACE).*"  
  }  
  // 로그 레벨 라벨로 지정
  stage.labels {  
      values = {  
          level = "level",  
      }  
  }
  
  stage.static_labels {  
      values = {  
          service_name = "ticketon",  
          job = "var-log-collector",  
      }  
  }  
}  
// 4. Loki 로 전송
loki.write "default" {  
  endpoint {  
    url = "http://loki:3100/loki/api/v1/push"  
    tenant_id = "fake"  
    batch_wait = "1s"  
    batch_size = "1MB"  
  }  
}
```

Alloy 가 서버의 로그 파일을 직접 읽어 가공한 뒤 LoKi 로 쏘는 방식

**Spring 과 Alloy 가 로그 폴더를 Volume 으로 공유한다.**
`config.alloy` 에 있는 LOG_PATH 경로와 일치 - 로그 폴더

# Loki 설정하기
## loki-config.yml

```yml
auth_enabled: false  
  
server:  
  http_listen_port: 3100  
  
common:  
  path_prefix: /loki  
  storage:  
    filesystem:  
      chunks_directory: /loki/chunks  # 저장경로
      rules_directory: /loki/rules  
  replication_factor: 1  
  ring:  
    instance_addr: 127.0.0.1  
    kvstore:  
      store: inmemory  
  
schema_config:  
  configs:  
    - from: 2024-01-01  
      store: tsdb  # 저장 엔진
      object_store: filesystem  
      schema: v13  
      index:  
        prefix: index_  
        period: 24h  
  
compactor:  
  working_directory: /loki/compactor  
  compaction_interval: 10m  
  retention_enabled: true  
  delete_request_store: filesystem
  retention_delete_delay: 2h  
  retention_delete_worker_count: 150  
  
# 제한 설정: 7일 지난 로그 자동 삭제  
limits_config:  
  retention_period: 168h
```

# Grafana 설정하기

Grafana 콘솔 화면에 접속하면 Loki, Prometheus 를 따로 등록해야 되는데, 등록하지 않고 아래와 같은 파일을 만들어 놓으면 자동으로 등록된다.
![](../images/스크린샷%202026-01-11%2013.54.03.png)

### datasource.yml

```yml
apiVersion: 1  
  
datasources:  
  # 1. Prometheus 연결 설정  
  - name: Prometheus  
    type: prometheus  
    uid: prometheus_uid  
    access: proxy  
    url: http://prometheus:9090  
    isDefault: true   # 기본 데이터소스로 설정  
    editable: true  
  
  # 2. Loki 연결 설정  
  - name: Loki  
    type: loki  
    uid: loki_uid  
    access: proxy  
    url: http://loki:3100  
    editable: true
```


# 통합 docker-compose.yml

```yml

// spring application
spring-backend:
  container_name: ticketon
  volumes:
  # spring의 ./logs 폴더를 컨테이너 내부 로그 경로(/var/log/ticketon)와 연결
	- ./logs:/var/log/ticketon
  networks:
    - monitoring
      

// Alloy
alloy:  
  image: grafana/alloy:latest  
  container_name: alloy  
  restart: always  
  volumes:  
    # Alloy 설정 파일 마운트 (주의 : 패키지 경로와 같게 설정)
    - ./monitoring/alloy/config.alloy:/etc/alloy/config.alloy
    # spring이 쓰는 로그 폴더를 똑같이 마운트 (읽기 전용 :ro)   
    - ./logs:/var/log/ticketon:ro  
  command: [  
    "run",  
    # 외부접속 허용
    "--server.http.listen-addr=0.0.0.0:12345",  
     # Alloy 가 작동하면서 생기는 임시 데이터 저장 위치
    "--storage.path=/var/lib/alloy/data", 
    # 설정 파일 config.alloy 를 실행
    "/etc/alloy/config.alloy"  
  ]  
  ports:  
    - "12345:12345" # Alloy UI  
  networks:  
    - monitoring

// Prometheus
prometheus:  
  image: prom/prometheus  
  container_name: prometheus  
  ports:  
    - "9090:9090"  
  command:  
    - "--config.file=/etc/prometheus/prometheus.yml"  # 이미지 내부의 기본 설정 파일 적용
    - "--web.enable-remote-write-receiver"  # Alloy가 보내는 데이터 수신 허용
    - "--storage.tsdb.retention.time=15d"  # 데이터 보관 기간 15일
  volumes:  
    - prometheus-data:/prometheus  
  networks:  
    - monitoring

// Loki        
loki:  
  image: grafana/loki:latest  
  container_name: loki  
  ports:  
    - "3100:3100"  
  command: -config.file=/etc/loki/local-config.yaml  
  volumes:  
    - ./monitoring/loki/loki-config.yml:/etc/loki/loki-config.yml  # 설정 파일 
    - loki-data:/loki  # 데이터 저장소
  networks:  
    - monitoring

// Grafana
grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus
      - loki
    networks:
      - monitoring
    volumes:
      # 대시보드 저장용 (차트 날라가면 안 되니까 필수)
      - grafana-storage:/var/lib/grafana
      # 데이터소스 자동 설정을 위한 마운트 (monitoring 디렉토리 안에 grafana 디렉토리 있음)
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
```

#### alloy
- Spring Application 과 Alloy 가 똑같은 ./logs 폴더를 Volume 에 가지고 있어야 함
- `config.alloy`에 있는 `LOG_PATH`= `/var/log/ticketon`

#### prometheus
- Volume
	- `/prometheus` : 컨테이너 내부 경로
		- 컨테이너 안에서 데이터가 저장될 위치
	- `prometheus-data` : 도커 볼륨 이름
		- 내 컴퓨터가 관리할 영구 저장소

### 주의

1. **폴더 생성** : 터미널에서 미리 logs 폴더를 만들어서 권한 문제 방지
	> `mkdir logs`

2. **디렉토리 경로 설정**
	![](../images/스크린샷%202026-01-11%2015.25.44.png)
	- monitoring 디렉토리는 현재 최상단 루트 경로에 위치

	1. 만약 docker-compose-local.yml 이 루트 경로에 있고 이 파일로 실행하면?
		` volumes: ./monitoring/loki/loki-config.yml:/etc/loki/loki-config.yml`
		- 앞에 `./monitoring` 도 붙여야함
		- docker-compose 기준으로 상대 경로로 봐야함
		
		  