### 1. IAM 역할 생성
- 권한 정책에서 `AmazonSSMManagedInstanceCore` 추가
- IAM 역할 업데이트
### 2. SSM Agent 확인
- Systems Manager -> Fleet Manager에서 EC2 등록된 거 확인

### 3. Github Actions IAM 유저에 권한 추가
사용자 권한 정책에서 새로운 인라인 정책 추가
```
{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "ec2:DescribeInstances"
        ],
        "Resource": "*"
      },
      {
        "Effect": "Allow",
        "Action": [
          "ssm:SendCommand"
        ],
        "Resource": [
          "arn:aws:ssm:*:*:document/AWS-RunShellScript",
          "arn:aws:ec2:ap-northeast-2:757052694924:instance/i-0c3a7ea59344d10b0"
        ]
      },
      {
        "Effect": "Allow",
        "Action": [
          "ssm:GetCommandInvocation"
        ],
        "Resource": "*"
      },
       {
    "Effect": "Allow",
    "Action": ["ssm:StartSession"],
    "Resource": [
      "arn:aws:ec2:ap-northeast-2:757052694924:instance/i-0c3a7ea59344d10b0",
      "arn:aws:ssm:*:*:document/AWS-StartSSHSession",
      "arn:aws:ssm:*:*:document/AWS-StartPortForwardingSession",
      "arn:aws:ssm:*:*:document/*"
    ]
  }
    ]
  }
```

## 로컬 터미널에서 SSM으로 EC2 접속하는 방법

1. brew install --cask session-manager-plugin
2. AWS CLI 자격증명 확인
	- aws sts get-caller-identity
		- IAM 사용자 이름으로 되어있어야함
3. 접속
	- aws ssm start-session --target [인스턴스 ID]
4. ubuntu 유저로 변경
	- sudo su - ubuntu

## 그라파나 접속

aws ssm start-session \
    --target i-0c3a7ea59344d10b0 \ # 인스턴스ID
    --document-name AWS-StartPortForwardingSession \
    --parameters '{"portNumber":["3000"],"localPortNumber":["3000"]}'

## PostgreSQL 접속

aws ssm start-session \
    --target i-0c3a7ea59344d10b0 \
    --document-name AWS-StartPortForwardingSession \
    --parameters '{"portNumber":["5432"],"localPortNumber":["5432"]}'

  터미널 열어두고 → DataGrip에서 Host: localhost, Port: 5432로 직접 연결 (SSH 터널 설정 없이)