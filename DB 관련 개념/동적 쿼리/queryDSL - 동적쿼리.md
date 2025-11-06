#인터페이스 #database #mysql 

___

## 개요

- 사용자의 요청에 따라 조건이 달라질때 동적 쿼리를 적용하기 위한 가장 적합한 프레임워크이다.
- **인터페이스와 구현체를 분리해서 구현하도록 한다.(의존성 분리 원칙)**

#### 사용 예시

- 검색 조건이 여러 개이고, 조합이 다양하게 변하는 경우
- 필터링 조건이 존재하는 경우
- 조건이 존재할수도 있고 존재하지 않을 수도 있는 경우

___

## 환경설정

#### build.gradle

```yml
 //querydsl  
    implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'  
    annotationProcessor "com.querydsl:querydsl-apt:5.0.0:jakarta"  
  
}  
  
// [3] Querydsl 설정부  
def generated = 'src/main/generated'  
  
// [4] querydsl QClass 파일 생성 위치를 지정  
tasks.withType(JavaCompile) {  
    options.getGeneratedSourceOutputDirectory().set(file(generated))  
}  
  
// [5] java source set 에 querydsl QClass 위치 추가  
sourceSets {  
    main.java.srcDirs += [ generated ]  
}
```

#### QClass

```java
import hyper.run.domain.user.entity.QUser;  
import hyper.run.domain.inquiry.entity.QCustomerInquiry;

QUser user = QUser.user;
QCustomerInquiry inquiry = QCustomerInquiry.customerInquiry;
JPAQueryFactory queryFactory;
```

- QClass 라이브러리를 상단에 추가해야 한다.
- Q 객체의 인스턴스를 사용한다.

#### 인식 안될 때
`./gradlew clean build`
___

## 코드 예시 요구사항

1. 필드값 (문의일, 문의 상태, 문의자명, 전화번호) 내림차순 정렬
2. 문의일(시작날짜~종료날짜), 문의상태, 문의 유형에 따른 필터링(__모든 값이 필수값 아님__)
3. 키워드 검색(필수값 아님)

## 전체 구조

1. 검색 조건(요청DTO) 와 Pageable 를 파라미터로 받는다.
2. WHERE 조건 생성(사용자가 요청할 수도 있는 조건들)
3. 데이터 조회
4. 전체 개수 조회
5. 결과 반환

___

## 코드 구현

### 1. 조건 필터링 메서드

#### 1-1 날짜 범위 선택

```java
private BooleanExpression dateRange(LocalDateTime startDate,LocalDateTime endDate){
	if(startDate != null && endDate != null){
		return inquiry.createDateTime.between(startDate,endDate);
	}
	return null;
}
```

#### 1-2 문의사항 상태에 따른 검색

```java
private BooleanExpression inquiryStateEq(InquiryState state){
	if(state != null){
		return inquiry.state.eq(state);
	}
	return null;
}
```

#### 1-3 키워드 검색

```java
private BooleanExpression keywordContains(String keyword){
	if(StringUtils.hasText(keyword)){
		return inquiry.user.name.containsIgnoreCase(keyword)
				.or(inquiry.user.email.containsIgnoreCase(keyword))
				.or(inquiry.user.phoneNumber.containsIgnoreCase(keyword));
	}
	return null;
}
```

- Inquiry 와 User 는 **다대일 매핑 관계**

___

### 2. 동적 WHERE() 조건 생성

```java
private BooleanBuilder createWhereClause(InquirySearchRequest request) {  
    BooleanBuilder builder = new BooleanBuilder();  
    builder.and(dateRange(request.getStartDate(), request.getEndDate()))  
            .and(inquiryStateEq(request.getState()))  
            .and(inquiryTypeEq(request.getType()))  
            .and(keywordContains(request.getKeyword()));  
    return builder;  
}
```

- `WHERE` 절에 들어갈 여러 `BooleanExpression` 조건들을 담는 **컨테이너**입니다. `.and()` 나 `.or()` 메서드로 조건들을 조합할 수 있다.

- QueryDSL의 `where()`나 `BooleanBuilder.and()`는 **`null`이 들어오면 해당 조건을 무시**하기 때문에, `if`문 없이도 깔끔하게 동적 쿼리가 완성된다.

