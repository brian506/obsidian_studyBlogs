기존 Nginx가 포트 번호를 찾아서 무중단 배포를 실행하는 blue-green 무중단 배포를 스크립트를 통해서 진행했었다. 컨테이너 상태와 복구 로직을 수동으로 관리하지 않고, 현재 ECS로 관리하는 장점을 이용해서 ECS의 Rolling Update로 무중단 배포를 수행하고자 한다.

![](../images/스크린샷%202026-04-30%2022.11.22.png)

### App Service가 bridge 모드였으면?
- bridge 모드는 EC2 호스트의 네트워크를 공유한다.
- 기존 App V1이 EC2 호스트의 8080 포트를 물고 서비스 중
- 새 App V2를 배포한다. V2도 켜지면서 8080포트를 사용하려고 하지만 EC2의 포트를 같이 쓰므로 8080포트는 유일해야 하기 떄문에 같은 컨테이너가 같이 띄워질 수 없다. 

### 왜 App Service 만 awsvpc 모드?
- awsvpc 모드는 컨테이너마다 완전히 독립된 고유의 사설IP(Task ENI)를 부여한다.
- 기존 App V1이 `10.0.1.10`이라는 고유 IP를 받고 8080포트를 연다. (10.0.1.10:8080)
- 새 App V2를 배포한다. V2는 `10.0.1.11`이라는 새로운 IP를 받고 8080 포트를 연다ㅏ.
- IP주소 자체가 서로 다르기 때문에, 한 대의 EC2안에서도 포트 충돌이 발생하지 않는다.

## 무중단 배포 과정

![](../images/스크린샷%202026-04-30%2022.56.24.png)

1. ECS가 App V2 태스크를 생성한다. 새로운 Task ENI(10.0.1.2)를 할당 받는다
	- EC2 Medium 스펙은 ENI를 최대 3개까지 가능하므로, (Primary ENI(bridge 네트워크와 연결된) + App V1 + App V2)로 가능
2. V2가 정상적으로 부팅되면, ECS는 CloudMap의 app.local 도메인에 V2의 IP(10.0.1.2)를 추가한다. 
	- 이 짧은 순간에 V1,V2 가 동시에 존재
3. 앞단에 있는 Nginx는 프록시 요청을 보낼 때 CloudMap 내부 DNS를 찌른다.
	- Nginx는 이제 V2의 IP를 인지하고 트래픽을 V2로도 보낸다.
4. ECS가 CloudMap에서 V1의 IP를 삭제한다.
5. Nginx는 더 이상 V1으로 트래픽을 보내지 않고, V1은 기존에 처리 중이던 남은 요청만 마무리한 뒤 종료된다. Task ENI도 반납

## 설정

최소 정상 백분율(Minimum healthy percent) 
최대 백분율 (Maximum percent)

ex) 최소 정상 백분율 : 100%, 최대 백분율 : 200%
- 기존 V1을 유지한 상태에서 새 V2를  띄워 총 200%를 만든 다음, 트래픽을 넘기고 V1을 죽여서 다시 100%로 맞추라는 지시어


### Nginx DNS 동적 Resolving
- Nginx는 기본적으로 한 번 찾은 IP를 영원히 캐싱한다.  CloudMap에서 IP가 바뀌었는데 Nginx가 모르면 에러가 나기 때문에, 지속적으로 CloudMap에 요청해서 IP주소를 갱신해야한다.
- app.local의 실제 사설 IP가 무엇인지 Nginx가 알기 위함

```
# AWS VPC 기본 내부 DNS (169.254.169.253), 5초마다 IP 변경 감지 resolver 169.254.169.253 valid=5s; set $backend "http://app.local:8080"; proxy_pass $backend;
```

`169.254.169.253` 
> AWS 환경(EC2, 컨테이너 등) 내부에 기본적으로 존재하는 내부 전용 DNS 서버의 고정된 IP 주소. 

- Nginx에게 최근 반영된 주소를 알기 위해 이 AWS 내부 DNS에게 쿼리한다.
`valid=5s` : 5초 간격으로 물어본다.
`$backend` : 변수에 담아서 Nginx가 찾는 IP주소를 갱신하도록 한다.

### 보안 그룹 (Security Group) 설정:

App Service(awsvpc)는 Nginx(bridge, 호스트 IP)로부터 오는 트래픽을 받아야 합니다.
App Service에 연결된 보안 그룹의 인바운드 규칙에 Nginx가 속한 EC2의 보안 그룹이나 사설 IP 대역(10.0.x.x)에 대해 8080 포트를 열어두어야 통신이 가능합니다 

