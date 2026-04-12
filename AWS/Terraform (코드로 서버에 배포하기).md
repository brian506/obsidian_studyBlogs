#AWS 

새로운 프로젝트를 진행하면서 aws 콘솔에 계속 왔다 갔다 하는 것도 귀찮고, 내가 뭘 설정 했는 지 구체적으로 생각도 안날때가 많았다. 그리고 팀원 모두가 같은 인프라 환경을 다룰 때 공통적으로 관리하며 놓치지 않는 부분이 있으면 좋겠다는 생각이 들었다. 
그리고 최근에 aws 계정 관련해서 비용 폭탄을 맞았는데, 이 클라우드 환경을 효과적으로 관리하기 위해 terraform 이라는 IaS 를 사용해보고자 한다.

Terraform 은 클라우드 인프라를 코드로 정의하고, 버전을 관리하며(Git), 변경사항을 추적하고 인프라를 생성/변경/삭제하는 등의 작업을 자동화할 수 있다.

먼저 디렉토리 파일을 보고 어떻게 구성되는 지 살펴보자.

```
terraform/
├── main.tf          # provider 설정
├── variables.tf     # 변수 선언
├── outputs.tf       # 출력값
├── terraform.tfvars # 실제 변수값 (git에 올리면 안됨)
├── vpc.tf
├── security_groups.tf
├── ec2.tf
├── rds.tf
└── backend.tf       # S3 원격 상태 관리
```

ec2 에 설정할 항목들에 대해서 .tf 파일 형식으로 관리한다.
각 파일 구성은 밑에서 실제로 recaring 프로젝트 인프라 환경에 맞춰서 적용해볼 것 이다.
### 워크플로우

1. **코드 작성**
2. **terraform init** 
	- provider 에 설정된 플러그인 설치
3. **terraform plan**
	- state.tf 파일과 현재 Provider의 현재 상태와 비교
	- 비교한 내용들에 대해서 어떠한 변경 계획이 있는 지 출력
4.  **terraform apply**
	- 인프라 적용
5.  **terraform destroy**

## main.tf

어떤 클라우드의 region 을 사용할지 정하는 파일이다.

```hcl
provider "aws" {
	region = var.aws_region
}
```

- var 로 별도 변수 파일에 있는 값을 불러온다.

## backend.tf

Terraform의 기억 장치라고 할 수 있는 **state 파일**을 어디에 저장할지 설정하는 파일이다.
보통 AWS의 S3 에 저장한다.
#### 왜 필요한가?
- 팀원 모두가 동일한 인프라 상태를 공유해야 함
- 내 로컬 컴퓨터가 문제가 생겨도 인프라 정보 유지
- 두 사람이 동시에 인프라를 수정해서 꼬이는 것 방지 (dynamodb_table)


```hcl
terraform { 
	required_version = ">= 1.0" 
	
	required_providers { 
		aws = { 
			source = "hashicorp/aws" 
			version = "~> 5.0" # 5.x 버전 사용, 6.0은 자동 업그레이드 안됨 
			} 
	}

terraform {
	backend "s3" {
		bucket = "recaring-terraform-state" # 미리 만들어둔 S3 버킷 
		key = "/recaring/dev/terraform.tfstate" # S3 버킷 내부에 저장될 상태 파일(State File)의 '경로와 이름
		region = "ap-northeast-2" 
		dynamodb_table = "terraform-lock" # 동시 apply 충돌 방지(선택)
		encrypt = true
	}
}
```
- key 값에 경로 설정을 할 때, 보통 운영용, 개발용 등으로 나눠서 꼬이지 않도록 설정한다.
- dynamodb 는 여러 사람이 배포를 함께 진행했을 경우에 배포 환경이 꼬이지 않도록 락을 걸어서 동시 수정이 불가능하게 해준다.

## vcp.tf (네트워크 설정)

**프로젝트 기준 구조**

현재 구조는 Nginx 에서만 80,443 포트를 열어 요청을 받는다. 
app 서버는 Security Group 으로 외부에서 요청을 제한하고,
rds 는 app 서버에서 들어오는 요청만을 받는다.
또한, Nginx + app 은 하나의 Ec2 안에 있고 public subent 에 위치한다.
그리고 rds 는 private subnet 으로 아예 외부와 단절시킨다.
어차피 rds 는 외부에 요청을 보내거나 할 필요가 없고 app 에서만 요청을 받으면 되므로 app 포트만 열어주고 private subnet으로 단절시킨다. NAT 비싸서 일단 이게 최선인듯

