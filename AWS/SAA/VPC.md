
### 1. CIDR 기본 ⭐

- `/32` = 단 1개 IP / `/0` = 모든 IP
- VPC CIDR 범위: 최소 `/28` (16개 IP) ~ 최대 `/16` (65,536개 IP)
- 사설 IP 범위: `10.0.0.0/8`, `172.16.0.0/12`(AWS 기본 VPC), `192.168.0.0/16`

---

### 2. VPC 기본 구조 ⭐⭐

- 리전당 최대 **5개 VPC** (소프트 리밋)
- VPC당 최대 **5개 CIDR**
- 서브넷 IP 중 **5개는 AWS 예약** (첫 4개 + 마지막 1개)
    - 예: `/24` → 256 - 5 = **251개** 사용 가능

**서브넷 예약 IP 예시 (10.0.0.0/24):**

- `10.0.0.0` = 네트워크 주소
- `10.0.0.1` = VPC 라우터
- `10.0.0.2` = DNS
- `10.0.0.3` = 미래 사용을 위해 예약
- `10.0.0.255` = 브로드캐스트

---

### 3. Internet Gateway & 라우팅 ⭐⭐

- **Internet Gateway (IGW)**: VPC 안의 리소스가 인터넷과 통신하도록 함. VPC당 1개만 연결
- IGW만으로는 부족 — **서브넷 라우팅 테이블에 IGW를 목적지로 추가**해야 함
- **퍼블릭 서브넷**: 라우팅 테이블에 IGW 경로 있음
- **프라이빗 서브넷**: 인터넷 직접 연결 없음 (NAT Gateway 경유)

---

### 4. NAT Gateway ⭐⭐

프라이빗 서브넷 EC2가 인터넷에 **아웃바운드** 접근할 수 있게 해주는 서비스.

- **퍼블릭 서브넷에 배포**, Elastic IP 필요
- AWS 완전 관리형 (NAT Instance보다 고성능)
- 대역폭 5Gbps → 최대 45Gbps 자동 확장
- 보안 그룹 관리 불필요
- **단일 AZ 내에서만 복원 가능** → 고가용성 위해 **각 AZ마다 NAT Gateway 별도 생성** 필요
- 흐름: 프라이빗 EC2 → NAT Gateway(퍼블릭 서브넷) → IGW → 인터넷

**NAT Gateway vs NAT Instance:**

![](../../images/스크린샷%202026-04-13%2014.40.09.png)

---

### 5. Security Groups vs NACLs ⭐⭐⭐

![](../../images/스크린샷%202026-04-13%2014.40.19.png)

**NACL 규칙 번호:** 낮을수록 우선. 마지막에 `*`(별표) = 나머지 모두 거부. AWS는 100씩 증가 권장.

**Ephemeral Ports (임시 포트):** NACL에서 반환 트래픽 허용을 위해 아웃바운드에 임시 포트 범위(1024-65535) 열어야 함.

**트러블슈팅:**

- 인바운드 REJECT → NACL 또는 SG 중 하나가 거부
- 인바운드 ACCEPT, 아웃바운드 REJECT → **NACL 문제** (SG는 stateful)

---

### 6. VPC Flow Logs ⭐

- VPC/서브넷/ENI 레벨에서 IP 트래픽 기록
- 전송 대상: S3, CloudWatch Logs
- 주요 필드: `srcaddr`, `dstaddr`, `srcport`, `dstport`, `action`(ACCEPT/REJECT)
- ELB, RDS, ElastiCache, Redshift, NAT GW, Transit GW 포함 AWS 관리 인터페이스도 캡처
- 활용: S3 + Athena(SQL 분석), CloudWatch + Contributor Insights(Top 10 IP)

---

### 7. VPC Peering ⭐⭐

두 VPC를 AWS 내부 네트워크로 프라이빗 연결.

- CIDR 범위가 겹치면 안 됨
- **전이(Transitive) 안 됨** — A↔B, B↔C라고 A↔C 되지 않음. 직접 피어링 설정 필요
- **각 VPC 서브넷의 라우팅 테이블 업데이트** 필요
- 다른 계정/리전 간에도 가능
- 피어링된 VPC의 보안 그룹 참조 가능 (동일 리전)

---

### 8. VPC Endpoints ⭐⭐

VPC 내에서 AWS 서비스에 **인터넷 없이** 접근.

|유형|특징|지원 서비스|비용|
|---|---|---|---|
|**Gateway Endpoint**|라우팅 테이블에 추가, 보안 그룹 없음|**S3, DynamoDB만**|**무료**|
|**Interface Endpoint**|ENI 프로비저닝, 보안 그룹 필요|대부분 AWS 서비스|유료 (시간당 + GB당)|

