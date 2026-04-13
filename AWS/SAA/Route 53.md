### 1. Route 53 기본

- AWS에서 **100% SLA**를 제공하는 **유일한 서비스**
- 완전 관리형 **권위 있는(Authoritative) DNS** — 고객이 직접 DNS 레코드 수정 가능
- Domain Registrar 역할도 겸함 (도메인 직접 구매 가능)
- 포트 **53**번이 전통적인 DNS 포트라서 Route 53

---

### 2. 레코드 타입 ⭐

|타입|역할|
|---|---|
|**A**|호스트명 → IPv4|
|**AAAA**|호스트명 → IPv6|
|**CNAME**|호스트명 → 다른 호스트명 (루트 도메인 불가)|
|**NS**|호스팅 존의 네임서버. 도메인 트래픽 라우팅 제어|

---

### 3. Hosted Zones (호스팅 영역)

DNS 레코드를 담는 컨테이너. **월 $0.50/개**

- **Public Hosted Zone**: 인터넷 트래픽 라우팅. `application1.mypublicdomain.com`
- **Private Hosted Zone**: VPC 내부 트래픽 라우팅. `application1.company.internal`

---

### 4. TTL ⭐

클라이언트가 DNS 결과를 캐싱하는 시간.

- **High TTL** (예: 24시간): Route 53 트래픽 적음, 변경 반영 느림
- **Low TTL** (예: 60초): Route 53 트래픽 많음(비용↑), 변경 반영 빠름
- **Alias 레코드는 TTL 설정 불가** — Route 53이 자동 관리

> **실전 팁**: 레코드를 변경할 계획이면 미리 TTL을 낮춰두고, 변경 후 다시 높이는 게 좋음

---

### 5. CNAME vs Alias ⭐⭐⭐ (초빈출)

![](../../images/스크린샷%202026-04-12%2020.26.42.png)

**Alias 대상이 될 수 있는 AWS 리소스들:** ELB, CloudFront, API Gateway, Elastic Beanstalk, S3 Website, VPC Interface Endpoint, Global Accelerator, 같은 Hosted Zone의 Route 53 레코드

> ⚠️ **EC2의 DNS 이름에는 Alias 레코드 설정 불가!** 시험에 그대로 나옵니다.

---

### 6. 라우팅 정책 8가지 ⭐⭐⭐ (이 챕터 가장 중요)

**중요 전제**: Route 53은 트래픽을 직접 라우팅하는 게 아니라 **DNS 쿼리에 응답**하는 것. 실제 트래픽은 클라이언트가 받은 IP로 직접 감.

|정책|동작 방식|헬스 체크|핵심 키워드|
|---|---|---|---|
|**Simple**|단일 리소스로 보냄. 여러 값 반환 시 **클라이언트가 랜덤 선택**|❌|단순, 기본|
|**Weighted**|가중치 비율대로 분산 (합이 100 불필요)|✅|A/B 테스트, 비율 분산|
|**Latency**|사용자 기준 **지연 시간 가장 짧은** 리전으로|✅|빠른 응답|
|**Failover**|Primary 장애 시 Secondary로 (Active-Passive)|✅ (필수)|재해복구|
|**Geolocation**|사용자 **실제 위치** 기반 라우팅|✅|국가/대륙별|
|**Geoproximity**|위치 + **bias(편향값)**로 트래픽 조절|-|Traffic Flow 필요|
|**IP-based**|클라이언트 **IP/CIDR** 기반 라우팅|-|ISP별, 네트워크 비용 절감|
|**Multi-Value**|여러 정상 리소스 최대 **8개** 반환|✅|ELB 대체 아님|

---

**각 정책 핵심 포인트:**

**Weighted**: 가중치 0 = 트래픽 차단. 모두 0이면 모두 동일하게 반환.

**Geolocation vs Latency 차이** — 시험에 자주 나오는 함정:

- Geolocation: 사용자의 **물리적 위치** 기반 (한국 사용자 → 항상 한국 서버)
- Latency: 사용자 기준 **실제 응답 속도** 기반 (한국 사용자도 미국이 더 빠르면 미국으로)

**Geoproximity**: Geolocation과 달리 **bias 값으로 트래픽 경계를 인위적으로 조절** 가능. 반드시 **Route 53 Traffic Flow** 기능 사용 필요.

**Failover**: Primary에 반드시 헬스 체크 연결. Primary 비정상 → Secondary(Passive)로 자동 전환.

**Multi-Value**: 헬스 체크 통과한 리소스만 반환. 하지만 **ELB를 대체하는 건 아님** — DNS 레벨의 간단한 부하분산일 뿐.

---

### 7. Health Checks ⭐⭐

**3가지 종류:**

1. **엔드포인트 모니터링**: HTTP/HTTPS/TCP로 직접 체크
2. **Calculated Health Check**: 여러 헬스 체크를 AND/OR/NOT으로 조합. 최대 256개 하위 체크
3. **CloudWatch Alarm 기반**: 프라이빗 리소스 모니터링에 사용

**엔드포인트 모니터링 세부 사항:**

- 전 세계 **약 15개** 헬스 체커가 동시에 확인
- **18% 이상**이 정상이라고 판단하면 Route 53도 정상 간주
- 기본 간격 **30초** (10초로 단축 가능, 비용 추가)
- 2xx / 3xx 응답 코드 = 정상
- 응답 처음 **5120바이트** 내 텍스트로 통과/실패 판단 가능

**Private Hosted Zone의 헬스 체크 문제:** Route 53 헬스 체커는 VPC 외부에 있어서 프라이빗 리소스에 직접 접근 불가. 해결책 → **CloudWatch 메트릭 + 경보 생성 → 해당 경보를 모니터링하는 헬스 체크 생성**.

---

### 8. Domain Registrar vs DNS Service

- **Domain Registrar**: 도메인을 구매/등록하는 곳 (GoDaddy, Route 53 등). 연간 비용 발생
- **DNS Service**: DNS 레코드를 관리하는 곳
- 둘은 같은 서비스일 필요 없음 — GoDaddy에서 도메인 구매 후 Route 53으로 DNS 관리 가능

**3rd Party 도메인 + Route 53 DNS 사용하는 방법:**

1. Route 53에서 Public Hosted Zone 생성
2. 3rd party 사이트에서 **NS 레코드를 Route 53 네임서버로 업데이트**

---

### 시험 TIP 최종 정리

- Route 53 = AWS 유일 **100% SLA** 서비스
- **루트 도메인(`example.com`)에 CNAME 불가** → Alias 사용
- **EC2에는 Alias 레코드 불가**
- Geolocation ≠ Latency: 위치 기반 vs 속도 기반
- Geoproximity는 반드시 **Traffic Flow** + **bias** 키워드 세트로 기억
- 프라이빗 리소스 헬스 체크 → **CloudWatch Alarm 경유**
- Multi-Value는 ELB 대체제가 **아님**