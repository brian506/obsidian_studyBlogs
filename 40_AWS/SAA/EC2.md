> 클라우드 가상 서버 (Elastic Compute Cloud) - IaaS

### 구성
- **OS** : Linux, Window or Mac
- **CPU** : 가상 머신에 사용할 컴퓨팅 성능과 코어 개수
- **RAM** : RAM 크기
- **Storage Space** :
	- Network - attached (EBS & EFS)
	- Hardware (EC2 Instance Store)
- Network card: Public IP address
- Firewall rules: **security group**
- Bootstrap script (configuration at first launch): EC2 User Data

### EC2 User Data
> EC2 인스턴스에 부팅될 때 실행되는 스크립트 또는 데이터를 의미.
> 주로 인스턴스의 초기 설정을 자동화하는 데 사용

- EC2 User Data Script를 사용하여 인스턴스를 Bootstrap(머신이 작동될 때 명령을 시작하는 것) 할 수 있음
- 스크립트는 처음에만 한 번 실행되고 다시 실행되지 않음
- 루트 계정에서 실행
- 민감한 정보 포함X

### EC2 Instance Types
- General Purpose (범용)
- Compute Optimized (컴퓨팅 최적화)
- Memory Optimized (메모리 최적화)
- Accelerated Computing (가속화된 컴퓨팅)
- Storage Optimized (스토리지 최적화)
- Instance Features (인스턴스 기능)
- Measuring Instance Performance (인스턴스 성능 측정)

> - > m5.2xlarge  
    >    m: instance class  
    >    5: generation (AWS improves them over time)  
    >    2xlarge: size within the instance class

#### General Purpose (범용)
- 앱 서버나 코드 레파지토리 같은 다양한 작업에 적합
- 컴퓨팅, 메모리, 네트워크 간의 균형 굿

#### Compute Optimized (컴퓨팅 최적화)
- 고성능 프로세서의 활용
	- Batch processing workloads
    - Media transcoding
    - High performace web server
    - **Hight performance computing (HPC)**
    - Scientific modeling & machine learning
    - Dedicated gaming servers

#### Memory Optimized (메모리 최적화)
- 메모리에서 대규모 데이터 세트를 처리하는 워크로드를 위한 빠른 성능 제공
- 사례 
	- High performance, relational/non-relational databases
	- Distributed web scale cache stores
	- **In-memory databases optimized for BI (business intelligence)**
	- Applications performing real-time processing of big unstrutured data

#### Storage Optimized (스토리지 최적화)
- 로컬 스토리지에서 **매우 큰 데이터 세트**에 대해 많은 순차적 읽기 및 쓰기 액세스를 요구하는 워크로드를 위해 설계됨.
- 사용 사례
    - High frequency online transcation processing (OLTP) systems
    - Relational & NoSQL databases
    - Cache for in-memory databases (ex: Redis)
    - Data warehousing applications
    - Distributed file systems

## Security Group

**특징**
- 특정 지역과 VPC(Virtual Private Cloud)에 제한되도록 설정되어 있음. 따라서 지역을 전환하면 새 보안 그룹을 생성하거나 다른 VPC를 생성해야 함.
- EC2 "외부"에 있음. 따라서 트래픽이 차단되면 EC2 인스턴스는 확인할 수 없음.
- **SSH 액세스를 위해 하나의 별도 보안 그룹을 유지하는 것이 좋음**. 보통 SSH 액세스는 가장 복잡하므로 별도 보안 그룹이 잘 완료됐는지 확인해야 함.
- **타임 아웃**으로 인해 애플리케이션에 접근할 수 없을 때에는 **보안 그룹의 문제**임.
- "connection refused" 에러인 경우엔 보안 그룹은 실행되었고 트래픽은 통과되었지만, 애플리케이션에 문제가 생겼거나 실행되지 않는 등의 문제가 발생된 것임.
- 기본적으로 모든 인바운드 트래픽은 차단되어 있음
- 모든 아웃바운드 트래픽은 허용됨.

## EC2 Instance Purchase

### EC2 On Demand
> 사용한만큼 지불하는 방식

- 비용이 가장 많이 듬
- 장기적인 약정 필요 없음
- **단기적이고 중단이 없는 워크로드**가 필요할 때 사용
- 애플리케이션의 거등을 예측할 수 없을 때 사용

