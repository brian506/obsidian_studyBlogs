
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

