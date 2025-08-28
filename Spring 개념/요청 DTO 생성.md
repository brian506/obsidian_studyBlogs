#class #spring 

___

## 개요

사용자가 DTO 로 요청을 보낼 때 빈 DTO 객체의 값 바인딩이 어떻게 이루어지는지 알아보자

___

### @Setter 를 이용한 방법

- 빈 객체를 만들고 값을 하나씩 채워나가는 방법

```java
@NoArgsConstructor  
@AllArgsConstructor  
@Getter  
@Builder  
@Setter  
public class InquirySearchRequest {  
    private LocalDateTime startDate;  
    private LocalDateTime endDate;  
    private String keyword;  
    private InquiryState state;  
    private InquiryType type;  
}

```

**동작 원리:**

1. 프레임워크는 먼저 **`@NoArgsConstructor`**를 이용해 텅 빈 `InquirySearchRequest` 객체를 생성
    
2. 그 다음, HTTP 요청에 들어온 `startDate`, `keyword` 등의 각 필드 값을 찾아서 해당하는 **setter 메소드(`setStartDate(...)`, `setKeyword(...)` 등)를 호출**하여 객체의 내용을 채워 넣는다.

**단점:**

- 다른 객체나 메서드에서 값을 바꿀 수 있으므로 사용자가 요청한 값에 대한 일관성이 확보되지 못한다.

___

### final 필드를 이용한 방법

- 객체가 생성되는 순간 모든 값을 확정되고 이후에는 절대 변하지 않는 방식

```java
@AllArgsConstructor  
@Getter  
@Builder  
public class InquirySearchRequest {  
    private final LocalDateTime startDate;  
    private final LocalDateTime endDate;  
    private final String keyword;  
    private final InquiryState state;  
    private final InquiryType type;  
}
```

- final 필드를 이용하면 생성자나 빌더를 통해서 최초 생성시에 한번만 값을 정한다.
- **객체가 생성된 이후로 값이 변하지 않는 장점이 있다.**

___

### Record 를 이용한 방법

- 데이터 보관이라는 목적에만 집중하는 클래스이다.

- 모든 필드에 대한 **`private final`** 선언
- 모든 필드를 포함하는 **생성자 (`AllArgsConstructor`)** 
- 모든 필드에 대한 **접근자 메소드 (`Getter`)**
- `toString()`, `equals()`, `hashCode()` 메소드

```java
public record InquirySearchRequest( LocalDateTime startDate, LocalDateTime endDate, String keyword, InquiryState state, InquiryType type ) {}
```