### VPC 상세 설정

```hcl
resource "aws_vpc" "main" { 
	cidr_block = "10.0.0.0/16" 
	enable_dns_hostnames = true # EC2에 DNS 호스트명 부여 
	enable_dns_support = true 
	
	tags = { 
		Name = "${var.project_name}-vpc" 
		Environment = var.environment 
		} 
	}
```
- 여기서 설정한 cidr_block 은 내 VPC 주소이다.
- tags 를 이용해서 자원의 식별값을 직관적으로 알 수 있다.
- `10.0.0.0/16` : default 값으로 생성되는 주소
	- VPC(10.0.0.0/16) : 서울시 전체 (약 65000개 사용 가능)

### Public 서브넷 (EC2 - Nginx + App)

```hcl
resource "aws_subnet" "public" {
	vpc_id = aws_vpc.main.id
	cidr_block = "10.0.1.0/24"
	availablility_zone = "ap-northeast-2a"
	map_public_ip_on_launch = true
	
	tags = {
		Name = "${var.project_name}-public-subnet"
	}
}
```

- `map_public_ip_on_launch = true` : 이 서브넷에서 생성되는 모든 인스턴스에 자동으로 Public IP를 할당한다는 뜻
### Private 서브넷 (RDS)

```hcl
# RDS 가용영역 a
resource "aws_subnet" "private_a" {
	vpc_id = aws_vpc.main.id
	cidr_lock = "10.0.2.0/24"
	availabilty_zone = "ap-northeast-2a"
	
	tags = {
		Name = "${var.project_name}-private-subnet-a"
	}
}

# RDS 가용영역 b
resource "aws_subnet" "private_c" {
	vpc_id = aws_vpc.main.id
	cidr_lock = "10.0.3.0/24"
	availabilty_zone = "ap-northeast-2c"
	
	tags = {
		Name = "${var.project_name}-private-subnet-c"
	}	
}
```

- RDS 는 가용영역을 2개로 둬야 한다. -> 즉 서브넷을 2개를 만든다는 것이다.
	- AWS 에서 강제적으로 가용영역을 2개 만들도록 한다.
		- 인스턴스가 장애가 발생 시에 구동 될 수 있는 새로운 자리(서브넷)을 임시로 하나 더 만들어 두는것이다. 이렇게 됨으로써 중지 되지 않고 계속 운영할 수 있다.
		- `multi_az = true` : 이 명령어는 서브넷이 할당된 수만큼 실제로 인스턴스를 n개 생성하는 것이다.
		- `multi_az = false` : 이 명령어는 서브넷 수만 할당하고 실제 인스턴스는 늘리지 않는다.

### 인터넷 게이트웨이(IGW)  / Route Table
```hcl
resource "aws_internet_gateway" "main" { 
	vpc_id = aws_vpc.main.id 

	tags = { 
		Name = "${var.project_name}-igw" 
	} 
}

resource "aws_route_table" "public" {
	vpc_id = aws_vpc.main.id
	
	route {
		cidr_blick = "0.0.0.0/0" # 외부 모든 포트
		gateway_id = aws_internet_gateway.main.id
	}
	tags = { 
		Name = "${var.project_name}-public-rt" 
	}
}

# igw 와 route 연결
resource "aws_route_table_association" "public" {
	subnet_id = aws_subnet.public.id
	route_table_id = aws_route_table.public.id
}
```
AWS 콘솔에서 VPC 연결할 때는 인터넷 게이트웨이 설정을 따로 안했지만 그건 알아서 해준 것이다.

Public Subnet 이 외부 인터넷과 통신하려면 인터넷 게이트웨이를 할당해야지만 통신할 수 있다.

즉 쉽게 말하면,
VPC : 우리 집
IGW : 우리 집 현관문
Route Table : 밖(외부 인터넷)으로 나가려면 현관문으로 가라 라는 이정표

**인터넷 통신 흐름**
VPC -> IGW -> Subnet
IGW에 '이정표'의 여부(public/private)에 따라 subnet 으로 연결되는 것

## security_groups.tf

