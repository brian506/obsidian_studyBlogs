#네트워크 

사용자의 요청은 80포트(http://) 또는 443포트(https://)로 요청을 할 수 있다.
여기서 https는 http가 암호화된 프로토콜을 의미하고, 이 https 요청을 쓰려면 인증서를 통해 해당 요청을 암호화해야한다.


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
docker compose -f docker-compose-dev.yml run --rm \ --entrypoint certbot certbot \ certonly \ --webroot \ --webroot-path=/var/www/certbot \ -d re-caring.duckdns.org \ --email choiyngmin506@gmail.com \ --agree-tos \ --no-eff-email
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