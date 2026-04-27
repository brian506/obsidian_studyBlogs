### 1. Docker 기초

컨테이너에 앱을 패키지화해서 어디서든 동일하게 실행하는 플랫폼. VM보다 가벼움 (호스트 OS 공유).

**이미지 저장소:**

- **Docker Hub**: 퍼블릭 리포지토리
- **Amazon ECR**: AWS 프라이빗 리포지토리

---

### 2. AWS 컨테이너 관리 옵션

| 서비스         | 특징                            |
| ----------- | ----------------------------- |
| **ECS**     | Amazon 전용 컨테이너 플랫폼            |
| **EKS**     | Kubernetes 관리형 버전 (오픈소스)      |
| **Fargate** | 서버리스 컨테이너 실행 (ECS/EKS와 함께 사용) |
| **ECR**     | 컨테이너 이미지 저장소                  |

---

### 3. Amazon ECS ⭐⭐

**두 가지 시작 유형:**

**EC2 Launch Type:**

- EC2 인스턴스를 직접 프로비저닝 및 관리해야 함
- 각 EC2 인스턴스에 **ECS Agent**가 실행되어야 함
- 비용: EC2 인스턴스 비용 발생

**Fargate Launch Type:**

- **서버리스** — EC2 인스턴스 직접 관리 불필요
- Task Definition만 생성하면 AWS가 CPU/RAM에 맞게 자동 실행
- 확장 시 태스크 수만 늘리면 됨 → 훨씬 간단

---

### 4. IAM Roles for ECS ⭐⭐

**두 가지 역할을 명확히 구분해야 함 — 시험 빈출:**

**EC2 Instance Profile** (EC2 Launch Type 전용):

- ECS 에이전트가 사용
- ECS API 호출, CloudWatch Logs 전송, **ECR에서 이미지 Pull**, Secrets Manager/SSM 참조

**ECS Task Role:**

- 각 태스크마다 개별 역할 할당 가능
- 태스크가 S3, SQS 등 다른 AWS 서비스를 호출할 때 사용
- **Task Definition에서 정의**

> **시험 TIP**: EC2 Instance Profile = ECS 인프라 운영용 / Task Role = 실제 앱이 AWS 서비스 호출용

---

### 5. Load Balancer 통합 ⭐

- **ALB**: 대부분의 사용 사례에 적합, ECS와 잘 통합됨
- **NLB**: 고처리량 또는 AWS PrivateLink와 함께 사용할 때 권장
- CLB는 지원은 되지만 권장하지 않음 (Fargate 미지원)

---

### 6. ECS Data Volumes — EFS ⭐

- EC2 + Fargate Launch Type 모두 EFS 마운트 가능
- 멀티 AZ에 걸쳐 **영속적인 공유 스토리지** 제공
- Fargate + EFS = **완전 서버리스 스토리지**
- S3는 ECS 파일 시스템으로 마운트 **불가**

---

### 7. ECS Service Auto Scaling ⭐

- CPU, 메모리, ALB 요청 수 기반으로 자동 스케일링
- **Target Tracking**: 특정 CloudWatch 메트릭 목표값 유지
- **Step Scaling**: CloudWatch Alarm 기반 단계적 조정
- **Scheduled Scaling**: 예약 기반

EC2 Launch Type 사용 시 EC2 인스턴스 수도 함께 스케일링 필요:

- **Auto Scaling Group**: CPU 사용률 기반
- **ECS Cluster Capacity Provider**: 자동으로 EC2 추가/제거 (권장)

---

### 8. Amazon ECR ⭐

**Elastic Container Registry** — Docker 이미지 저장소.

- **Private**: AWS 계정 전용
- **Public**: Amazon ECR Public Gallery
- ECS와 완전 통합, IAM으로 접근 제어
- 이미지 취약점 스캐닝, 버전 관리, 태그, 수명 주기 관리 지원

---

### 9. Amazon EKS ⭐⭐

**Elastic Kubernetes Service** — AWS에서 Kubernetes 클러스터를 관리형으로 제공.

- ECS 대안. 오픈소스 Kubernetes 사용 원할 때 또는 이미 Kubernetes 쓰는 팀
- ECS와 목적은 비슷하지만 API가 다름
- **멀티 클라우드** 환경에서도 Kubernetes를 쓰기 때문에 선택하기도 함

**EKS 노드 유형:**

- **Managed Node Groups**: EC2 노드를 AWS가 관리 (ASG 사용). On-Demand/Spot 인스턴스 지원
- **Self-Managed Nodes**: 직접 EC2 노드 생성 및 등록. 더 많은 제어 가능
- **Fargate Mode**: 노드 관리 불필요 (완전 서버리스)

**스토리지:** EBS, EFS, FSx for Lustre/NetApp ONTAP 지원. StorageClass + CSI 드라이버 사용.

---

### 10. ECS vs EKS 비교

![](../../images/스크린샷%202026-04-12%2020.39.19.png)

---

### 시험 TIP 최종 정리

- **서버리스 컨테이너** → **Fargate** (EC2 관리 불필요)
- **EC2 Instance Profile** = ECS 에이전트용 / **Task Role** = 앱이 AWS 호출용
- **ECR** = Docker 이미지 저장소 (IAM으로 접근 제어)
- **Fargate + EFS** = 완전 서버리스 + 공유 영속 스토리지
- **EKS** = Kubernetes 관리형, 멀티클라우드/오픈소스 원할 때
- "ECS 태스크가 S3에 접근해야 한다" → **Task Role** 부여