
Docker 를 사용하여 하나의 EC2 안에서 컨테이너 두개 (Spring Blue/Spring Green)로 Blue-green 무중단 배포를 해보려고 한다.
![](../images/스크린샷%202026-03-25%2014.52.13.png)

### Blue-Green 배포 시나리오
현재 Blue(8080) 띄워져있음, 그리고 Green(8081) 가정

1. **트리거 및 빌드**
	- 개발자가 PR을 머지하면 새로운 애플리케이션(Green) 코드를 빌드하고 Docker 이미지로 만든다.
2. **새 컨테이너 실행**
	- Git Action이 서버에 접속하여 8081 포트로 Green 컨테이너를 띄운다.
	- 아직 요청은 Blue에게 가는중임
3. **헬스 체크**
	- Spring 애플리케이션이 켜지고 바로 트래픽을 전환하는게 아님
		- 애플리케이션이 DB와의 커넥션 풀을 맺을 때까지 시간이 걸리기 때문
4. **Nginx 방향 전환**
	- Green이 정상적으로 띄워졌으면, Nginx 설정 파일대로 자동으로 Green으로 전환(셸 스크립트 이용)
5. **Nginx Reload 적용**
	- `sudo nginx -s reload` 실행
		- Nginx 가 새로운 설정 파일만 읽어오고 기존에 연결되어 통신된 사용자들의 요청은 끝까지 수행하도록 보장한다.
	- Nginx 를 갱신하여 앞으로의 요청은 Green으로 향한다.
6. **구버전 종료**
	- 트래픽이 완전히 넘어가면, 기존 Blue 컨테이너를 중지하고 삭제하여 리소를 확보한다.


## 배포 과정

###  1. Docker 설치

```bash
curl -fsSL https://get.docker.com | sh # docker 설치 파일 가져와서 다운로드
sudo usermod -aG docker ubuntu 
newgrp docker
```

`sudo usermod -aG docker ubuntu`
- ubuntu 에게 docker 소켓과 연결할 수 있는 권한을 주는 것

#### 왜 ubuntu에게 초기에 소켓 파일에 접근할 수 있는 권한을 주는 걸까?
- 원래 docker 데몬은 root 권한만 사용할 수 있음
- docker 는 `/var/run/docker.sock`로 명령어를 받아서 docker를 움직일 수 있는데, ubuntu가 보내는 명령어로 통해 docker 소켓과 통신해야 하므로 ubuntu에게 root의 많은 권한 중 하나인 docker 소켓 권한을 준 것임

### 2. 파일 초기 생성

```bash
mkdir -p /home/ubuntu/recaring/nginx/conf.d 
mkdir -p /home/ubuntu/recaring/scripts 
mkdir -p /home/ubuntu/recaring/data/certbot/conf 
mkdir -p /home/ubuntu/recaring/data/certbot/www
```

- Nginx 의 파일이 서버 먼저 있지 않으면 Nginx 가 설정을 못 읽게 되어 실행될 때 에러남
>`-p`
> 해당 디렉토리 생성에 필요한 중간 경로도 알아서 다 만들어줌


### 3. 레포에서 초기 파일 복사

```bash
cd /tmp # 임시 디렉토리에서 깃 클론하기 위해서 위치 이동
git clone [깃 경로]

# 파일 복사 
cp re-caring/docker-compose-dev.yml /home/ubuntu/recaring/ 
cp re-caring/nginx/conf.d/myapp.conf /home/ubuntu/recaring/nginx/conf.d/ 
cp re-caring/nginx/conf.d/service-url.inc /home/ubuntu/recaring/nginx/conf.d/ 
cp re-caring/scripts/deploy.sh /home/ubuntu/recaring/scripts/

# 임시 클론 정리
rm -rf /tmp/re-caring
```

- 깃 액션이 첫 배포할 때 서버에서 deploy.sh 셸 스크립트를 호출하는데, 셸 스크립 안에서 사용하는 파일들을 미리 서버에다가 넣어둬야 한다.
- 위에서 디렉토리 처음에 생성할 때와 같은 이유임

### 4. nginx 임시 설정 (초기 SSL 인증서 발급 위해)

참고 : [Certbot (https 인증서)](../네트워크/Certbot%20(https%20인증서).md)

### 5. nginx + certbot 시작
`docker compose -f docker-compose-dev.yml up -d nginx certbot`

## deploy.yml

