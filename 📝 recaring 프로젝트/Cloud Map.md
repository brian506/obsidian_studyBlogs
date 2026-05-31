
## 도입 배경

awsvpc 모드에서의 App Task는 Rolling Update로 컨테이너를 계속 새로 띄우기 때문에, **매번 다른 private IP**를 할당 받는다.
Nginx는 트래픽을 App Task로 프록시해야 하는데, 매번 바뀌는 IP를 읽기 위해 어떤 방법을 써야할까?

## Cloud Map

>VPC 내부에서만 통하는 사설 DNS 서비스
>Route 53의 Private Hosted Zone을 자동으로 관리해주는 ECS 통합 서비스

- Namespace : 도메인 (ex. app.local)
- Service : namespace 안의 이름(ex. app)


|               | 대상           | 역할                     |
| ------------- | ------------ | ---------------------- |
| **Route 53**  | 실제 서비스 사용자   | 도메인의 Public IP 주소록     |
| **Cloud Map** | VPC 내부의 컨테이너 | 각 컨테이너의 Private IP 주소록 |

ECS Service에서의 **Cloud Map 동작 과정**
1. 태스크 V1 생성 (IP : 10.0.2.35)
2. ECS가 Cloud Map API 호출
3. Cloud Map : app.local의 A 레코드에 10.0.2.35 추가
	- DNS 등록 완료
4. 태스크 V1 종료
5. ECS가 Cloud Map API 호출
6. Cloud Map : app.local(도메인)에서 10.0.2.35 제거

즉 태스크의 생사를 Cloud Map이 **실시간으로 추적한다.** 
이로써 Nginx는 `app.local`을 조회하면 상태에 따른 다른 IP가 조회된다. 그래서 Nginx는 IP를 모르고도 살아있는 컨테이너의 IP에 도달할 수 있다.

# 프로젝트에서 고려해야 했던 것들

## Nginx DNS 캐싱 문제

Nginx는 도메인을 시작할 때 한 번만 해석하고 메모리에 캐싱한다.

**문제 상황**
1. Nginx 시작 시 `app.local`을 해석해서 private IP를 메모리에 저장한다.
2. 그 다음부터는 DNS를 다시 묻지 않는다.
3. 태스크가 바뀌어도 1번에서 저장한 private IP로 트래픽을 보낸다.
4. 그런데 해당 private IP는 이미 죽은 태스크 이므로 502 Bad Gateway가 발생한다.

**해결**
Nginx의 `resolver` 지시어를 써서 매 요청마다 DNS를 다시 해석하도록 강제한다.

```nginx
resolver 169.254.169.253  valid=5s;

server {
  location / {
    set $backend "http://app.local:8080";
    proxy_pass $backend;
  }
}
```

- `resolver`로 DNS 서버 명시 : `169.254.169.253` 는 AWS VPC 내부 DNS의 link-local 주소
	- VPC 내부 도메인과 인터넷 도메인 둘 다 해석
- nginx가 매 요청마다 변수를 다시 평가하고 DNS도 다시 해석

## Redis (bridge 네트워크 모드) CloudMap 사용한 이유?

Redis 컨테이너는 bridge 네트워크 모드이므로 재시작되어도 프라이빗IP가 변하지 않는다. 근데 왜 CloudMap을 사용할까?

### 상황 A : CloudMap 미사용
`SPRING_DATA_REDIS_HOST: 10.0.1.55` - EC2 Private IP

- DNS 쿼리가 없어서 빠르고 심플
- 만약 EC2 인스턴스에 문제가 생겨서 **EC2를 다시 띄우면, 새 EC2는 새로운 Private IP**를 받게 된다. EC2를 재시작할 때마다 **프라이빗 IP를 수동 업데이트** 해야 한다.
### 상황 B : CloudMap 사용
`SPRING_DATA_REDIS_HOST: redis.[도메인 내부 이름]` - CloudMap이 만들어준 내부 도메인

- EC2가 재시작되어도, ECS가 알아서 EC2의 새로운 Private IP를 redis.[도메인 내부 이름]에 갱신해준다.
- DNS 쿼리 비용이 발생하지만 매우 미미

#### 안써도 되는 서비스 예시
Nginx : Nginx는 트래픽을 받아서 안으로 넘겨주는 역할을 하므로, 다른 서비스가 Nginx를 찾을 일이 없다
Monitoring : Prometheus 기반의 모니터링은 각 컨테이너에 PULL하는 방식이므로 다른 서비스들이 Monitoring 컨테이너를 찾을 일이 없다.