#docker #project 

# 도커 기본 명령어

|목적|명령어|설명|
|---|---|---|
|도커 버전 확인|`docker --version`|설치 확인|
|도커 상태 확인|`docker info`|현재 도커 시스템 정보 출력|
|이미지 목록 확인|`docker images`|로컬에 저장된 이미지 목록|
|컨테이너 목록 확인|`docker ps`|실행 중인 컨테이너 목록|
|전체 컨테이너 확인|`docker ps -a`|중지 포함 모든 컨테이너 목록|
|이미지 삭제|`docker rmi <이미지ID>`|이미지 삭제|
|컨테이너 삭제|`docker rm <컨테이너ID>`|컨테이너 삭제|
|중지된 컨테이너 한꺼번에 삭제|`docker container prune`|⛔ 실행 주의|

# 도커 컨테이너 실행 / 제어

| 목적           | 명령어                                | 설명                 |
| ------------ | ---------------------------------- | ------------------ |
| 이미지로 컨테이너 실행 | `docker run -it <이미지명>`            | 터미널 상호작용           |
| 백그라운드 실행     | `docker run -d <이미지명>`             | `-d` = detached 모드 |
| 포트 매핑        | `-p 8080:80`                       | 호스트:컨테이너 포트        |
| 이름 지정 실행     | `--name mycontainer`               | 컨테이너에 이름 부여        |
| 컨테이너 정지      | `docker stop <이름 or ID>`           | 컨테이너 정지            |
| 재시작          | `docker restart <이름 or ID>`        | 컨테이너 재시작           |
| 컨테이너 진입      | `docker exec -it <이름> bash`        | bash 셸 진입          |
| 로그 확인        | `docker logs <이름>`                 | stdout 로그 확인       |
| 몽고DB 접속      | `docker exec -it [컨테이너이름] mongosh` |                    |
#### docker 로 띄운 mysql 에 접속하고 싶을 때
`docker exec -it [컨테이너 이름] mysql -u root -p`


# 도커 compose 명령어

| 목적        | 명령어                                                | 설명              |
| --------- | -------------------------------------------------- | --------------- |
| 컨테이너 실행   | `docker compose up -d`                             | 백그라운드 실행        |
| 로그 확인     | `docker compose logs -f`                           | 실시간 로그 보기       |
| 서비스 중지    | `docker compose down`                              | 네트워크/컨테이너/볼륨 정리 |
| 빌드 및 실행   | `docker compose up --build`                        | 변경사항 반영 재빌드     |
| 특정 파일로 실행 | `docker compose -f docker-compose-local.yml up -d` | 다른 파일로 실행       |
| 빌드 캐시 삭제  | `docker builder prune`                             | 캐시 삭제           |

___

# Volume 이 무엇?

_개념_  
- 컨테이너가 데이터를 저장하는 장소
- 볼륨을 사용하면 컨테이너가 사라져도 데이터는 유지됨

```yml
volumes:  
  - ./wait-for-it.sh:/wait-for-it.sh
```

 `./wait-for-it.sh` : 컨테이너 실행 순서를 제어하기 위한 셸 스크립트
- 서버가 먼저 실행되기 전에 레디스 컨테이너가 실행되어야 DB 연결이 원활하게 되는데, 여기서 컨테이너 실행 순서를 정해줘서 먼저 실행되게 한다.

# Network 가 무엇?

_개념_
- 컨테이너 간 통신을 가능하게 해주는 가상 네트워크
- 기본적으로 도커는 각 컨테이너에 고유한 IP 주소를 부여
- 같은 네트워크 안에 있는 컨테이너끼리는 컨테이너 이름으로 접근 가능

```yml
networks:
  monitoring:
    driver: bridge

services:
  kafka:
    networks:
      - monitoring
  zookeeper:
    networks:
      - monitoring

```

- 서로 다른 서비스끼리 network 를 통해서 통신 가능