![](../../images/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-05-11%2016.14.49.png)

비용상의 문제로 ALB를 사용하지 못해서 인증서 갱신을 주기적으로 자동화할 수 있게 세팅을 해야한다. 
인증서는 90일의 유효시간을 갖고, 보통 만료 30일전에 갱신하는 것이 좋다.
여기서 certbot은 15일마다 갱신주기를 가질 것이다.
### Nginx가 EC2 launch type인 이유
- Fargate 컨테이너는 독립된 마이크로 VM에 격리되어 있어 EC2 호스트의 포트 80/443을 직접 바인딩할 수 없다. 
- EC2-bridge 네트워크 모드에서는 `hostPort: 80`, `hostPort: 443`으로 직접 바인딩이 가능하다. 

### Certbot이 Fargate인 이유
- Certbot은 주기적 실행(스케줄 태스크) 패턴이라 항상 컨테이너가 떠 있을 필요가 없다. Fargate는 실행 시간만큼만 비용이 발생하므로 간헐적 실행 작업인 인증서 갱신과 같은 일회성 작업에 적합하다.
- Fargate는 EFS를 별도 플러그인 없이 네이티브로 직접 마운트한다. 

### EFS를 인증서 스토리지로 사용하는 이유
- Certbot이 갱신한 인증서를 nginx가 읽어야 한다. 두 컨테이너는 분리된 환경이기 때문에 로컬 스토리지나 EBS 볼륨으로는 파일 공유가 불가능하다.
- EFS는 NFS 기반 네트워크 파일시스템으로, 여러 EC2와 Fargate 태스크가 동시에 같은 파일시스템을 마운트할 수 있다. Certbot이 갱신한 인증서를 EFS에 쓰면 Nginx가 동일한 EFS 경로를 읽어 최신 인증서를 사용할 수 있다.
- EC2가 다운됐을 때를 대비해서 인증서를 서버리스 서비스에 넣어두어 장애지점을 극복한다.

`/letsencrypt` : 인증서 경로
`/webroot` : ACME 검증 파일

ACME : 해당 도메인이 정말 리얼 도메인이 맞는 지 확인하는 방식
## 인증서 갱신 전체 흐름

### 1. EventBridge가 매월 1,15일 오전 2시에 certbot Fargate 태스크 실행
**IAM 권한 정책**
- `ecs:RunTask` : certbot Fargate 태스크를 실행하는 권한
- `iam:PassRole`: ECS 태스크에 Task Execution Role을 넘겨주는 권한

### 2. certbot entrypoint.sh 실행
- 존재하면 -> certbot renew
- 없으면 -> certbot certonly(최초 발급)
certbot renew는 만료 30일 이내일 때만 실제 갱신하고, 아니면 그냥 종료함

### 3. certbot <-> Let's Encrypt 통신 후 인증서 발급
1. Certbot -> Encrypt 서버 : 해당 도메인의 인증서 요청
2. Encrypt 서버 -> Certbot : 랜덤 토큰의 파일 생성 요청
3. Certbot : EFS에 해당 토큰의 파일 생성
4. Encrypt 서버 : 해당 토큰 파일로 http 요청
5. Nginx가 수신 후 EFS 해당 경로로 서빙
6. Encrypt 서버 -> Certbot : 파일 확인되면 인증서 발급
7. EFS의 /letsencrypt 에 인증서 저장

### 4. certbot이 ECS API 호출로 nginx reload(재배포)
- nginx 컨테이너가 새로 시작되면서 EFS /letsencrypt 최신 인증서 로드
- nginx는 live/ 경로를 참조하므로 재시작시 자동으로 최신 인증서를 읽음
**certbot과 nginx는 분리된 컨테이너므로 직접 접근하지 못함. 따라서 ECS API를 통한 강제 재배포 방식 사용**
- Certbot 컨테이너가 nginx의 컨테이너를 재배포하는 방식이므로 certbot 컨테이너 안에 `ecs:UpdateService` 권한이 있어야함

![](../../images/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-05-11%2019.16.31.png)

## EC2 호스트 레벨 마운트 vs ECS 태스크 레벨 마운트

### ECS 태스크 레벨 마운트
- ECS Agent가 `amazon-ecs-volume-plugin`을 통해 자동으로 EFS를 마운트한다. Fargate에서는 네이티브로 지원되어 잘 동작하지만, EC2 Launch type에서는 플러그인이 필요하다.
- 현재 도커 버전에서는 플러그인 호환이 안되어 EC2 호스트 레벨로 마운트할 것이다.

### EC2 호스트 레벨 마운트
- EC2 OS 레벨에서 EFS를 직접 마운트
- ECS 태스크 정의에서 host bind mount 사용
`"volumes": [{"name": "letsencrypt", "host":{"sourcePath":"/mnt/efs/letsencrypt"}}]`
