> Ticket-on 은 대규모 트래픽 속에서도 안정적으로 티켓을 예매할 수 있는 분산 티켓팅 시스템입니다.
>실시간 경쟁 상황에서도 공정성과 성능을 최우선으로 설계하였습니다.

#project #backend #database #spring #docker #kafka 

✅ 목표 : 여러 사용자의 예약 요청과 결제 요청을 짧은 시간 내에 데이터 손실 없이 정확한 예약을 할 수 있게 하는 것이 목표입니다.

### 기술 스택

[](https://github.com/Ticket-on/Ticket-on-BE#%EA%B8%B0%EC%88%A0-%EC%8A%A4%ED%83%9D)

**Version** : `JDK21`  
**Backend** : `Spring Boot`, `JPA`, `QueryDSL`  
**Database** : `MySQL`, `kafka`
**Devops** : `Nginx`, `Docker`

### 시스템 아키텍처(예약)

![사진](flow.png)

## 참고 링크

### 시스템 아키텍처(결제)
![사진](kafkastruc.png)



## 참고 링크

- [[🎫  예약,결제 요청 건에 대한 Nginx 처리]]
- [[결제 요청 건에 대한 Kafka Producer]]
- [[결제 요청 건에 대한 Kafka Consumer]]
- [[🧪 레디스 락의 트랜잭션 정합성 보장과 부하테스트 성능 측정]]
- 기술을 선택한 이유

___



### 부하 테스트 모니터링

![사진](stress.png)

- Opentelemetry : 앱 내부 로그, 메트릭,트레이스 수집
- Loki : 애플리케이션 시스템 로그 수집 및 검색
- Prometheus : CPU, 메모리, RPS, 오류율 등 수치 기반 성능 메트릭 수집
- Tempo : 트레이스 데이터 시각화
- Grafana : 통합 대시보드 시각화
- PMM(Percona Monitoring and Management) : MySQL 쿼리 성능, 슬로우 쿼리, 커넥션, 인덱스 분석

# 어플리케이션 실행 방법
