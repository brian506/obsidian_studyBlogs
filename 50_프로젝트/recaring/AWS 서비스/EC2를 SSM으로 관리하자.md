## 기존 방식 - SSH

기존에는 서버에 접속하기 위해서 EC2의 인바운드 그룹(22번)을 열고 22번 포트와 Public IP주소로 접속했다. 
- 서버에 접속하려면 이와 같이 무조건 EC2에 **Public IP 주소와 인바운드 22번 포트**가 필수적이다.
	- 보안상 불리하다

**인바운드 방식** : 클라이언트(SSH) -> 서버 
## SSM(Systems Manager)

> AWS가 제공하는 EC2 인스턴스 관리 도구 모음

접속 : aws ssm start-session --target [인스턴스 ID]

- Session Manager : SSH키 없이 EC2에 접속
- Run Command : 여러 EC2에 한 번에 명령어 실행
- Patch Manager : OS 패치 자동화
- Parameter Store : 환경 변수/시크릿 키 등 저장
- Inventory : EC2에 뭐가 설치됐는지 자동 수집

SSM은 EC2안의 SSM Agent가 AWS SSM으로 요청을 보내서 뭐 할 거 없는지 Polling 하는 방식

**아웃바운드 방식** : 서버(EC2 내의 **SSM Agent**) -> AWS SSM 서비스 엔드포인트
- EC2의 SSM Agent가 HTTPS 아웃바운드로 SSM 서비스에 수행해야할 '명령'이 있는 지 물어본다.

>SSM Agent 는 Amazon Linux 2/2023 AMI로 인스턴스 생성해야함.

### SSM 동작 방식
1. **부팅 : ssm + ec2messages** 등록
	1. ec2 부팅
	2. SSM Agent 시작
	3. ssm.ap-northeast-2.amazonaws.com 호출
	4. ec2 등록하고, ec2messages 엔드포인트로 long-polling 시작
		- 명령 20초 간격으로 polling
2. **ssmmessages : aws ssm start-seesion**
	1. SSM 서비스에 세션 요청
	2. SSM 서비스 -> ec2messages 방식으로 Agent에게 '세션 요청' 푸시
	3. Agent -> ssmmessages 엔트포인트로 웹소켓 연결
	4. 로컬도 ssmmessages로 웹소켓 연결
	5. 로컬 PC <-> Agent 는 이후 ssmmessages로 로컬의 명령어 통신
**ssmmessages가 중간에서 중계하는 방식**

### SSM Endpoint ? 
SSM의 Endpoint의 주소는 `ssm.ap-northeast-2.amazonaws.com`와 같이 인터넷에 노출된 공인 도메인으로 이루어져있다.

즉, EC2에서 호출할 때
- **Public Subnet**은 0.0.0.0/0 으로 인터넷에 접속하고
- **Private Subnet**은 NAT을 통해 인터넷에 접속한다.

근데 여기서 **AWS 서비스끼리의 통신은 IGW를 거치지 않고 VPC Endpoint를 이용해서 Private IP를 사용해서 연결**이 가능하다.
즉, **Private Subnet**에서의 통신 방식은 **VPC Endpoint**와 **NAT**을 통해 통신한다.

### VPC Endpoint?
VPC Interface Endpoint는 **IGW를 거치지 않고** 하나의 **VPC 안에서** 새로운 **ENI(Private IP)** 를 만들고, 이 ENI와 **VPC 밖에 있는 AWS 서비스와 연결**할 수 있도록 해준다.
- ENI의 목적지를 AWS 서비스 등으로 연결
- Private Subnet에서 외부 인터넷으로 아웃바운드할 때 NAT을 사용하지 않기 위해 사용한다.(비용 문제)

**동작 방식**
1. EC2 가 SSM 도메인 해석
2. VPC 내부 Route53 Resolver가 응답을 가로챔
3. VPC Endpoint ENI의 Private IP 반환
	- 원래라면 공인 IP주소를 반환하지만, ENI 주소를 반환
-> 같은 도메인이 **VPC 안에서는 Private IP로**, **VPC 밖에서는 Public IP로** 해석

**장점**
- 트래픽이 인터넷으로 안나감
- NAT GW 데이터 비용 안듬
- 빠름

### EC2마다 각각 VPC Endpoint가 생성되나?
VPC Endpoint는 EC2를 감싸는 VPC 안에 설치하는 공동 인프라이다.

- **Private Subnet** : 프라이빗 서브넷 공간에 SSM용 VPC Endpint를 한 개 만들면, 그 서브넷 안에 모든 EC2가 그 하나의 ENI를 공유해서 SSM과 통신한다.
- **Public Subnet** : 퍼블릭 서브넷에 있는 EC2는 이미 IGW와 통신 가능하므로, VPC Endpoint 불필요

즉, 하나의 VPC 안에서 생성된 EC2는 그 VPC안에 VPC Endpoint가 생성되는 것이다.

## 트러블슈팅 - VPC Endpoint의 보안그룹

![](../../../images/스크린샷%202026-05-28%2004.07.22.png)

먼저 결론을 말하자면, **VPC Endpoint 는 ENI를 가지니까, ENI에는 보안그룹**이 붙어야 한다.

**동작 예시**) Private Subnet인 App -> SSM
1. App : 아웃바운드 443으로 요청
2. VPC Endpoint ENI : 인바운드 443 허용

VPC 외부의 서비스를 VPC 내부로 끌어오기 위해 서브넷 안에 **독립적인 ENI와 Private IP**를 할당 받으므로, **독립된 IP를 가진 목적지**가 생겼으니 이 엔드포인트로 들어오는 트래픽을 통제할 **보안 그룹 필요**

결국 내가 겪었던 문제는, 해당 VPC Endpoint ENI는 서비스도 아니고 뭣도 아닌데 보안그룹이 필요하는지도 몰랐다. Private Subnet과 VPC 밖에 있는 서비스를 연결해 주기 위해 하나의 통로(VPC Endpoint ENI)를 만들어줘야 하는데, 이 IP도 결국 목적지를 가지므로 보안그룹 인바운드 443에 App-sg를 허용해줘야 한다. 
- 소스를 넘어오는 **IP주소 또는 해당 전송자(서비스)의 보안그룹**으로 인바운드에 추가한다.

### Public, Private Subnet에 있는 EC2에서 VPC Endpoint 로 통신하면, Public Subnet의 EC2는 어떻게 통신하나?
- 퍼블릭 서브넷도 IGW가 아닌 VPC Endpoints를 사용한다.

그 이유는 Private DNS의 영향 범위는 VPC 전체 단위이다. AWS는 해당 VPC 내부에 있는 모든 EC2에게 SSM 주소를 찾으면 내부의 VPC Endpoint ENI를 사용하게끔 일괄적으로 강제한다.

따라서 Public Subnet의 보안그룹을 VPC Endpoint의 인바운드 443포트 소스로 설정해야한다.

### 보안 그룹 - Stateful
한번 인바운드나 아웃바운드로 **허용된 연결**은, 그 연결이 끝나고 **되돌아오는 트래픽**에 대해서 보안 그룹이 **기억하고 자동으로 통과**시켜준다.
