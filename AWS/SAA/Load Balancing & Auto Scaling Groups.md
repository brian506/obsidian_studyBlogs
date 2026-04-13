### 1. Scalability vs High Availability 개념 구분

**Scalability (확장성)** — 더 많은 부하를 처리할 수 있는 능력

- **수직 확장 (Scale Up/Down)**: 인스턴스 크기 자체를 키움. RDS, ElastiCache에서 사용. 하드웨어 한계 존재
- **수평 확장 (Scale Out/In)**: 인스턴스 수를 늘림. 분산 시스템 전제. EC2 + ASG가 대표 사례

**High Availability (고가용성)** — 2개 이상의 AZ에서 동시에 서비스가 돌아가는 것. 하나의 센터가 죽어도 서비스 유지가 목표.

> **시험 포인트**: Scalability ≠ High Availability. 둘은 연관되지만 다른 개념입니다.

---

### 2. 로드 밸런서 4종류 ⭐⭐⭐ (가장 중요!)

|종류|레이어|프로토콜|키워드|
|---|---|---|---|
|**ALB**|L7|HTTP, HTTPS, WebSocket|URL/호스트/쿼리 기반 라우팅, 마이크로서비스, 컨테이너|
|**NLB**|L4|TCP, UDP, TLS|고성능, 초당 수백만 req, 고정 IP, 지연 100ms|
|**GWLB**|L3|IP|방화벽/침입탐지 등 3rd party 어플라이언스, GENEVE 포트 6081|
|**CLB**|L4/L7|HTTP, HTTPS, TCP|구버전, 시험에서 제외됨|

---

### 3. ALB 상세 ⭐⭐

라우팅 방식이 3가지라 시험에 자주 나옵니다.

- **경로 기반**: `example.com/users`, `example.com/posts`
- **호스트 기반**: `one.example.com`, `other.example.com`
- **쿼리스트링 기반**: `example.com/users?id=123`

**ALB Target Groups** (대상 그룹)에 포함될 수 있는 것들: EC2 인스턴스, ECS Task, Lambda 함수, Private IP 주소

**중요 포인트**: ALB 뒤에 있는 EC2는 클라이언트의 실제 IP를 직접 볼 수 없고, `X-Forwarded-For` 헤더를 통해 확인해야 합니다.

---

### 4. NLB 상세 ⭐

- **AZ 당 고정 IP** 1개 보유 → 특정 IP를 화이트리스트할 때 유리
- Free tier 미포함
- Target Groups: EC2, Private IP, **ALB도 Target으로 지정 가능**
- Health Check 프로토콜: TCP, HTTP, HTTPS 모두 지원

---

### 5. GWLB 상세

트래픽을 방화벽 같은 보안 어플라이언스로 먼저 통과시키고 싶을 때 사용합니다. 모든 트래픽이 GWLB를 거쳐 어플라이언스로 갔다가 다시 돌아오는 구조예요.

- **GENEVE 프로토콜, 포트 6081** — 시험에 그대로 나옴
- Target Groups: EC2, Private IP

---

### 6. Cross-Zone Load Balancing ⭐

AZ를 넘어서 고르게 트래픽을 분산하는 기능입니다.

|로드 밸런서|기본값|AZ 간 요금|
|---|---|---|
|**ALB**|활성화 (기본)|무료|
|**NLB / GWLB**|비활성화|활성화 시 요금 발생|
|**CLB**|비활성화|활성화해도 무료|

---

### 7. Sticky Sessions (고정 세션) ⭐

같은 클라이언트가 항상 같은 EC2로 연결되도록 쿠키를 사용하는 기능. CLB, ALB에서 지원.

**쿠키 종류:**

- **Application-based (Custom)**: 앱이 직접 생성. `AWSALB`, `AWSALBAPP`, `AWSALBTG` 이름은 사용 불가
- **Application-based (LB 생성)**: 쿠키명 `AWSALBAPP`
- **Duration-based**: LB가 생성. ALB는 `AWSALB`, CLB는 `AWSELB`

> **주의**: 고정 세션 활성화 시 특정 인스턴스에 부하가 몰릴 수 있음

---

### 8. SSL/TLS & SNI ⭐

- 인증서는 ACM(AWS Certificate Manager)으로 관리
- **SNI**: 하나의 서버에 여러 SSL 인증서를 로드해서 여러 도메인을 동시에 처리. **ALB, NLB, CloudFront에서만 지원** (CLB 불가)
- CLB는 SSL 인증서 **1개만** 지원 → 여러 도메인이면 CLB 여러 개 필요

---

### 9. Connection Draining / Deregistration Delay ⭐

인스턴스가 내려갈 때, 진행 중인 요청(in-flight)이 완료될 때까지 기다려 주는 기능.

- CLB에서는 **Connection Draining**
- ALB/NLB에서는 **Deregistration Delay**
- 기본값 300초, 범위 1~3600초, 0으로 설정 시 비활성화
- 요청 처리 시간이 짧은 서비스라면 낮게 설정하는 게 좋음

---

### 10. Auto Scaling Group (ASG) ⭐⭐

**핵심 기능**: 부하에 따라 EC2 인스턴스 수를 자동 조절 + 비정상 인스턴스 자동 교체 + LB에 자동 등록. **ASG 자체는 무료.**

**Launch Template에 들어가는 것들**: AMI, 인스턴스 타입, User Data, EBS, 보안 그룹, SSH Key, IAM Role, 네트워크/서브넷, LB 정보

---

### 11. ASG 스케일링 정책 4가지 ⭐⭐

|정책|설명|예시|
|---|---|---|
|**Target Tracking**|특정 지표를 목표값으로 유지|평균 CPU 40% 유지|
|**Simple / Step**|CloudWatch 알람 기반으로 단계적 조정|CPU > 70% → +2대|
|**Scheduled**|예측 가능한 패턴에 시간 예약|매주 금요일 5시 +10대|
|**Predictive**|ML로 부하 예측해서 미리 스케일링|과거 패턴 기반 자동 예측|

**스케일링 지표로 좋은 것들**: `CPUUtilization`, `RequestCountPerTarget`, `Average Network In/Out`, CloudWatch 커스텀 지표

---

### 12. Scaling Cooldown (쿨다운) ⭐

스케일링 이벤트 발생 후 **300초(기본)** 동안 추가 스케일링 불가. 지표가 안정될 때까지 기다리는 시간이에요.

> **팁**: 미리 준비된 AMI를 쓰면 인스턴스 부팅 시간이 줄어서 쿨다운을 더 짧게 설정할 수 있음

---

### 시험 TIP 한 줄 정리

- "HTTP 라우팅, 마이크로서비스, 컨테이너" → **ALB**
- "고성능, 고정 IP, TCP/UDP, 화이트리스트" → **NLB**
- "방화벽/침입탐지 앞에 두고 싶다" → **GWLB**
- "여러 도메인에 SSL 하나의 LB로" → ALB or NLB (SNI 지원), CLB는 불가
- "인스턴스 수 자동 조절" → **ASG**
- "인스턴스 교체 시 기존 요청 안 끊기게" → **Connection Draining / Deregistration Delay**