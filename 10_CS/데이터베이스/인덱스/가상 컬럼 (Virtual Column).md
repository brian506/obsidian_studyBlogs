DB에서 사용하고 있는 컬럼에 대해서 함수를 이용하여 가공한 상태의 값을 조회하는 일이 생길 경우 인덱스를 어떻게 탈 수 있을까?
이럴 경우 새로운 칼럼 즉, 가상 칼럼을 DB에 추가하여 해당 가상 컬럼을 인덱스로 사용할 수 있다.

두 가지 종류가 있다.

**VIRTUAL (기본값)** 
- 디스크에 저장 안함
- 조회할 때마다 실시간으로 계산
- 인덱스 생성 불가

**STORED**
- 디스크에 실제로 저장
- INSERT/UPDATE 시 자동으로 계산해서 저장
- 인덱스 생성 가능

```sql
-- VIRTUAL (저장 안 됨, 인덱스 불가)
ADD COLUMN notify_date DATE 
GENERATED ALWAYS AS (DATE_SUB(expiry_date, INTERVAL 3 DAY)) VIRTUAL;

-- STORED (저장됨, 인덱스 가능)
ADD COLUMN notify_date DATE 
GENERATED ALWAYS AS (DATE_SUB(expiry_date, INTERVAL 3 DAY)) STORED;

CREATE INDEX idx_notify ON ticket(notify_date); -- STORED만 가능
```

```java
@Column(name = "notify_date", insertable = false, updatable = false) 
private Integer notifyDate;
```
- JPA 에게는 이 값을 읽기만 하도록 제약 조건을 붙여준다. 
	- (`insertable = false, updatable = false)`)