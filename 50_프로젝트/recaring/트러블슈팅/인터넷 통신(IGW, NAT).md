## 문제 발단

ECS 태스크를 awsvpc 모드로 전환하고 EC2 launch type에 올렸다. 해당 EC2는 Public Subnet에서 돌아가고 있어서 외부와의 인터넷 통신에 대해 아무 문제가 없을 줄 알았다. 그런데 스프링 앱 컨테이너에서 아웃바운드로 CoolSMS로 보내는 요청이 전부 타임아웃이 발생했다.

결론부터 말하자면, awsvpc 모드는 EC2 launch type에서 쓰면 ENI에 Public IP를 자동 할당할 수 없다. 따라서 해당 EC2를 Private Subnet으로 옮기고 Nginx EC2(로드밸런서용)만 Public Subnet으로 분리했다. Private Subnet의 아웃바운드 요청은 NAT Gateway가 처리한다.

## 인터넷은 Public IP로만 라우팅된다

**IGW(Internet Gateway)** 는 VPC와 인터넷을 잇는 논리적 컴포넌트이다.
1. 라우팅 게이트 : 퍼블릭 서브넷의 라우팅 테이블에서 0.0.0.0/0 -> IGW로 보낼 때, IGW가 트래픽을 받아 인터넷으로 보냄
2. Public IP로만 통신

EC2 인스턴스는 자기 자신의 **Public IP를 알지 못하고** Private IP만 갖는다.

Private IP 대역은 다음 세 가지다.

```
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
```

이 주소를 출발지로 갖는 패킷이 인터넷 라우터에 도착하면 드롭된다. 이 Private IP는 네트워크 대역 개수 문제로 값이 중복될 수 있는 IP이다. 따라서 고유한 Public IP로 통신해야한다.

여기서 **NAT**으로 해결한다. 내부에서 나가는 패킷의 출발지 IP를 NAT이 가진 공인 IP로 바꿔서 내보내고, 응답이 돌아오면 다시 원래 내부 IP로 바꿔서 전달한다. 

```
[집]
  192.168.0.x (내부 기기) → 공유기(NAT) → 공인 IP → 인터넷

[AWS]
  10.0.2.x (프라이빗 태스크) → NAT Gateway → Elastic IP → 인터넷
```

___

### IGW (인바운드 : 인터넷 -> 내부)

```
클라이언트 → IGW → EC2(Elastic IP) → nginx
```

EC2가 EIP(Public IP)를 가지고 있으므로 인터넷 어디서든 해당 EIP로 패킷을 보낼 수 있고, IGW가 받아서 EC2에 전달한다.
IGW는 IP변환을 직접 수행하지 않고, VPC 내부에서 1:1 NAT 매핑(Public IP -> Private IP)이 이루어진다. 

### NAT Gateway(아웃바운드 : 내부(Public IP 없는) -> 인터넷)

```
Spring App -> NAT Gateway -> NAT의 EIP로 변환 -> IGW -> 인터넷
```

NAT Gateway는 자기 자신의 Public IP를 가진다. 
Private IP만 가진 리소스 대신 자기 EIP를 빌려줘서 인터넷으로 내보낸다.
**Nginx를 거치지 않고 바로 인터넷으로 나간다**
#### EIP가 두 개 필요한 이유(NAT, EC2)

왜 EC2, NAT 둘 다에 EIP를 할당해야할까? 
-> **요청 주체가 다르면 Public IP 소유자도 달라야한다.**

| 주체                          | 방향       | 필요한 Public IP     |
| --------------------------- | -------- | ----------------- |
| Public Subnet에 있는 Nginx EC2 | 인바운드 수신  | EC2 Elastic IP    |
| NAT GW                      | 아웃바운드 대행 | NAT GW Elastic IP |
#### NAT Gateway의 대안 : VPC Endpoint

ECR이나 Parameter Store처럼 AWS 서비스에 접근하기 위해서만 NAT GW가 필요한 환경이라면, Interface Endpoint로 대체해서 NAT GW를 아예 없앨 수도 있다. 다만 외부 인터넷(FCM, CoolSMS 같은 third-party API)에 나가야 한다면 NAT GW가 필요하다.

## Public Subnet / Private Subnet

라우팅 테이블에서 어떤 서브넷으로 갈 지는 **Longest Prefix Match**로 결정된다. 
- 목적지 IP와 가장 구체적으로 일치하는 것이 우선

