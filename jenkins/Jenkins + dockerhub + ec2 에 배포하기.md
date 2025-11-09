#AWS #Jenkins #docker 

# 개요
처음엔 EC2 서버 내에 Jenkins 를 설치해서 Jenkins 가 docker 이미지 생성, 컨테이너 실행등 모든 역할을 담당하도록 Jenkinsfile 을 구성했지만, 현실적으로 갖고 있는 EC2 사양이 낮아서 너무 많은 시간이 소요되었다.

이번엔 Jenkins 를 로컬에서 실행하고, Jenkins 가 docker image 를 build 하고 docker hub 에 push 하면 EC2 서버는 dockerhub 에서 image 를 pull 해서 배포하는 방식으로 할 것이다.

**순서**
*Jenkins 안에서*
1. ./gradlew build
2. docker image build
3. docker image push(To dockerhub)
*EC2 안에서*
4. docker image pull(From dockerhub)
___
## 배포 준비

### 1. EC2 서버 준비
1. **22(ssh), 9000 포트** 열어주기

2. EC2 서버 내에 **docker 설치**
```shell
// 필수 패키지 업데이트
sudo apt update -y && sudo apt upgrade -y

// docker,docker-compose 파일 설치
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu  # root 없이 실행 가능하게

sudo apt install docker-compose-plugin -y # dockerCli(docker-compose)
# docker-compose 설치 안되면
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose # 권한부여
```

3. **스왑 메모리 생성**(EC2 인스턴스 사양 작을 때 필수) 
```shell
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

4. EC2 서버에서 Dockerhub 로그인
```shell
docker login -u <DOCKER_HUB_USERNAME>
```
- dockerhub 에서 생성한 엑세스토큰으로 비밀번호 입력
___
### 2. Jenkins 로컬 서버 준비
#### 1. Jenkins용 dockerfile & container 준비

```yml
// dockerfile ...

FROM jenkins/jenkins:lts  
USER root  
  
# Docker CLI 설치  
RUN apt-get update && \  
    apt-get install -y ca-certificates curl gnupg lsb-release && \  
    mkdir -p /etc/apt/keyrings && \  
    curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \  
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \    https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null && \  
    apt-get update && \  
    apt-get install -y docker-ce-cli git && \  
    apt-get clean  
  
USER jenkins
________________________________________________________________________________

// docker-compose-jenkins.yml ...
services:  
  jenkins:  
    build:  
      context: . # 현재 jenkins 폴더 안에 dockerfile, .yml 있음
      dockerfile: Dockerfile  
    container_name: jenkins  
    ports:  
      - "9000:8080"  
      - "50000:50000"  
    volumes:  
      - jenkins_home:/var/jenkins_home  
      - /var/run/docker.sock:/var/run/docker.sock  
    restart: unless-stopped  
  
volumes:  
  jenkins_home:
```

**-> Docker out of Docker(DooD) 방식
- Jenkins 컨테이너가 **DockerCli 을 사용**하여 *Docker 소켓*을 통해 ***호스트 엔진에게 컨테이너 실행 명령***을 내리는 구조
- Jenkins Container 가 도커 명령어(docker build, docker-compose up 등)을 실행하기 위해서 호스트 도커 데몬에 접근해야함
- 직접 도커 데몬을 실행하기 위해서는 소켓을 통해서 접근하여 도커 자체를 실행시켜줌
- **Jenkins Container 안에서 호스트에서 돌아가는 컨테이너들을 제어할 수 있게됨**

 >*Jenkins.Dockerfile*
- 디폴트로 있는 Jenkins 이미지를 poll(`FROM jenkins/jenkins:lts`)
- docker, git 명령어를 사용할 수 있도록 DockerCli 를 다운
- root 유저가 아닌 Jenkins 유저로 다시 전환

 >docker-compose-jenkins.yml*
```yml
volumes:  
      - jenkins_home:/var/jenkins_home  
      - /var/run/docker.sock:/var/run/docker.sock  
```
- `~/jenkins_home` : Jenkins 설정 정보를 담은 volume
- `~/docker.sock`: 호스트 엔진 도커에게 명령을 낼 수 있도록 하는 socket
	- 예: Jenkinsfile에서 `docker build` 하면, 실제 EC2(호스트)에서 컨테이너가 빌드됨

#### 2. Jenkins 서버 설정

1. `http://localhost:9000`으로 Jenkins 서버 접속
2. **Jenkins 서버 비밀번호 확인** : `sudo docker logs jenkins` 명령어로 나온 비밀번호로 입력
3. **Jenkins Pipeline Job 생성** 
4. **Jenkins Credentials 입력** 
	- *Git* : Secret-text 로 PAT 입력
	- *.env 파일* : Secret-file 로 붙여넣기
	- *EC2 ssh 접속* : SSH-Username with private key 로 입력
		- `cat <시크릿키이름>.pem` 으로 private key 확인
	- *Dockerhub 인증* : Secret-text 로 Access token 입력