```hcl
# EC2 (Nginx + App + Redis + Certbot)
resource "aws_security_group" "app_server" {
	name = "${var.project_name}-recaring-app-sg"
	description = "App 서버 Security Group"
	vpc_id = aws_vpc.main.id
	
	# Nginx Inbound 
	ingress {
		description = "HTTP"
		from_port = 80
		to_port = 80
		protocol = "tcp"
		cidr_blocks = ["0.0.0.0/0"]
	}
	
	ingress {
		description = "HTTPS"
		from_port = 443
		to_port = 443
		protocol = "tcp"
		cidr_blocks = ["0.0.0.0/0]
	}
	
	# ssh inbound
	ingress {
		description = "SSH"
		from_port = 22
		to_port = 22
		protocol = "tcp"
		cidr_blocks = ["0.0.0.0/0]
	}
	
	# outbound
	egress {
		description = "모든 아웃바운드 허용"
		from_port = 0
		to_port = 0
		protocol = "-1"
		cidr_blocks = ["0.0.0.0/0]		
	}
	tags = {
		Name = "${var.project_name}-recaring-app-sg"
		Environment = var.environment
	}	
	
# RDS SG
resource "aws_security_group" "rds" {
	name = "${var.project_name}-rds-sg"
	description = "RDS Security Group"
	vpc_id = aws_vpc.main.id
	
	ingress {
		description = "App서버에서만 DB 접근 가능"
		from_port = 5432
		to_port = 5432
		protocol = "tcp"
		source_security_group_id = aws_security_group.app_server.od
	}
	
	egress {
		from_port = 0
		to_port = 0
		protocol = "-1"
		cidr_blocks = ["0.0.0.0/0]		
	}
	
	tags = {
		Name = "${var.project_name}-rds-sg"
		Environment = var.environment
	}
}
}
```

- App EC2 에서 외부 요청은 Nginx 로 80/443 만 허용한다.
- ssh 는 현재 깃 액션으로 배포를 하고 있기 때문에 깃 서버 포트를 지정하기 애매해서 일단 전체 포트로 열어뒀다.
- rds 는 ec2에서 오는 요청으로만 통신해야 한다.
	- EC2 의 SG 를 Inbound 한다.

## ec2.tf

```hcl
# 최신 Ubuntu 22.04 LTS AMIdata "aws_ami" "ubuntu" {  
    most_recent = true  
    owners = ["099720109477"]  
  
    filter {  
       name = "name"  
       values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]  
    }  
  
    filter {  
       name = "virtualization-type"  
       values = ["hvm"]  
    }  
}  
  
resource "aws_instance" "recaring-app-server" {  
  ami           = data.aws_ami.ubuntu.id  
  instance_type = var.app_instance_type  
  subnet_id     = aws_subnet.public.id  
  key_name      = var.key_name  
  
  vpc_security_group_ids = [aws_security_group.app_server.id]  
  
  root_block_device {  
    volume_type = "gp3"  
    volume_size = 30  
    encrypted   = true  
  }  
  
  user_data = <<-EOF  
    #!/bin/bash    
    set -e    
    exec > /var/log/user-data.log 2>&1 
     
    # 1. Docker 설치  
    curl -fsSL https://get.docker.com | sh    
    usermod -aG docker ubuntu    
    apt-get install -y docker-compose-plugin 
     
    # 2. 디렉토리 생성  
    mkdir -p /home/ubuntu/recaring/nginx/conf.d    
    mkdir -p /home/ubuntu/recaring/scripts    
    mkdir -p /home/ubuntu/recaring/data/certbot/conf    
    mkdir -p /home/ubuntu/recaring/data/certbot/www    
    chown -R ubuntu:ubuntu /home/ubuntu/recaring  
    
    # 3. 레포에서 파일 복사  
    cd /tmp    
    git clone https://github.com/brian506/re-caring.git    
    cp re-caring/docker-compose-dev.yml /home/ubuntu/recaring/    
    cp re-caring/nginx/conf.d/service-url.inc /home/ubuntu/recaring/nginx/conf.d/    
    cp re-caring/scripts/deploy.sh /home/ubuntu/recaring/scripts/    
    chmod +x /home/ubuntu/recaring/scripts/deploy.sh    
    rm -rf /tmp/re-caring  
    
    # 4. 초기 nginx 설정 (certbot HTTP 인증용)  
    cat > /home/ubuntu/recaring/nginx/conf.d/myapp.conf << 'NGINX'    
    server {        
	    listen 80;        
	    server_name re-caring.com;        
	    location /.well-known/acme-challenge/ {            
		    root /var/www/certbot;        
		}        
		
		location / {            
			return 200 'ok';        
		}    
	}    
	NGINX  
	
    # 5. nginx + certbot 컨테이너 시작  
    cd /home/ubuntu/recaring    
    docker compose -f docker-compose-dev.yml up -d nginx certbot  
EOF  
    
  tags = {  
    Name        = "${var.project_name}-app-server"  
    Environment = var.environment  
  }  
}  
  
# Elastic IP (기존 IP 재사용)  
resource "aws_eip" "app_server" {  
  instance = aws_instance.recaring-app-server.id  
  domain   = "vpc"  
  
  tags = {  
    Name        = "${var.project_name}-eip"  
    Environment = var.environment  
  }  
}  
  
output "app_server_ip" {  
  value       = aws_eip.app_server.public_ip  
  description = "앱 서버 퍼블릭 IP (43.200.235.247)"  
}
```

