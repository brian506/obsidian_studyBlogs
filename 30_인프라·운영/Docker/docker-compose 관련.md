#docker 

# 문제 상황

코드를 변경하고 모든 컨테이너를 돌리지 않고 특정 컨테이너만 돌리기 위해 `docker compose -f docker-compose-local.yml up [컨테이너 이름]`으로 컨테이너를 작동시켰다.
하지만 변경된 코드가 적용되지 않고 곅속 예전의 에러가 똑같이 발생했다. 왜 그런지 함 알아보자.

기존의 `./start.sh`에서는 아래와 같은 단계로 빌드했다.
1. `./gradlew clean build -x test` 
	- ** 새 .jar 생성**
2. `docker compose -f docker-compose-local.yml build --no-cache`
	- **예전 이미지를 지우고 위에서 생성된 새로운 .jar 파일로 빌드**
3. `docker compose -f docker-compose-local.yml up -d`
	- **새 이미지로 실행**

개별 컨테이너로 했을 때
`docker compose -f docker-compose-local.yml up [컨테이너 이름]`
- 도커 `--build`옵션이 없어서 기존에 있던 이미지를 재사용하게 되어서 코드를 변경해도 계속 이전의 `.jar`파일을 읽게 되어 에러가 동일하게 나왔던 것이다.

# 결론

Gradle 빌드와 Docker 빌드는 별개이다.

정상적으로 새로운 `.jar` 이미지가 반영되기 위해서는
1. `gradlew build` - 새로운 `.jar` 생성
2. `--build` - DockerFile 다시 빌드
3. 실행