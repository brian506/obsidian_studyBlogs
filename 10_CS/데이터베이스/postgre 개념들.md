
```
# 접속
psql -h [호스트] -U [유저] -d [DB명]

psql -h recaring-db.clgoyi6u008c.ap-northeast-2.rds.amazonaws.com -p 5432 -U recaring -d recaring

# 종료
\q

\l                  -- DB 목록 (MySQL: show databases;)
\c [DB명]           -- DB 선택 (MySQL: use DB명;)
\dt                 -- 테이블 목록 (MySQL: show tables;)
\d [테이블명]        -- 테이블 구조 (MySQL: desc 테이블명;)

-- 테이블 데이터 전체 삭제 (AUTO_INCREMENT 초기화 포함) 
TRUNCATE TABLE members; 

-- 컬럼 추가 
ALTER TABLE members ADD COLUMN nickname VARCHAR(20);

-- 컬럼 삭제 
ALTER TABLE members DROP COLUMN nickname;

-- 인덱스 확인 
\di
```

## postgreSQL의 MVCC 내부 동작

postgreSQL은 동시성 제어를 위해 READ COMMITTED의 격리수준을 갖는다.

postreSQL의 모든 테이블엔 우리가 만든 컬럼 외에, 시스템이 관리하는 숨겨진 컬럼 두 개가 존재

- **xmin (생성 시간표)** : 데이터를 INSERT(생성)한 트랜잭션의 ID 기록
- **xmax(수정/삭제 시간표)** : 데이터를 DELETE나 UPDATE한 트랜잭션의 ID 기록
	- UPDATE는 기존 데이터를 고치는 게 아니라, 기존 데이터에 xmax(삭제 표시)를 기록
	- INSERT는 xmin과 함께 데이터를 기록

### 동작 시나리오

#### 상황1 : A가 트랜잭션 커밋 안했을 때 

1. A가 트랜잭션을 열고 UPDATE 실행
	- 기존 데이터의 **xmax**에 "A의 트랜잭션 ID" 기록 - version1
	- 새로운 데이터의 **xmin**에 "A의 트랜잭션 ID" 기록 - version2

2. B가 조회 요청 : B에게 커밋된 데이터만 볼 수 있는 스냅샷 버젼들을 제공
	- version2 제공했을 때 : A의 트랜잭션 ID가 **xmin**인 데이터를 받지만, 커밋 전이므로 무시
	- version1 제공했을 떄 ; A의 트랜잭션 ID가 **xmax**인 데이터를 받음.
		- 삭제 표시(xmax)가 있지만, 아직 커밋 전이므로 version1을 읽음

**결과** : A가 락을 걸고 데이터를 수정하고 있어도, B는 삭제 표시를 통해 과거 데이터를 읽음

#### 상황2 : A가 트랜잭션 커밋 했을 때 

- B가 다시 조회를 했을 때 수정본이 커밋 됐으므로 version2를 읽는다.
- 여기서 삭제 표시가 남겨진 예전 데이터(version1)은 이제 쓰이지 않게됨 -> **Dead Tuple**
	- postgre 에서는 record가 아닌 tuple이라고 명칭
## VACUUM

> 위에서 발생한 Dead Tuple을 없애버리는 역할

- Dead Tuple을 사용하고 있던 공간을 비움
- 불필요한 Dead Tuple을 치워주기 때문에, 디스크를 읽는 양이 줄어들어 쿼리 성능 향상

### 일반 VACUUM (Autovacuum)

- **특징** : 테이블에 락을 걸지 않고 백그라운드로 실행
- 데드 튜플을 찾아 재사용 가능 상태로 만듬
	- 디스크 파일 자체의 물리적인 크기를 줄이진 않음

### VACUUM FULL

- **특징** : 테이블에 배타적 락을 걸기 때문에, 실행되는 동안 테이블에 읽기/쓰기 제한 
-  살아있는 데이터만 모아서 새로운 파일로 다시 쓰기 때문에, 물리적 크기 줄여줌