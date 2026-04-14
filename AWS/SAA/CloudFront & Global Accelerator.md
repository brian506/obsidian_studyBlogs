
### 1. Amazon CloudFront

CDN(Content Delivery Network). 콘텐츠를 전 세계 **엣지 로케이션(총 216개)**에 캐싱해서 사용자 가까운 곳에서 제공. 읽기 성능 향상 + DDoS 방어(Shield, WAF 통합).

**엣지 로케이션** : 사용자에게 가장 가까이 있는 AWS의 미니 데이터 센터
- 가장 가까운 엣지 로케이션에 쏘면, AWS의 초고속 전용망을 타고 리전까지 빠르게 전달됨

---

### 2. CloudFront Origin 종류 ⭐

**S3 버킷:**

- 파일 분산 및 엣지 캐싱
- **OAC(Origin Access Control)** 로 S3 버킷 보안 강화 — OAI(Origin Access Identity)의 후속 기능
- CloudFront → S3 방향으로 데이터 업로드(Ingress)에도 사용 가능


**Custom Origin (HTTP):**

- ALB, EC2 인스턴스, S3 정적 웹사이트, 모든 HTTP 백엔드
- 로드밸런싱하여 해당 경로에 뿌려줌

**보안 설정 포인트 — 시험에 자주 나옴:**

- **ALB를 오리진으로 쓸 때**: ALB는 **퍼블릭** 필수, EC2는 **프라이빗** 가능 (ALB↔EC2 간 내부 연결이므로)
- **EC2를 직접 오리진으로 쓸 때**: EC2도 **퍼블릭** 필수. CloudFront 엣지 로케이션 IP들이 EC2 보안 그룹에 허용되어야 함

---

### 3. CloudFront vs S3 Cross-Region Replication ⭐⭐

![](../../images/스크린샷%202026-04-12%2020.33.05.png)

---

### 4. CloudFront 주요 기능

**Geo Restriction (지리적 제한):**

- **Allowlist**: 허용된 국가만 접근 가능
- **Blocklist**: 차단된 국가는 접근 불가
- 국가 판별은 서드파티 Geo-IP DB 사용
- 사용 사례: 저작권 법 준수, 콘텐츠 지역 제한

**Price Classes (요금 등급):**

- **All**: 모든 리전 사용 — 최고 성능, 최고 비용
- **200**: 대부분 리전 (가장 비싼 리전 제외)
- **100**: 가장 저렴한 리전만 (북미, 유럽)

**Cache Invalidation (캐시 무효화):** 오리진을 업데이트해도 CloudFront는 TTL 만료 전까지 모름. **CloudFront Invalidation**으로 강제 새로고침 가능. 전체(`*`) 또는 특정 경로(`/images/*`) 단위로 무효화.

---

### 5. AWS Global Accelerator ⭐⭐

공용 인터넷 대신 **AWS 내부 글로벌 네트워크**를 통해 애플리케이션으로 라우팅. 수많은 인터넷 홉(hop)으로 인한 지연 최소화.

**동작 방식:**

1. 애플리케이션에 **2개의 애니캐스트 IP** 생성
2. 클라이언트 → 가장 가까운 엣지 로케이션으로 트래픽 전송
3. 엣지 → AWS 사설 네트워크를 통해 ALB/EC2로 전달

**Anycast IP:** 모든 엣지 서버가 동일한 IP를 보유 → 클라이언트는 자동으로 가장 가까운 서버로 라우팅됨

**주요 특징:**

- 지원 대상: Elastic IP, EC2, ALB, NLB (퍼블릭/프라이빗 모두 가능)
- **IP가 변경되지 않음** → 클라이언트 캐시 문제 없음
- 헬스 체크 → 장애 시 **1분 안에** 정상 엔드포인트로 자동 전환
- **단 2개의 외부 IP**만 화이트리스트 등록 필요
- AWS Shield로 DDoS 방어

---

### 6. CloudFront vs Global Accelerator ⭐⭐⭐ (초빈출)

둘 다 AWS 글로벌 네트워크 + 엣지 로케이션 사용, Shield로 DDoS 방어 제공.

![](../../images/스크린샷%202026-04-12%2020.33.20.png)

---

### 시험 TIP 최종 정리

- **S3 오리진 보안** → **OAC** (OAI는 구버전)
- **CloudFront + ALB**: ALB 퍼블릭, EC2는 프라이빗 가능
- **CloudFront + EC2 직접**: EC2도 퍼블릭 필수
- **정적 콘텐츠 전세계 배포** → CloudFront / **실시간 동적 데이터 특정 리전** → S3 CRR
- **캐시 업데이트 즉시 반영** → **Cache Invalidation** 사용
- **고정 IP 필요** → **Global Accelerator** (CloudFront는 고정 IP 없음)
- **UDP/게임/IoT** → **Global Accelerator** (CloudFront는 HTTP만)
- **빠른 리전 Failover (1분)** → Global Accelerator