`exec > /var/log/user-data.log 2>&1 ` : 모든 출력을 해당 경로 파일에 기록
	`>` : 표준 출력을 파일로 리다이렉트
	`2>&1` : 표준 에러도 표준 출력과 같은 곳으로 보냄
	`exec`: 현재 쉘 프로세스 자체에 적용
## rds.tf

```hcl
# Parameter Store에서 RDS 환경변수 조회 
data "aws_ssm_parameter" "db_name" { 
	name = "/${var.project_name}/${var.environment}/DB_NAME" 
	with_decryption = true 
} 

data "aws_ssm_parameter" "db_username" { 
	name = "/${var.project_name}/${var.environment}/DB_USERNAME" 
	with_decryption = true 
} 

data "aws_ssm_parameter" "db_password" { 
	name = "/${var.project_name}/${var.environment}/DB_PASSWORD" 
	with_decryption = true # SecureString 타입 
}

# RDS 서브넷 그룹
resource "aws_db_subnet_group" "main" { 
	name = "${var.project_name}-rds-subnet-group" 
	subnet_ids = [aws_subnet.private_a.id, aws_subnet.private_c.id] 
	
	tags = { 
		Name = "${var.project_name}-rds-subnet-group" 
		Environment = var.environment 
	} 
}

resource "aws_db_instance" "postgres" {
	identifier = "${var.project_name}-postgres"
	
	engine = "postgres"
	engine_version = "16"
	instance_class = var.db_instance_class
	
	allocated_storage = 20
	max_allocated_storage = 100
	storage_type = "gp3"
	storage_encrypted = true
	
	db_name = data.aws_ssm_parameter.DB_NAME.value
	username = data.aws_ssm_parameter.DB_USERNAME.value
	password = data.aws_ssm_parameter.DB_PASSWORD.value
	
	vpc_security_group_ids = [aws_security_group.rds.id]
	db_subnet_group_name = aws_db_subnet_group.main.name
	
	publicly_accessible = false # 외부 직접 접근 차단 
	skip_final_snapshot = false # 삭제 시 스냅샷 생성 
	deletion_protection = true # 실수 삭제 방지 
	backup_retention_period = 7 # 7일치 자동 백업 
	backup_window = "03:00-04:00" # 백업 시간 (UTC) 
	maintenance_window = "Mon:04:00-Mon:05:00" 
	
	tags = { 
		Name = "${var.project_name}-postgres" 
		Environment = var.environment 
	}
}
```

## terraform 으로 배포하기

### 1. 먼저 terraform 과 AWS CLI 를 로컬에 설치한다.

```bash
# terraform 설치
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

terraform -version

# AWS CLI 설치
brew install awscli 
sudo apt install awscli 

aws --version
```

### 2. AWS IAM 권한 설정

![](../images/스크린샷%202026-03-29%2018.19.32.png)

- IAM 권한 추가 후 각 툴에 대한 권한을 전부 허용해준다.
	- 원래는 전부 허용하지 않고 필요한 것만 하는데, 일단 다 허용하자
	

```bash
aws configure

```
- aws configure 를 command 하고 Access Key ID, Secret Access Key 를 입력

### 3. terraform init
- 플러그인 / 모듈 초기화

### 4. terraform plan
- create/update/destory 실행 계획을 보여준다.

### 5. terraform apply
- 실행 ㄱㄱ

## 배포 이후 확인 과정

### 1. 기존 서버의 ssh 키 제거

````bash
ssh-keygen -R 43.200.235.247
````

- 이미 만들어져 있는 서버와 같은 인스턴스를 가진 서버가 ssh 키를 공유할 때 ssh 키의 호스트 키를 기록을 지워야한다.

docker compose -f docker-compose-dev.yml run --rm certbot certonly \ --webroot \ --webroot-path=/var/www/certbot \ -d re-caring.duckdns.org \ --email choiyngmin506@gmail.com \ --agree-tos \ --no-eff-email