### EC2 Reserverd Instance
> 특정 인스턴스를 예약하고 사용하는 방식
>  On-demand에 비해 최대 72% 할인 제공

- 예약 주기 : 1년 또는 3년
- 후불, 부분, 전액 결제 방식
- 사용량이 일정한 애플리케이션에 예약 인스턴스를 사용하는 것이 좋음(ex.DB)

### EC2 Spot Instances
> 끊길수도 있는 인스턴스
> On-demand 에 비해 최대 90% 할인율

- **가장 비용적으로 효율적인 방식**
- 복구 가능한 워크로드에 효율적
	- 배치 작업
	- 데이터 분석
	- 이미지 프로세싱
	- 시작,종료 시간이 정해져 있지 않은 작업
- 끊기면 안되는 인스턴스에 적용하면 안됨 (ex.DB)

### EC2 Saving Plans
> 장기간 사용 시 할인
> 1년 또는 3년 동안 시간당 10달러

- 사용량이 한도 넘으면 On-demand 가격으로 청구
- 특정 Instance Family & region으로 고정

### EC2 Dedicated Hosts
> 사용자 전용으로 할당된 인스턴스

- 가장 비싼 인스턴스 방식
- 복잡한 라이센싱 모델을 자긴 소프트웨어 또는 강력한 규제와 규정 준수 요구사항을 가진 회사에 유용

### EC2 Dedicated Instances
> 사용자 전용 하드웨어에서 인스턴스 실행

- 동일한 계정 내 다른 인스턴스와 하드웨어를 공유 가능
- 인스턴스 배치에 대한 제어가 없음

Dedicated Instance는 사용자만의 인스턴스를 **자신만의 하드웨어**에 갖는 반면, Dedicated Host는 **물리적 서버** 자체에 접근권을 갖고, 낮은 수준의 하드웨어에 대한 가시성을 제공한다.

### EC2 Capacity Reservation
> 지정된 특정 AZ에서 필요한 기간 동안 On-demand 인스턴스 용량 예약 가능

- 필요할 때 언제든지 EC2 용량에 엑세스 가능
- 약정 기간 없고, 요금 할인 없음
- 특정 AZ에서 실행해야 하는 단기적이고 중단되지 않는 워크로드에 적합

리조트에 비유해서,

- **On demand**: 원하는 때에 리조트에 들어가서 전액 지불.
- **Reserved**: 미리 계획을 세우고 오랜 시간 머물 계획이 있는 경우 높은 할인을 받을 수 있음.
- **Savings Plans**: 일정 기간 동안 매시간 일정 금액을 지불하고 원하는 방 유형(킹, 스위트, 오션뷰 등)에 머물 수 있음.
- **Spot instances**: 리조트에서 빈 방을 경매로 판매하고, 가장 높은 입찰가를 제시한 사람이 방을 얻음. 언제든지 방에서 쫓겨날 수 있음.
- **Dedicated Hosts**: 리조트 전체 건물을 예약함.
- **Capacity Reservations**: 일정 기간동안 방을 예약하고, 체류할 지 안 할지 확실하지 않다고 말하는 것과 같음. 하지만 체류하지 않더라도 가격을 지불해야함.

## Private vs Public IP 

**Private IP**
- 기기가 프라이빗 네트워크에서만 식별
- IP는 프라이빗 네트워크 안에서만 고유하면 됨
	- 두 개의 다른 회사는 같은 프라이빗 IP를 가질 수 있음
- 기기가 프라이빗 네트워크에 있을 때에는, NAT와 IGW를 사용하여 인터넷에 연결

**Public IP**
- 인터넷 상에서 기기 식별됨
- 전체 웹에서 고유해야함
- 지리적 위치를 쉽게 식별 가능

**Elastic IPs**
- EC2 인스턴스를 중지하고 다시 시작하면, 퍼블릭 IP 가 변경되므로, 바뀌지 않는 고정된 IP 가 필요할 떄 탄력적 IP 사용
- 한 번에 하나의 인스턴스에만 연결 가능
- 일반적으로 탄력적IP 사용을 피하는 것이 좋음
	- 임의의 퍼블릭 IP를 써서 DNS 이름을 할당하는 것이 좋음
	- 또는 로드 밸런서를 사용하여 퍼블릭IP를 사용안할 수 있음

