## IAM

> Identify and Access Management, Global Service

- Root Account 
	- 기본적으로 생성됨
	- 생성 후에는 루트 계정을 더 이상 사용해서도, 공유해서도 안됨
	- 대신 사용자를 생성해야함
- Users : 하나의 사용자는 조직 내의 한 사람에 해당
- Groups : 그룹에는 사용자만 배치 가능. 다른 그룹은 포함할 수 없음
- AWS에서는 Least Privilege Principle(최소 권한의 원칙)을 사용
	- 사용자가 꼭 필요로 하는 것 이상의 권한을 주지 않는 것을 의미

### MFA (Multil Factor Authentication)
> password + security device you own
- 루트 계정은 반드시 보호해야함. 
- 계정을 보호하기 위해 비밀번호 외에 디바이스를 사용해서 보호

## IAM Roles

### IAM Security Tools

**IAM Credentials Report (accout - level)**
- 계정에 있는 사용자와 다양한 자격 증명의 상태 포함
**IAM Access Advisor (user - level)**
- 사용자에게 부여된 서비스의 권한과 해당 서비스에 마지막으로 엑세스한 시간을 볼 수 있음
- 해당 도구를 사용하여 어떤 권한이 사용되지 않는지 볼 수 있고, 이를 통해 사용자의 권한을 줄여 최소 권한의 원칙을 지킴


