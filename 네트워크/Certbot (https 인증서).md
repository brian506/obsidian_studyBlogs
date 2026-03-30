#네트워크 

사용자의 요청은 80포트(http://) 또는 443포트(https://)로 요청을 할 수 있다.
여기서 https는 http가 암호화된 프로토콜을 의미하고, 이 https 요청을 쓰려면 인증서를 통해 해당 요청을 암호화해야한다.

## 브라우저 - 서버 요청 / 응답 방식

사용자의 요청이 80포트로 왔을 때, 해당 요청을 443포트로 리다이렉트 해야한다.
https 요청을 하기 전에 **TLS 핸드셰이크**라는 과정이 일어난다.

1. 브라우저 -> 서버 : 
	- 지원하는 TLS 버전 전송
	- 브라우저가 갖고 있는 **암호화 방식 목록** 전송
	- Client 난수 값 전송
2. 서버 -> 브라우저 : 
	- 서버는 브라우저의 **암호화 방식과 암호화 알고리즘**을 선택하여 전송
	- Server 난수 값 전송
	- **인증서** (도메인명, 공개키, 유효기간, 서명) 전송
3. 브라우저 인증서 검증
	- 인증서에 서명한게 신뢰할 수 있는 인증기관인가?
	- 도메인이 일치하는가?
	- 유효기간이 지나지 않았는가?
4. 브라우저 -> 서버 :
	- pre-master secret 난수값 생성 후 **서버의 공개키로 암호화**해서 전송
5. 서버 
	- 자신의 **개인 키로 복호화해서 pre-master secret** 획득

브라우저와 서버는 서로 Sever, Client 난수 값, pre-master secret 값을 가짐
또한, 이 세 값을 조합해서 얻은 **Session Key를 생성**

이후 서로의 **Http 요청은 이 Session Key(대칭 키)로 암호화되어 통신**

**TLS HandShake**
- 비대칭 키 (공개 키/ 개인 키) 사용
   -> pre-master secret을 안전하게 전달하기 위해

**실제 HTTP 통신**
- 대칭 키 (Session Key) 사용
	-> 실제 데이터를 암호화/복호화 하기 위해

이 과정을 하기 위해 Certbot은 인증서를 생성해주고, 인증서는 해당 서버의 신원 증명, 키 교환을 하기 위해 쓰인다.

## Certbot 적용하기

Certbot을 통해서 인증서를 발급 받는 방법에는 총 4가지 있다.
그 중에 나는 webroot 방식으로 사용하고자 한다.

**Webroot** : 사이트 디렉토리 내에 인증서 유효성을 확인할 수 있는 파일을 업로드하여 인증서를 발급하는 방법
- 실제 작동하고 있는 웹서버의 특정 디렉토리의 특정 파일 쓰기 작업을 통해서 인증
- nginx를 중단할 필요 없는 장점을 지님
- 하나의 인증 명령에 하나의 도메인 인증서만 발급 가능한 단점을 지님

Webroot 방식으로 컨테이너를 직접 띄워서 관리하도록 한다.

### certbot
```yml
certbot:  
  image: certbot/certbot  
  container_name: certbot  
  restart: unless-stopped  
  volumes:  
    - ./data/certbot/conf:/etc/letsencrypt 
    - ./data/certbot/www:/var/www/certbot  
  entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
```

- `./data/certbot/conf:/etc/letsencrypt `
	- 발급된 SSL 인증서와 Cerbot 설정 파일이 저장되는 곳
	- 나중에 웹 서버가 이 폴더를 참조하여 HTTPS를 킨다.
- `./data/certbot/www:/var/www/certbot`
	- 도메인이 진짜 맞는 지 확인하기 위해 임시 파일을 생성하는 webroot 폴더
- enrtypoint 명령어
	- 12시간마다 certbot 을 갱신하여 인증서를 갱신한다.

### Nginx
```yml
nginx:  
  image: nginx:latest  
  container_name: nginx  
  restart: unless-stopped  
  environment:  
    TZ: Asia/Seoul  
  ports:  
    - "80:80"  
    - "443:443"  
  volumes:  
    - ./nginx/conf.d:/etc/nginx/conf.d  # Nginx 설정 파일 폴더
    - ./data/certbot/conf:/etc/letsencrypt  # Certbot이 발급한 인증서를 Nginx가 읽기 위해 공유
    - ./data/certbot/www:/var/www/certbot  
  command: "/bin/sh -c 'while :; do sleep 6h & wait $${!}; nginx -s reload; done & nginx -g \"daemon off;\"'"
```

- command 명령어 
	- 6시간마다 Nignx 를 리로드하여, 갱신된 새 인증서를 적용

### myapp.conf
```conf
server {  
    listen 80;  
    server_name re-caRing.com  
  
     location /.well-known/acme-challenge/ {  
        root /var/www/certbot;  
     }  
  
     location / {  
        return 301 https://$host$request_uri;  
     }  
}  
  
server {  
    listen 443 ssl;  
    server_name re-caRing.com  
      
    ssl_certificate     /etc/letsencrypt/live/re-caRing.com/fullchain.pem;  
    ssl_certificate_key /etc/letsencrypt/live/re-caRing.com/privkey.pem;  
}
```

### 최초 인증서 발급 (한 번만 수행)

원래 파일 설정에는 443 블록이 있는데, 처음엔 SSL 인증서 파일이 없어서 에러가 난다.
그래서 인증서 받을 때까지만 HTTP 전용 임시 설정으로 바꿔야함
인증서 받고 나서 원래 스크립트로 복구해줘야함

```bash
cat > /home/ubuntu/recaring/nginx/conf.d/myapp.conf << 'EOF' 
server { listen 80; server_name re-caRing.com; 

location /.well-known/acme-challenge/ { root /var/www/certbot; } 

location / { return 200 'ok'; } 
} 
EOF
```

```bash
docker compose -f docker-compose-dev.yml run --rm certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  -d re-caRing.com \
  --email your-email@example.com \
  --agree-tos \
  --no-eff-email
```
- `run --rm` : 명령 실행 후 컨테이너 자동 삭제
- `certonly` : 인증서만 발급 (nginx 설정 건드리지 않음)
- `--webroot` : 실행 중인 nginx를 통해 도메인 소유권 검증하는 방식
- `--webroot-path` : certbot이 인증 파일을 놓을 경로 (nginx가 이 경로를 서빙 중)
- `-d` : 인증서 발급할 도메인
- `--agree-tos`, `--no-eff-email` : 약관 동의, 이메일 수신 거부 자동 처리

**발급 확인**
```bash
ls /home/ubuntu/recaring/data/certbot/conf/live/re-caRing.com/
# fullchain.pem, privkey.pem 이 보이면 성공
```

**nginx reload**
`docker exec nginx nginx -s reload`