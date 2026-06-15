

### 배경: 왜 메시지 서비스가 필요한가?

동기적 통신은 트래픽 폭증 시 한쪽 서비스가 과부하로 중단됨. 서비스를 **분리(decouple)**하기 위해 비동기 메시지 서비스를 사용.

- **SQS**: 대기열 모델 (생산자 → 대기열 → 소비자)
- **SNS**: Pub/Sub 모델 (1개 토픽 → 다수 구독자)
- **Kinesis**: 실시간 스트리밍, 대용량 데이터

---

### 1. Amazon SQS ⭐⭐⭐

**Standard Queue 특성:**

- **무제한 처리량**, 저장 메시지 수 제한 없음
- 메시지 보존: 기본 **4일**, 최대 **14일**
- 지연 시간: 10ms 미만
- 메시지 크기 최대 **256KB**
- **중복 메시지 가능** (at-least-once), **순서 보장 안 됨**

**Message Visibility Timeout:** 소비자가 메시지를 가져가면 다른 소비자에게 **일시적으로 안 보임**. 기본 **30초**. 처리 중 시간이 더 필요하면 `ChangeMessageVisibility` API로 연장. 시간 초과 후 삭제 안 됐으면 대기열에 다시 나타남.

**Long Polling (긴 폴링):** 대기열이 비어있을 때 소비자가 메시지가 올 때까지 기다리는 방식. SQS API 호출 수 감소, 효율 향상. 대기 시간 1~20초 설정, **20초 권장**. `WaitTimeSeconds`로 설정.

**FIFO Queue 특성:**

- 순서 보장(First In First Out), 중복 제거(exactly-once)
- 처리량 제한: 배치 없이 **300 msg/s**, 배치 사용 시 **3,000 msg/s**
- FIFO 대기열 이름은 `.fifo`로 끝나야 함

**Dead Letter Queue (DLQ):** 처리 실패 메시지를 일정 횟수(`maxReceiveCount`) 이상 재시도 후 DLQ로 이동. 디버깅에 유용. DLQ도 Standard/FIFO에 따라 맞게 설정해야 함. 보존 기간을 최대 14일로 설정 권장.

**Delay Queue:** 메시지를 최대 **15분**까지 지연 후 소비자에게 전달.

**SQS + ASG 패턴:** CloudWatch Alarm → ASG 스케일링. SQS를 DB 앞에 버퍼로 둬서 트랜잭션 유실 방지.

---

### 2. Amazon SNS ⭐⭐

**pub/sub 모델**: 생산자 하나 → SNS 토픽 → 다수 구독자에게 동시 전달. (MSA 환경에 적합)

- 주제당 최대 **12,500,000개** 구독
- 최대 주제 수: **100,000개**
- 구독자: SQS, Lambda, Kinesis Firehose, HTTP/HTTPS, Email, SMS, 모바일 푸시
- 메시지 저장 X
- 일반적으로는 순서를 보장하지 않음

**SNS + SQS Fan Out 패턴 ⭐⭐⭐:** SNS 토픽 하나에 여러 SQS 큐를 구독시켜 동시에 메시지 전달. 서비스 추가 시 SQS만 구독 추가하면 됨. SQS에 대한 SNS 쓰기 권한 필요. **시험에 매우 자주 나옴!**

**Message Filtering:** JSON 정책으로 구독자에게 특정 조건의 메시지만 전달. 필터 정책 없으면 모든 메시지 수신.

**SNS FIFO:** SQS FIFO처럼 순서 지정 + 중복 제거. 구독자는 **SQS FIFO 대기열만** 가능.

---

### 3. Amazon Kinesis ⭐⭐

실시간 스트리밍 데이터 처리. 로그, 메트릭, 클릭스트림, IoT 데이터 등.

**4가지 구성 요소:**

- **Data Streams**: 수집·처리·저장
- **Data Firehose**: AWS 저장소로 로드 (S3, Redshift, OpenSearch 등)
- **Data Analytics**: SQL/Flink로 실시간 분석
- **Video Streams**: 비디오 처리

**Kinesis Data Streams 특성:**

- 샤드(Shard) 단위로 구성
- 보존 기간: 1일~**365일**
- 데이터 삽입 후 삭제 불가 (**불변성**)
- 동일 파티션 키 → 동일 샤드 → **순서 보장**
- Provisioned mode: 샤드당 입력 1MB/s, 출력 2MB/s, 시간당 과금
- On-demand mode: 자동 스케일링, 기본 4MB/s, 사용량 기반 과금

---

### 4. SQS vs SNS vs Kinesis 비교 ⭐⭐⭐

![](../../images/스크린샷%202026-04-12%2020.37.10.png)

---

### 5. Amazon MQ ⭐

기존 온프레미스 메시지 브로커(RabbitMQ, ActiveMQ)를 사용하던 애플리케이션을 **클라우드로 마이그레이션**할 때 사용. MQTT, AMQP, STOMP 등 오픈 프로토콜 지원.

- SQS/SNS처럼 확장성이 크지 않음 — **서버 기반**
- Multi-AZ로 장애 조치 설정 가능 → 백엔드 스토리지로 **EFS** 사용
- SQS 유사 기능(대기열) + SNS 유사 기능(토픽) 모두 제공

> **시험 TIP**: "기존 MQTT/AMQP 프로토콜 유지하면서 마이그레이션" → **Amazon MQ**, 새로 만드는 서비스면 → SQS/SNS

---

### 시험 TIP 최종 정리

- **SQS 메시지 크기**: 최대 **256KB**
- **SQS 기본 보존**: 4일, 최대 **14일**
- **Visibility Timeout 기본**: **30초**
- **Long Polling**: `WaitTimeSeconds` 설정, 최대 **20초**
- **FIFO 처리량**: 배치 없이 300/s, 배치 사용 **3,000/s**
- **Fan Out** = SNS + SQS 조합 → 하나의 이벤트를 여러 서비스에 동시 전달
- **Kinesis 불변성**: 삽입 후 삭제 불가
- **Kinesis 순서 보장**: 같은 파티션 키 → 같은 샤드
- **Amazon MQ**: 기존 브로커 마이그레이션용, MQTT/AMQP 지원


