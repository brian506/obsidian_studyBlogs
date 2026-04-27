현재 EC2 위에 띄워진 Redis, Prometheus TSDB, Loki 로그, Grafana 설정, Spring 로그, SSL 인증서 등 영속 데이터가 EC2 로컬 디스크에 있어 Task 재시작 시 데이터 유실 위험이 있다.
EFS를 붙여 각 컨테이너의 영속 볼륨을 NFS 로 분리하는 것이 목적이다.

**NFS(Network File System)** : 네트워크를 통해 다른 컴퓨터의 파일을 내 파일처럼 사용할 수 있게 하는 프로토콜이다.
- 서버와 클라이언트가 NFS 규격에 맞춰 데이터를 주고 받음

## EFS(Amazon Elastic File System)
> 클라우드 기반의 완전 관리형 파일 저장소 서비스

- 내부적으로 NFS 프로토콜 사용
- 사용자는 NFS 방식으로 EFS에 접속
- 데이터 양에 따라 자동으로 크기가 늘어나고 줄어듬
- EC2에서 일반 디렉토리처럼 mount해서 사용
- 여러 EC2/Task가 동시에 같은 파일시스템을 공유할 수 있음
비용 : 저장된만큼만 과금 (0.3달러 / 월, 4.5달러 / 15GB )

### Access Point?
> EFS 안의 특정 디렉토리를 격리해서 마운트하는 진입점

- 서비스별로 /redis, /loki 등 분리 마운트 가능
- UID/GID 설정으로 컨테이너 권한 충돌 방지

### Mount Target?
> EFS를 특정 VPC 서브넷에서 접근할 수 있게 해주는 네트워크 엔드포인트

- EC2와 같은 서브넷(AZ)에 마운트 타켓이 있어야 연결 가능

## EFS 적용하기

### 1. 별도의 보안그룹 생성
- 별도의 보안그룹을 EFS 용을 생성해준다.
	- Type : NFS
	- Port : 2049
- source 에 EC2 IP가 아닌 EC2 보안 그룹 ID를 지정해준다.
	- EC2 IP가 바뀌어도 연결 유지할 수 있어서

### 2. 마운트 타겟 생성
**AZ** : EC2가 있는 AZ
**Subnet** : EC2와 동일한 서브넷
- EC2와 정확히 같은 AZ의 서브넷에 만들어야함
- 다른 AZ이면 cross-AZ 비용 및 지연시간 증가
**SG** : 'efs-sg'

### 3. Access Points 생성

EFS 서비스를 사용하려는 컨테이너 이름을 지정해주면 됨
ex) Redis
Access Point : ap-redis
Root Directory Path : /redis

### 4. EC2에 마운트

```bash
  # EFS 헬퍼 설치 (Amazon Linux 2023)
  sudo yum install -y amazon-efs-utils

  # 마운트 디렉토리 생성
  sudo mkdir -p /mnt/efs/redis /mnt/efs/spring-logs /mnt/efs/prometheus \
                /mnt/efs/loki /mnt/efs/grafana /mnt/efs/alloy /mnt/efs/certbot

  # 각 Access Point별 마운트 (fs-ID, fsap-ID는 콘솔에서 확인)
  sudo mount -t efs -o tls,accesspoint=fsap-xxxxxxx fs-xxxxxxxxx:/ /mnt/efs/redis
  # 나머지 6개도 동일하게

  영구 마운트 (/etc/fstab):
  fs-xxxxxxxxx:/ /mnt/efs/redis efs tls,accesspoint=fsap-xxxxxxx,_netdev 0 0
```

### 5. 볼륨 교체

![](../images/스크린샷%202026-04-20%2019.50.40.png)

## EFS 선택 이유

핵심 문제: 컴퓨팅과 스토리지가 붙어있음

  ECS는 컨테이너를 언제든 종료하고 새로 띄우는 구조. Task가 교체되면 컨테이너 내부 데이터는 사라짐. 로컬 named volume도 결국 EC2 디스크에 있기 때문에 EC2 자체가 내려가면 동일하게 유실.

  이를 해결하려면 컴퓨팅(EC2/ECS Task)과 스토리지를 분리해야 함 — PostgreSQL을 RDS로 분리한 것과 같은 원칙.

  **이유 1 — EC2 장애 시 데이터 유실 방지**

  EC2가 크래시되거나 교체되면 로컬 디스크의 named volume도 함께 사라짐. EFS는 EC2와 독립된 네트워크 파일시스템이라 EC2가 내려가도 데이터가 그대로 유지됨. 
  새 EC2나 새 Task가 뜨면 그냥 다시 마운트하면 됨.

  **이유 2 — Redis AOF 복구 보장**

  Redis는 인메모리 DB라 재시작하면 데이터가 사라짐. AOF(Append-Only File) 방식으로 디스크에 write log를 쌓으면 복구 가능하지만, 그 AOF 파일이 EC2 로컬에 있으면 의미가 없음.

  EFS에 AOF 파일을 저장하면 Task 재시작 시 EFS의 AOF 파일을 그대로 읽어 데이터 유실 없이 복구됨. 설정은 appendonly yes, appendfsync everysec (NFS 레이턴시 고려해 always는 배제).

  **이유 3 — 모니터링 데이터 보존**

  Prometheus TSDB(15일 보존), Loki 로그(7일 보존), Grafana 대시보드 설정이 ECS Task 교체마다 초기화되면 운영이 불가능함. EFS에 마운트해 Task 생명주기와 무관하게 데이터를 유지.

  **이유 4 — 컨테이너 간 파일 공유**

  Spring 앱이 /var/log/re-caring에 로그 파일을 쓰고, Alloy 컨테이너가 같은 경로를 read-only로 마운트해서 Loki로 전송하는 구조. 두 컨테이너가 같은 EFS 경로를 공유하기 때문에 가능함.

  이유 5 — ElastiCache/EBS 대비 비용

![](../images/스크린샷%202026-04-20%2020.07.24.png)

  ElastiCache는 비용 대비 MVP 규모에서 과함. EBS는 하나의 EC2에만 붙어서 멀티 Task 구성 시 공유 불가. EFS는 낮은 비용으로 NFS 기반  공유 스토리지 역할을 함.