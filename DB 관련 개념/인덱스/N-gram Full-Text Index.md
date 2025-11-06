#database #mysql

# 개요
DB 에 저장되어 있는 수많은 문자열로 되어있는 데이터를 `keyword`로 조회할 때 `LIKE`쿼리를 사용하면 풀스캔을 하게 되어 응답 속도가 매우 늦어진다. 
ElasticSearch 를 이용해서 응답 속도를 개선할 수 있지만, 비용이 많이 드는 이유로 Mysql 내에서 지원하는 N-Gram Full-Text-Index 를 공부해서 적용해보고자 한다.


