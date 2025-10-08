#AWS

## ssh 로 EC2 에 접속하기

`ssh -i "security-msa.pem" ubuntu@[PublicIP]`
# EC2 인프라 설정
## 보안그룹 설정
>_인바운드_ : 외부 -> EC2 내부로 접속 허용
  _아웃바운드_ : EC2 내부 -> 외부
### 보안그룹 세분화
#### Gateway-SG (EC2)
- 외부 포트 80(http) 또는 443(https)로 **Gateway 서비스만** 열어둔다.
	- 나머지 서비스들은 Docker 네트웨크 안에서 **내부 통신(expose)하므로 보안그룹 차단**
	- Gateway (80:19091) -> 외부에서 80으로 통신
	- 보안그룹에서 인바운드 규칙으로 `ssh` 포트만 허용
*인바운드* : http, https, ssh 허용

#### RDS-SG(RDS)
- EC2 와 같은 VPC/Subnet 에 두고, **EC2 랑만 연결 허용**
	- EC2 보안 그룹 안에서만 접근
- 현재 상황은 모든 서비스가 접근할 수 있으므로 나중에 각 서비스들을 독립적으로 EC2 를 분리할 때는 Auth-service-SG 만 허용
*인바운드* : Gateway-SG 허용

#### Redis-SG(ElasticCache)
- RDS 와 마찬가지로 EC2랑만 연결 허용
- 원래 2개의 레디스를 사용하지만 현재 상황은 하나의 레디스를 사용하고 DB 인덱스를 구분해서 사용
*인바운드* : Gateway-SG 허용

#### MongoDB
 **DocumentDB(AWS)**
 - RDS 와 마찬가지로 EC2랑만 연결 허용
 
  **Atlas(mongodb 콘솔)**
  - Atlas 콘솔 내에서 네트워크 설정에서 EC2 퍼블릭IP 주소를 할당(자체 IP 화이트리스트)
	- EC2로 접속해서 DB 볼 수 있음





**참고** : [[HTTP VS HTTPS]]