```yml
name: re-caRing Dev Deploy  
  
on:  
  workflow_dispatch: # 수동  
  pull_request: # 자동  
    branches:  
      - develop  
    types:  
      - closed  
  
jobs:  
  deploy:  
    if: >  
      github.event_name == 'workflow_dispatch' ||  
      github.event.pull_request.merged == true  
    environment: dev  
    runs-on: ubuntu-latest  
    steps:  
      - uses: actions/checkout@v4  
      - name: Setup JDK 21  
        uses: actions/setup-java@v4  
        with:  
          java-version: '21'  
          distribution: 'temurin'  
  
      - name: Setup Gradle  
        uses: gradle/actions/setup-gradle@v5  
  
      - name: Grant execute permisson for gradlew  
        run: chmod +x gradlew  
  
      - name: Test  
        run: ./gradlew clean test  
  
      - name: Build  
        run: ./gradlew clean build -x test  
  
      - name: Set short SHA  
        run: echo "SHORT_SHA=$(echo ${{ github.sha }} | cut -c1-8)" >> $GITHUB_ENV  
  
      - name: Docker Build  
        run: |  
          docker build \  
            -t ${{ vars.DOCKERHUB_USERNAME }}/${{ vars.DOCKERHUB_REPOSITORY }}:latest \  
            -t ${{ vars.DOCKERHUB_USERNAME }}/${{ vars.DOCKERHUB_REPOSITORY }}:${{ env.SHORT_SHA }} .  
  
      - name: Docker Hub Login  
        uses: docker/login-action@v3  
        with:  
          username: ${{ vars.DOCKERHUB_USERNAME }}  
          password: ${{ secrets.DOCKERHUB_TOKEN }}  
  
      - name: Docker Hub Push  
        run: |  
          docker push ${{ vars.DOCKERHUB_USERNAME }}/${{ vars.DOCKERHUB_REPOSITORY }}:latest  
          docker push ${{ vars.DOCKERHUB_USERNAME }}/${{ vars.DOCKERHUB_REPOSITORY }}:${{ env.SHORT_SHA }}  
  
      - name: Fetch secrets from Parameter Store  
        run: |  
          aws ssm get-parameters \  
            --names \  
              /recaring/DB_URL \  
              /recaring/DB_USERNAME \  
              /recaring/DB_PASSWORD \            
              /recaring/JWT_KEY \  
              /recaring/COOLSMS_API_KEY \  
              /recaring/COOLSMS_API_SECRET \  
              /recaring/COOLSMS_SENDER \              
            --with-decryption \  
            --query "Parameters[*].[Name,Value]" \  
            --output text | while read name value; do  
              key=$(basename $name)  
              echo "$key=$value" >> .env  
            done  
        env:  
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}  
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}  
          AWS_DEFAULT_REGION: ${{ secrets.AWS_REGION }}  
  
      - name: Copy files to server  
        uses: appleboy/scp-action@master  
        with:  
          host: ${{ secrets.SSH_HOST }}  
          username: ${{ secrets.SSH_USERNAME }}  
          key: ${{ secrets.SSH_KEY }}  
          source: "docker-compose-dev.yml,.env,nginx/conf.d,scripts"  
          target: /home/ubuntu/recaring  
  
      - name: Deploy To Server  
        uses: appleboy/ssh-action@master  
        with:  
          host: ${{ secrets.SSH_HOST }}  
          username: ${{ secrets.SSH_USERNAME }}  
          key: ${{ secrets.SSH_KEY }}  
          script: |  
            chmod +x /home/ubuntu/recaring/scripts/deploy.sh  
            bash /home/ubuntu/recaring/scripts/deploy.sh  
  
      - name: Notify Discord (dev 배포 성공)  
        if: success()  
        env:  
          WEBHOOK_URL: ${{ secrets.DISCORD_WEBHOOK_URL }}  
          PR_TITLE: ${{ github.event.pull_request.title }}  
          BRANCH: ${{ github.ref_name }}  
          ACTOR: ${{ github.actor }}  
          SHORT_SHA: ${{ env.SHORT_SHA }}  
        run: |  
          curl -X POST "$WEBHOOK_URL" \  
            -H "Content-Type: application/json" \  
            -d "{\"embeds\":[{\"title\":\"✅ Dev 배포 성공\",\"description\":\"$PR_TITLE\",\"color\":5763719,\"fields\":[{\"name\":\"브랜치\",\"value\":\"$BRANCH\",\"inline\":true},{\"name\":\"커밋\",\"value\":\"$SHORT_SHA\",\"inline\":true},{\"name\":\"배포자\",\"value\":\"$ACTOR\",\"inline\":true}]}]}"  
  
      - name: Notify Discord (dev 배포 실패)  
        if: failure()  
        env:  
          WEBHOOK_URL: ${{ secrets.DISCORD_WEBHOOK_URL }}  
          PR_TITLE: ${{ github.event.pull_request.title }}  
          BRANCH: ${{ github.ref_name }}  
          ACTOR: ${{ github.actor }}  
          SHORT_SHA: ${{ env.SHORT_SHA }}  
        run: |  
          curl -X POST "$WEBHOOK_URL" \  
            -H "Content-Type: application/json" \  
            -d "{\"embeds\":[{\"title\":\"❌ Dev 배포 실패\",\"description\":\"$PR_TITLE\",\"color\":15548997,\"fields\":[{\"name\":\"브랜치\",\"value\":\"$BRANCH\",\"inline\":true},{\"name\":\"커밋\",\"value\":\"$SHORT_SHA\",\"inline\":true},{\"name\":\"배포자\",\"value\":\"$ACTOR\",\"inline\":true}]}]}"
```