___
## 배포 시작

### 순서
1. Build Jar at Jenkins
2. Build Docker Image at Jenkins
3. Push Docker Image to DockerHub
4. Deploy to EC2
#### 1. Build Jar
자바 소스코드 파일을 컴파일하여 실행 가능한 .jar 파일로 생성

```shell
stage('Build Jar') {
    steps {
        sh 'chmod +x ./gradlew || true'
        sh './gradlew :eureka-server:clean :eureka-server:build -x test'
        // ... (다른 서비스들도 동일)
    }
}
```
- `sh 'chmod +x ./gralew || true` : Git 에서 파일을 받아오고 실행하기 위해 권한 부여
- `sh './gradlew : eureka-server:clean :eureka0server:build -x test'` : gradle 을 사용하여 각 서비스를 개별적으로 빌드 (테스트없이 빌드하여 속도 개선)

#### 2. Build Docker Image
.jar 파일을 이용하여 Docker Image 생성

```shell
stage('Build Docker Images') {
    steps {
        sh """
            docker build --platform linux/amd64 -t $DOCKER_HUB_USER/eureka-server:1.0.3 ./eureka-server
            ...
        """
    }
}
```
- docker-compose.yml 에 있는 `사용자명/이미지명:버전` 와 같이 Docker Image Build
- `--platform linux/amd64 -t` 
	- EC2 CPU 사양은 보통 `x86` 인데 mac 은 ARM 이므로 이미지가 ARM 으로 빌드됨
		-> EC2 에 빌드할 때 위와 같이 `linux/amd64`사양으로 빌드해야됨(AMD 와 ARM 차이)

#### 3. Push Docker Image
Jenkins 서버에서 만들어둔 Image를 DockerHub 으로 Push

```shell
stage('Push Docker Images') {
    steps {
        sh """
            echo $DOCKER_HUB_PASS | docker login ... --password-stdin
            docker push $DOCKER_HUB_USER/eureka-server:1.0.3
            ...
        """
    }
}
```
- Docker 에 로그인하여 만들어둔 Docker Image 를 전송

#### 4. Deploy to EC2
EC2 에 원격으로 접속하여 DockerHub 에서 Image 를 내려받고 설치

```shell
stage('Deploy to EC2') {  
            steps {  
                echo '=== Deploy on EC2 (scp + ssh) ==='  
  
                withCredentials([  
                    file(credentialsId: 'env-file', variable: 'ENV_FILE'),  
                    sshUserPrivateKey(credentialsId: 'ec2-key', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')  
                ]) {  
                    sh '''#!/bin/bash  
set -euo pipefail  
HOST=$(echo "$EC2_HOST" | cut -d'@' -f2)  
USER=$SSH_USER  
  
echo "[1/4] connect test"  
ssh -i "$SSH_KEY" -o IdentitiesOnly=yes -o StrictHostKeyChecking=no "$USER@$HOST" 'id && whoami'  
  
echo "[2/4] prepare dir on remote"  
ssh -i "$SSH_KEY" -o IdentitiesOnly=yes -o StrictHostKeyChecking=no "$USER@$HOST" '  
    set -e    mkdir -p /home/ubuntu/app    ls -ld /home/ubuntu/app'  
  
echo "[3/4] upload files via scp (no sudo)"  
scp -i "$SSH_KEY" -o IdentitiesOnly=yes -o StrictHostKeyChecking=no \  
    "$ENV_FILE" "$USER@$HOST":/home/ubuntu/app/.env.prodscp -i "$SSH_KEY" -o IdentitiesOnly=yes -o StrictHostKeyChecking=no \  
    docker-compose.yml "$USER@$HOST":/home/ubuntu/app/docker-compose.yml  
echo "[4/4] deploy with docker compose"  
ssh -i "$SSH_KEY" -o IdentitiesOnly=yes -o StrictHostKeyChecking=no "$USER@$HOST" '  
   set -e       export PATH=$PATH:/usr/local/bin:/usr/libexec/docker/cli-plugins       cd /home/ubuntu/app       docker compose pull       docker compose up -d --remove-orphans       docker image prune -af || true'
```

1. **[1/4] connect test** : 인증키를 가지고 `ssh` 명령어로 EC2 에 원격접속
2. **[2/4] prepare dir on remote** : EC2 서버에 `mkdir -p /home/ubuntu/app`경로를 *생성*하고 해당 파일 경로로 *이동*
3. **[3/4] upload files via scp** :  EC2 서버에 `scp`명령어로 *docker-compose.yml 파일과 .env 파일을 해당 경로에 복사*한다.
4. **[4/4] deploy with docker compose** : 해당 경로로 이동하여
	 1. docker-compose.yml *파일을 pull* 하고 
	 2. docker compose up 으로 *컨테이너를 실행한다.* 
	 3. 불필요한 이미지는 정리한다.