___ 

### 3. 동적 정렬 조건 생성

```java
private OrderSpecifier<?>[] getOrderSpecifiers(Sort sort) {  
    List<OrderSpecifier<?>> specifiers = new ArrayList<>();  
  
    // PathBuilder를 통해 동적으로 User 엔티티의 필드 경로를 생성  
    PathBuilder<CustomerInquiry> pathBuilder = new PathBuilder<>(CustomerInquiry.class, "customerInquiry");  
  
    if (sort != null && !sort.isEmpty()) {  
        for (Sort.Order order : sort) {  
            Order direction = order.isAscending() ? Order.ASC : Order.DESC;  
            String property = order.getProperty();  
            // 매핑 되어있는 다른 객체의 필드값을 쓸 때는 밑에 명시해서 정렬해야됨  
            if ("name".equals(property)) {  
                specifiers.add(new OrderSpecifier(direction, inquiry.user.name));  
            }else if("phoneNumber".equals(property)){  
                specifiers.add(new OrderSpecifier(direction,inquiry.user.phoneNumber));  
            }  
            else {  
                specifiers.add(new OrderSpecifier(direction, pathBuilder.get(property)));  
            }  
        }  
    }  
    // 기본 정렬 (만약 아무 정렬 조건도 없다면)  
    if (specifiers.isEmpty()) {  
        specifiers.add(new OrderSpecifier(Order.DESC, inquiry.createDateTime));  
    }  
  
    return specifiers.toArray(new OrderSpecifier[0]);  
}
```

- **`PathBuilder`**: 정렬 기준이 되는 속성(property) 이름이 동적으로 변경될 때, 해당 경로를 안전하게 생성하기 위해 사용된다.

- 조인된 엔티티의 필드(`user.name` 등)로 정렬해야 하는 경우를 별도로 처리하여 정확한 정렬이 가능하게 한다.

- 정렬 조건이 없으면 기본적으로 `createDateTime`을 기준으로 내림차순 정렬을 추가한다.

___

### 4. 데이터 목록 조회

```java
private List<CustomerInquiryResponse> fetchContent(BooleanBuilder whereClause, Pageable pageable) {  
    return queryFactory  
            .select(Projections.constructor(CustomerInquiryResponse.class,  
                    inquiry.id,  
                    inquiry.createDateTime,  
                    inquiry.state,  
                    inquiry.type,  
                    user.name,  
                    user.email,  
                    user.phoneNumber,  
                    inquiry.title,  
                    inquiry.message,  
                    inquiry.answer  
            ))  
            .from(inquiry)  
            .leftJoin(inquiry.user, user)  
            .where(whereClause)  
            .orderBy(getOrderSpecifiers(pageable.getSort()))  
            .offset(pageable.getOffset())  
            .limit(pageable.getPageSize())  
            .fetch();  
}
```

1. **Inquiry 엔티티**에서 응답DTO 객체에 해당하는 필드값들을 가져온다.(select,from)
2. User 엔티티를 left 조인한다.(leftJoin)
3. 이전에 생성한 동적 조건들로 조회한다. (where)
4. 이전에 생성한 동적 정렬 조건들로 정렬한다. (orderBy)
5. 건너뛸 데이터 개수를 불러온다. (offset)
6. 가져올 데이터 개수를 정한다. (limit)
7. 작성한 쿼리를 실행하여 결과를 가져온다. (fetch)

___

### 5. 전체 데이터 개수 조회

```java
private Long fetchTotalCount(BooleanBuilder whereClause){
	Long count = queryFactory
				.select(inquiry.count())
				.from(inquiry)
				.leftJoin(inquiry.user,user)
				.where(whereClause)
				.fetchOne();
	return Objects.requireNonNullElse(count,0L);
}
```

___

### 6. 객체 반환

```java
@Override  
public Page<CustomerInquiryResponse> searchInquiry(InquirySearchRequest inquiryRequest, Pageable pageable) {  
  
    BooleanBuilder whereClause = createWhereClause(inquiryRequest);  
  
    List<CustomerInquiryResponse> content = fetchContent(whereClause, pageable);  
  
    Long total = fetchTotalCount(whereClause);  
  
    return new PageImpl<>(content, pageable, total);  
}
```