>AWS의 Docker 이미지 저장소

nginx, redis 등 공식 이미지는 ECR 불필요

같은 리전 내 ECS/EC2 pull 시 데이터 전송 비용 0원 (리전 간이면 과금)
비용: 저장 $0.10/GB/월, 같은 리전 데이터 전송 무료

Image URI 형식

     {계정ID}.dkr.ecr.{리전}.amazonaws.com/{리포지토리}:{태그}
     예) 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/recaring/spring-app:abc1234
### 1. Image Lifecycle 설정
반복되는 배포 후에 쌓이는 이미지를 생명 주기 정책으로 삭제해줘야 비용 과금이 많이 발생하지 않는다.


![](../images/스크린샷%202026-04-20%2020.55.52.png)![](../images/스크린샷%202026-04-20%2020.56.47.png)

### 2. EC2 IAM Role 생성 및 연결

1. IAM Role 생성
- EC2 인스턴스에서 `Policy: AmazonEC2ContainerRegistryReadOnly` 추가  ← pull만, push 불가
2. IAM Role 연결
- EC2 인스턴스에서 IAM role 수정 후 위에서 생성한 IAM Role 연결

### 3. Github Actions용 OIDC 권한 부여
**동작 원리**
1. Github이 JWT 토큰 발급
2. AWS STS에 JWT 제출
	- STS : 외부(GitHub)의 토큰을 확인하고, 실제 AWS 자원을 만질 수 있는 임시 키(Access Key ID, Secret Key, Session Token)를 생성해주는 서비스
3. AWS가 Github OIDC Provider에 토큰 검증 요청
4. 검증 성공 -> 임시 자격증명 발급
5. ECR push

![](../images/스크린샷%202026-04-20%2023.50.48.png)

### 4. ECR 이미지 푸시


```bash

  # 1. JAR 빌드
  ./gradlew bootJar

 # SHA 추출
  SHA=$(git rev-parse --short HEAD)
  
  # 2. ECR 로그인
  aws ecr get-login-password --region ap-northeast-2 | \
    docker login --username AWS --password-stdin \
    757052694924.dkr.ecr.ap-northeast-2.amazonaws.com

  # 3. Docker 이미지 빌드
  aws ecr get-login-password --region ap-northeast-2 | \
    docker login --username AWS --password-stdin \
    757052694924.dkr.ecr.ap-northeast-2.amazonaws.com

  # 빌드 + 태그 + 푸시
  docker build -t 757052694924.dkr.ecr.ap-northeast-2.amazonaws.com/re-caring-api:$SHA .

  # 5. ECR 푸시
 docker push 757052694924.dkr.ecr.ap-northeast-2.amazonaws.com/re-caring-api:$SHA
```

### 5. 권한 생성
#### 1. EC2 인스턴스용 역할 
- 권한 : `AmazonEC2ContainerServiceforEC2Role`
- 이유 : EC2가 ECS에 자신을 등록하고 통신하기 위해 필요
#### 2. ECS 관리자 신분 역할
- 권한 : `AmazonECSTaskExecutionRolePolicy`
- 이유 : ECR에서 이미지를 pull하고, 로그를 CloudWatch에 찍어주기 위해 필요

- 권한 : `recaringSSMReadPolicy`(커스텀 정책)
- 이유 : SSM에 있는 환경 변수를 코드레벨이 아닌 SSM 자체에서 읽고 복호화하는 권한을 얻기 위해

#### 3. SQS에 접근하는 인라인 정책
- 이유 : 메시지 큐에 접근하기 위해 필요한 권한 부여
#### latest 태그의 문제점

- 롤백 불가능 : 새로운 코드를 빌드해서 latest로 푸시하면, 기존에 잘 돌아가던 latest 이미지가 덮어씌워짐. 만약 배포 후 장애가 발생했을 때, 이전 버전으로 되돌리고 싶어도 이전 버전이 뭔지 모름
- 추적성 상실 : 지금 실행 중인 이미지가 어떤 소스 코드 기반으로 빌드된 것인지 파악하기 힘듬
- 캐싱 배포 이슈 : 이미지를 가져올 때, 이미 로컬에 latest 라는 이름의 이미지가 있으면 새 이미지를 내려 받지 않고 기존의 낡은 이미지를 그대로 써버리는 경우 발생
l
## 배포 흐름

1. Code Push : Github에 코드 올림
2. Build & Push (CI) : Github Actions가 코드를 가져와서 빌드하고, 이를 도커 이미지로 만들어서 ECR에 Push 한다.
3. Pull & Run (CD) : EC2 서버가 ECR 창고에 접속해서 최신 이미지를 Pull 한 뒤 컨테이너로 실행한다.

## 도커 이미지 안에 있는 것들

이미지 내부 구성 요소
- 운영체제 레이어 : 가벼운 리눅스(예. Alpine, Ubuntu)
- 런타임 레이어 : JDK or JDE
- 애플리케이션 코드 : 빌드된 .jar 파일
- 환경 설정 및 실행 명령어