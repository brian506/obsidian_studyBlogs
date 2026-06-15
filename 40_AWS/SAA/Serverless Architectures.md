### 1. 마이크로서비스 아키텍처 ⭐

각 서비스가 **독립적으로 확장**하고 각자 코드 리포지토리를 가짐. REST API로 서로 통신.

**통신 패턴 두 가지:**

- **동기적 (Synchronous)**: API Gateway, 로드 밸런서 — 요청에 즉시 응답이 필요한 경우
- **비동기적 (Asynchronous)**: SQS, Kinesis, SNS, Lambda 트리거 — 응답 시점/내용을 신경 쓰지 않는 경우

**마이크로서비스 아키텍처 예시:**

```
service 1: ELB → ECS → DynamoDB (Route 53으로 DNS 라우팅)
service 2: API Gateway → Lambda → ElastiCache (서버리스)
service 3: ELB → EC2 ASG → RDS (전통적 아키텍처)
```

서비스끼리 서로 호출 가능. 예를 들어 service 2의 Lambda가 ELB를 통해 service 1을 호출해서 응답 생성.

**마이크로서비스의 문제점:**

- 서비스 늘어날 때마다 운영 오버헤드 발생
- 서버 사용률 최적화 어려움
- 여러 버전 동시 운영 복잡성
- 클라이언트 통합 코드 증가

**서버리스로 일부 해결 가능:**

- API Gateway + Lambda는 자동 확장 + 사용량 기반 과금 → 서버 걱정 없음
- API Gateway에서 환경/버전 쉽게 복제
- Swagger 통합으로 클라이언트 SDK 자동 생성 가능

---

### 2. Software Updates Offloading ⭐⭐

**문제 상황:** EC2 + ELB + ASG로 운영 중인 앱이 소프트웨어 업데이트를 배포할 때마다 요청이 폭증 → EC2 비용, 네트워크 비용 급증. 업데이트 파일은 EFS에 저장되어 있음.

**해결책: CloudFront를 앞에 배치.**

기존 아키텍처 변경 없이 CloudFront만 추가하면 됨.

- 소프트웨어 업데이트 파일은 **정적 파일** (한 번 배포되면 변하지 않음)
- 엣지 로케이션에 캐싱되면 **EC2까지 요청이 오지 않음**
- ASG가 많이 확장할 필요 없어짐 → **EC2, 네트워크, EFS 비용 대폭 절감**
- EC2는 서버리스가 아니지만 CloudFront는 서버리스이므로 자동 확장

> **시험 TIP 핵심**: "기존 아키텍처를 변경하지 않고 비용을 줄이고 전 세계 확장" → **CloudFront 앞에 추가**. 정적 콘텐츠(업데이트 파일, 이미지 등)가 있으면 CloudFront 캐싱으로 비용+성능 모두 해결.

---

### 시험 TIP 정리

- **마이크로서비스 간 실시간 응답 필요** → 동기 패턴 (API Gateway, ELB)
- **마이크로서비스 간 느슨한 결합** → 비동기 패턴 (SQS, SNS, Kinesis)
- **정적 콘텐츠 + 전 세계 배포 + 비용 절감** → **CloudFront** (기존 앱 변경 불필요)
- **서버리스 마이크로서비스 운영 오버헤드 감소** → API Gateway + Lambda 조합