> **시험 TIP**: "S3를 프라이빗 네트워크로 접근" → **Gateway Endpoint** (무료이므로 우선 선택) Interface Endpoint는 온프레미스(VPN/Direct Connect), 다른 VPC/리전에서 접근할 때 선호

---

### 9. Site-to-Site VPN ⭐⭐

온프레미스 데이터 센터 ↔ AWS VPC를 **퍼블릭 인터넷을 통해 암호화**하여 연결.

- **VGW (Virtual Private Gateway)**: AWS VPC 측에 연결
- **CGW (Customer Gateway)**: 온프레미스 측 장치
- CGW가 NAT 뒤에 있으면 **NAT-T(NAT Traversal)** 활성화 필요
- 라우팅 테이블에 VGW 경로 추가 + **Route Propagation** 활성화

**CloudHub:** 여러 CGW(지점)를 하나의 VGW에 연결 → 지점 간 통신 허용 (hub-and-spoke)

---

### 10. Direct Connect (DX) ⭐⭐

온프레미스 ↔ AWS 전용 **물리 전용선** 연결. 퍼블릭 인터넷 미경유.

- 사용 사례: 대용량 데이터, 실시간 데이터, 하이브리드 환경, 비용 절감
- 구축에 시간이 걸림 (1개월 이상)
- 암호화는 기본 없음 → IPSec VPN over Direct Connect로 추가 가능

**Direct Connect Gateway:** 단일 DX 연결로 **여러 리전의 여러 VPC**에 연결

**연결 유형:**

- **Dedicated**: 1/10/100Gbps. 물리 전용 포트. AWS → Direct Connect 파트너 처리
- **Hosted**: 50Mbps~10Gbps. 파트너를 통해 용량 추가/제거 가능. 유연성 높음

**Resiliency (복원력):**

- **High Resiliency**: 각 로케이션에 DX 연결 1개씩
- **Maximum Resiliency**: 각 로케이션에 DX 연결 2개씩 (권장)
- **DX 장애 대비 백업**: Site-to-Site VPN을 DX와 병렬로 설정

---

### 11. Transit Gateway ⭐⭐

수천 개의 VPC + 온프레미스를 **허브 앤 스포크(별 모양)** 구조로 전이적 연결.

- VPC Peering의 확장판 (Transitive 연결 지원)
- 리전 리소스. RAM으로 계정 간 공유 가능
- 라우팅 테이블로 VPC 간 통신 제어
- Direct Connect Gateway, Site-to-Site VPN과 연동
- **AWS에서 유일하게 IP 멀티캐스트 지원**
- **ECMP**: 여러 Site-to-Site VPN 연결로 대역폭 증가

---

### 12. 네트워킹 비용 절감 ⭐

- **같은 AZ 내 통신**: 무료 (사설 IP 사용)
- **다른 AZ 통신**: 사설 IP = $0.01/GB, 공용 IP = $0.02/GB
- **다른 리전 통신**: $0.02/GB
- **비용 절감 원칙**: 사설 IP 사용, 같은 AZ에 리소스 배치 (단, 가용성 저하 트레이드오프)
- **Egress 비용 최소화**: 직접 인터넷 접속보다 Direct Connect 사용, S3/CloudFront 게이트웨이 활용

---

### 13. 기타 VPC 기능

- **AWS PrivateLink**: 서비스를 수백/수천 개의 고객 VPC에 비공개로 노출. NLB + ENI 사용. VPC Peering/NAT GW/라우팅 테이블 불필요
- **Traffic Mirroring**: ENI 트래픽 복사 → 보안 분석
- **Egress-only Internet Gateway**: IPv6 전용 NAT Gateway 역할 (아웃바운드만)

---

### 시험 TIP 최종 정리 ⭐⭐⭐

- **VPC 서브넷 5개 IP 예약** → `/27` = 32-5 = 27개 사용 가능
- **NAT Gateway = 퍼블릭 서브넷에 배포**, Elastic IP 필요
- **고가용성 NAT Gateway** = **AZ마다 하나씩** 생성
- **NACL = Stateless** (인/아웃 별도 설정) / **SG = Stateful** (자동 응답)
- **VPC Peering = 전이 안 됨** (A-B-C ≠ A-C)
- **S3/DynamoDB 프라이빗 접근** = **Gateway Endpoint (무료)**
- **나머지 AWS 서비스** = Interface Endpoint (유료)
- **인터넷 암호화 연결** = **Site-to-Site VPN**
- **전용 물리선** = **Direct Connect** (설치 시간 필요, 암호화 기본 없음)
- **수천 VPC 연결** = **Transit Gateway** (멀티캐스트 유일 지원)
- **다른 AZ 통신 비용 절감** = 사설 IP 사용