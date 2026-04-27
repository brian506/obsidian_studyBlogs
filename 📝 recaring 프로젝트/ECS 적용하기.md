
ECR 관련해서 설정 파일을 마치고, EC2 연결을 할 것이다.
기존에 이미 있는 EC2 위에 ECS 를 띄울 것이다.

## ECS 개념

**실행 모드**
- Service : 24시간 상시 실행
- Task : 단발성 실행

### 1. EC2에 ECS 에이전트 설치 및 클러스터 등록

```bash
# ubuntu ecs 에이전트 설치
sudo apt-get update && sudo apt-get install -y amazon-ecs-init
 # 2. 클러스터 이름 설정
 sudo bash -c 'echo ECS_CLUSTER=recaring-cluster >> /etc/ecs/ecs.config'

 # 3. ECS 에이전트 시작
 sudo systemctl enable --now ecs

 # 4. 등록 확인 (잠시 후)
 sudo systemctl status ecs
```



## 네트워크 모드에 따른 장단점

ENI : 독립적인 가상 랜카드

![](../images/스크린샷%202026-04-25%2017.18.15.png)
### bridge 모드
특징 : 로컬에서 쓰던 일반적인 도커 네트워크. EC2 인스턴스의 IP 1개를 모든 컨테이너가 공유하며, 도커가 내부적으로 가상 브리지를 만들어 통신
장점 : ENI 개수 제한 없음. 동적 포트 매핑을 지원하여, 포트 충돌 없이 여러 ㅓㄴ테이너를 무한정 띄울 수 있음.
단점 : 컨테이너 개별로 AWS 보안 그룹을 걸 수 없고, EC2 호스트 수준에서만 방어 가능

### awsvpc 모드
특징 : 컨테이너 하나당 독립적인 ENI와 VPC 프라이빗 IP를 하나씩 꽂아줌
장점 : 컨테이너가 하나의 독립된 EC2 인스턴스처럼 동작하므로 보안과 네트워크 추적 완벽
단점 : EC2 인스턴스 타입마다 꽂을 수 있는 ENI 최대 개수 제한

### host 모드
특징 : 도커의 가상 네트워크 계층을 무시하고, 호스트의 네트워크를 그대로 씀
장점 : 네트워크 병목 없어서 개빠름
단점 : 포트 충돌에 취약.(하나의 포트를 쓰는 컨테이너를 두개 못띄움)

## 현재 Nginx 로 Blue- green 무중단 배포시 

#### 🚨 문제 1: `awsvpc` 모드 선택 시 "ENI 고갈 에러"

- `t3.medium` 인스턴스가 가질 수 있는 **최대 ENI 개수는 딱 3개**입니다.
    
- 기본적으로 EC2 자체가 1개를 씁니다. (남은 자리 2개)
    
- Nginx+Certbot Task가 1개를 씁니다. (남은 자리 1개)
    
- Spring(Blue) Task가 1개를 씁니다. (남은 자리 0개)
    
- **무중단 배포 시작:** Spring(Green) Task를 새로 띄우려고 합니다. 하지만 ENI 자리가 없어서 **"ENI Limit Exceeded" 에러가 나며 배포가 완전히 실패**합니다.

#### 🚨 문제 2: `host` 모드 선택 시 "포트 충돌 에러"

- Spring(Blue)가 EC2의 8080 포트를 물고 있습니다.
    
- 무중단 배포를 위해 Spring(Green)을 띄우려는데, 8080 포트가 이미 사용 중이라며 컨테이너가 죽어버립니다.
