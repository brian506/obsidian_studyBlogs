### 1. 암호화 기본 개념 ⭐

- **전송 중 암호화 (In-flight / SSL)**: HTTPS, SSL 인증서 사용. 중간자 공격(MITM) 방지
- **저장 데이터 암호화 (Server-side at rest)**: 서버 수신 후 암호화, 전송 전 복호화. 키는 KMS 등에서 관리
- **클라이언트 측 암호화 (Client-side)**: 서버는 절대 복호화 불가. 수신 클라이언트만 복호화. **봉투 암호화(Envelope Encryption)** 활용

---

### 2. AWS KMS ⭐⭐⭐

AWS에서 "암호화" = 대부분 KMS. IAM과 완전 통합, CloudTrail로 모든 키 사용 API 감사 가능.

**KMS 키 종류:**

|유형|키 이름|비용|관리|
|---|---|---|---|
|**AWS 관리 키**|`aws/서비스명` (예: `aws/s3`)|무료|AWS가 자동 관리|
|**고객 관리 키 (CMK)**|사용자 정의|$1/월|사용자 직접 관리|
|**고객 제공 키**|외부 가져오기|$1/월|사용자가 키 제공|

**대칭 vs 비대칭 키:**

- **대칭(AES-256)**: AWS 서비스 대부분 사용. 키 자체에 직접 액세스 불가, KMS API 호출 필요
- **비대칭(RSA, ECC)**: 퍼블릭 키 다운로드 가능, 프라이빗 키는 불가. AWS 외부에서 암호화 후 AWS에서 복호화할 때 사용

**KMS 자동 교체:**

- AWS 관리 키: 1년마다 자동 교체
- 고객 관리 키: 1년마다 자동 교체 활성화 가능 (선택)
- 가져온 키: 수동 교체만 가능

**KMS Multi-Region Keys ⭐:**

- 동일한 키 ID + 키 구성 요소를 여러 리전에 복제
- 한 리전에서 암호화 → 다른 리전에서 복호화 가능 (재암호화 불필요)
- **전역 단일 키가 아님** — 각 복제본은 독립적으로 관리
- 사용 사례: DynamoDB Global Tables의 클라이언트 측 암호화, Global Aurora 특정 필드 보호 (DB 관리자로부터도 보호)

**S3 복제 + 암호화 ⭐:**

- 기본 복제: 암호화 없는 객체 + **SSE-S3** 암호화 객체
- **SSE-KMS**: 별도 옵션 활성화 필요. 대상 KMS 키 지정, IAM Role 설정 필요
- **SSE-C**: **절대 복제 불가**

---

### 3. SSM Parameter Store ⭐⭐

구성값과 암호를 안전하게 저장하는 **서버리스** 서비스.

- KMS로 선택적 암호화 가능
- 버전 추적, IAM 보안, EventBridge 알림, CloudFormation 통합
- **계층적 경로 구조** (`/my-app/dev/db-password`) → IAM 정책으로 경로별 접근 제어

**Standard vs Advanced:**

![](../../images/스크린샷%202026-04-13%2014.37.22.png)

> **시험 TIP**: SSM Parameter Store vs Secrets Manager — 자동 암호 교체 필요 → **Secrets Manager**

---

### 4. AWS Secrets Manager ⭐⭐

- **암호 저장 + 자동 교체** 기능 특화 (X일마다 Lambda로 자동 교체)
- **RDS, Aurora, MySQL, PostgreSQL과 통합** ← 핵심 키워드
- KMS로 암호화
- **Multi-Region Secrets**: 여러 리전에 읽기 전용 복제본 동기화 → 재해 복구, 멀티 리전 DB에 유용

---

### 5. AWS Certificate Manager (ACM) ⭐

- TLS/SSL 인증서 프로비저닝 및 관리
- **퍼블릭 인증서 무료**, 자동 갱신
- 통합 가능: ALB, NLB, CloudFront, API Gateway
- **EC2에는 사용 불가** (인증서 추출 불가)

---

### 6. AWS WAF ⭐⭐

**7계층(HTTP) 웹 애플리케이션 방화벽**

배포 가능 대상: **ALB, API Gateway, CloudFront, AppSync GraphQL, Cognito User Pool**

**Web ACL 규칙 유형:**

- **IP Set**: 최대 10,000개 IP 차단/허용
- HTTP 헤더/본문/URI: **SQL Injection, XSS** 방어
- **Geo-match**: 특정 국가 차단
- **Rate-based rules**: DDoS 방어 (초당 요청 수 제한)
- 규칙 그룹(Rule Group): 재사용 가능한 규칙 집합

**WAF + ALB + 고정 IP:** ALB는 고정 IP 없음 → **Global Accelerator**(고정 IP 제공) + ALB에 WAF 적용하는 조합 사용

CloudFront: 전역 적용 / 나머지: 리전별 적용

---

### 7. AWS Shield ⭐

|버전|비용|보호 범위|
|---|---|---|
|**Standard**|**무료, 자동 활성화**|3/4계층 공격 (SYN Flood, UDP Flood, 반사 공격)|
|**Advanced**|**$3,000/월/조직**|EC2, ELB, CloudFront, Global Accelerator, Route 53 + DRP 24/7 대기, 비용 급증 방지, WAF 자동 규칙 배포|

---

### 8. AWS Firewall Manager ⭐

**AWS 조직 전체 계정의 방화벽 규칙을 중앙 관리**
여러 계정이나 여러 VPC에 있는 Network Firewall, WAF, Security Group 규칙들을 **중앙에서 한꺼번에 관리**할 때 쓴다.

- WAF 규칙, Shield Advanced, 보안 그룹, Network Firewall, Route 53 DNS Firewall
- **새 리소스 생성 시 자동으로 규칙 적용** ← 핵심

**WAF vs Shield vs Firewall Manager:**

- **WAF**: 세분화된 웹 ACL 규칙 (단일 서비스)
- **Shield Advanced**: DDoS 방어, 24/7 DRP
- **Firewall Manager**: 여러 계정/리소스에 WAF+Shield 규칙 일괄 배포 및 관리

---

### 시험 TIP 최종 정리

- **AWS에서 암호화** = 대부분 **KMS**
- **암호 자동 교체 + RDS 통합** → **Secrets Manager**
- **구성값 + 암호 저장 (교체 불필요)** → **SSM Parameter Store**
- **SSE-C 객체는 S3 복제 불가**
- **SSE-KMS S3 복제** → 별도 옵션 활성화 + IAM Role 필요
- **EC2에 ACM 사용 불가**
- **WAF 배포 가능**: ALB, API Gateway, CloudFront, Cognito
- **WAF + ALB 고정 IP** → **Global Accelerator** + ALB 조합
- **Shield Standard**: 무료, 자동 / **Shield Advanced**: $3,000/월, DRP 포함
- **Firewall Manager**: 조직 전체 규칙 일괄 관리, 새 리소스 자동 적용