- 목적지가 10.0.2.35라면 → `10.0.0.0/16`이 더 구체적이므로 local로 라우팅
- 목적지가 8.8.8.8이라면 → `10.0.0.0/16`은 일치하지 않고 `0.0.0.0/0`만 남으므로 IGW 또는 NAT GW로

### 모든 서브넷에 Public IP와 Private IP가 공존

퍼블릭, 프라이빗 서브넷 모두 Public, Private IP가 존재한다.
- **Private IP** : VPC 내부 통신에 사용 
- **Public IP** : 인터넷으로 나갈 때만 IGW가 변환

**IGW는 Public IP를 가진 리소스의 트래픽만 인터넷으로 내보낸다.**
Public Subnet에 EC2가 있어도 Public IP가 없는 리소스는 인터넷으로 나가지 못한다.

## 프라이빗 서브넷 이전

### 퍼블릭 서브넷에 있을 떄 보안상 유의해야할 점

**1. 보안 그룹은 임시방편**
- 보안그룹에 인바운드 규칙을 막아놔도, 디버깅을 위해 잠깐 열어놓게되면 인터넷에 그대로 노출된다. 방어선이 한겹이냐 두겹이냐의 차이

**2. 메타데이터 서비스 노출과 SSRF**

**3. 로그와 감사의 어려움**
- 퍼블릭 서브넷의 호스트는 인터넷 스캐너가 끊임없이 두드린다. VPC Flow Logs를 보면 정상적인 트래픽과 스캐닝이 섞여서 이상 행동을 찾기 어렵다. 프라이빗 서브넷은 트래픽 패턴이 단순해서 이상 징후가 눈에 잘 띈다.

### 해결 방안

**방안 A: 작은 EC2 한 대를 nginx 전용으로 두기**

```
[퍼블릭 서브넷]
  t3.nano (nginx + certbot 전용)
    ↓ Cloud Map / 내부 ALB
[프라이빗 서브넷]
  t3.medium (Spring App, Redis, Monitoring, LLM)
```

- 퍼블릭 서브넷에는 최소한의 표면적만 남는다 (nginx 80/443).
- 기존 t3.medium은 그대로 프라이빗 서브넷으로 옮긴다.
- 비용 증가: t3.nano 약 월 4달러 + NAT GW 트래픽 증가분.
- 단점: nginx 전용 인스턴스의 SPOF(단일 장애점). HA 원하면 ASG + 두 대 또는 ALB로 결국 돌아가야 한다.

**방안 B: ALB + Fargate 조합으로 전환**

- nginx를 ALB로 대체하면 EC2를 아예 없앨 수 있다.
- 모든 워크로드를 Fargate로 옮기면 ENI 한도 문제, EC2 관리 부담, 패치 모두 사라진다.
- 하지만 SSE는 ALB의 idle timeout 한계와 L7 종단 특성 때문에 깔끔하지 않다. NLB로 가야 하는데 그러면 ACM 인증서 처리와 Let's Encrypt 자동 갱신을 다시 설계해야 한다.
- 비용도 Fargate가 EC2 launch type보다 단위당 비싸다. 트래픽 패턴이 일정하다면 EC2 launch type이 보통 저렴하다.

### nginx + certbot을 EC2에서 직접 운영

- SSE 스트림 유지 (ALB의 idle timeout 한계 회피)
- Let's Encrypt의 HTTP-01 ACME 챌린지가 80포트로 직접 들어와야 함

### NAT Gateway 선택 근거

  **1. bridge 모드로 변경 → 탈락**
  - 단일 EC2에서 고정 포트 충돌 → 롤링 업데이트 불가
  - 이미 포폴에 awsvpc 롤링 업데이트로 작성

  **2. assignPublicIp: ENABLED → 탈락**
  - Fargate 전용, EC2 launch type에서 무시됨

 **3. ALB 추가 → 탈락**
  - ALB는 인바운드 라우팅, 아웃바운드 문제 해결 안 됨
  - nginx+certbot 제거 필요 (SSE 장기 연결 최적화 근거 사라짐)
  - NAT Gateway보다 비쌈 (~$60-65 vs ~$43)

  **4. NAT Gateway → 선택**
  - 태스크 ENI를 Private 서브넷으로 이동
  - Private 서브넷 라우팅: 0.0.0.0/0 → NAT GW
  - 아웃바운드 인터넷 해결
  - 기존 구조(awsvpc, nginx, Cloud Map) 전부 유지
