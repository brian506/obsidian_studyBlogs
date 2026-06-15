#spring개념 

## AllArgsConstrucor?

먼저 `AllArgsConstrucor`는 모든 멤버 변수에 대한 생성자를 자동으로 만들어주는 어노테이션이다.
각 필드의 데이터 타입이 같을 때 순서에 무관하게 필드에 대한 생성자를 만들어주므로 순서가 다르게 객체를 생성하게 될 때 주의해야한다.

그러므로 `AllArgsConstrucor`를 지양하고 `Builder`패턴을 이용하여 생성자를 만들어 주는 것이 유지보수에 좋다.

## Builder 패턴 적용

```java
@Entity  
@Getter  
@NoArgsConstructor(access = AccessLevel.PROTECTED)  
@Table(name = "food_category")  
public class FoodCategory extends BaseEntity {  
  
    @Id  
    @GeneratedValue(strategy = GenerationType.IDENTITY)  
    @Column(name = "food_category_id")  
    private Long id;  
  
    @Column(name = "name",nullable = false)  
    private String name;  
  
    @Column(name = "member_id", nullable = false)  
    private String memberId;  
  
    @Builder  
    private FoodCategory(String name, String memberId){  
        this.name = name;  
        this.memberId = memberId;  
    }  
  
    public static FoodCategory create(String name, String memberId){  
        return FoodCategory.builder()  
                .name(name)  
                .memberId(memberId)  
                .build();  
    }  
}
```

- `FoodCategory`라는 객체를 다른 클래스에서 생성할 때, `create()` 으로 생성하게 된다.
- 여기서 `@Builder`가 적용된 생성자를 기반으로 `create()`으로 객체를 생성할 수 있게 되는 것이다.
	- `id` 값은 DB에서 자동 생성이므로 제외
	- `@Builder`로 정의된 생성자가 없으면 `create()` 으로 호출해도 객체 생성 불가

**필요한 필드값과 필드값의 순서 실수 예방을 위해서 `AllArgsConstructor` 어노테이션은 지양하도록 하자**
