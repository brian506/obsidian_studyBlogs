new 연산자로 객체를 생성할 수 있고, Heap 영역에 할당되고 Stack 영역에서 참조 타입 변수를 통해 데이터에 접근한다.
이때 객체의 타입으로 Mutable, Immutable 의 객체가 있다.

## Mutable
**Immutable** : 객체가 생성된 후에는 **상태를 변경할 수 없는 객체**이다.
            상태를 변경하려고 하면 새로운 객체가 생성된다.

불변 객체 종류 : String, Boolean, Integer, Float, Long 등
- String을 제외하고 원시 타입의 Wrapper 타입이다.

### 불변 객체를 사용하는 이유

1. **스레드 안정성 (Thread Safety) 보장**
	- **멀티 스레드 환경**에서 동기화 문제가 발생하는 이뉴는 공유 자원에 **동시 쓰기 연산** 때문이다.
	- 멀티 스레드 환경에서 동기화 처리 없이 객체 공유가 가능하다.

2. 값의 변경 방지
	- 생성 시점에 값을 생성한 후에 변경할 수 없게 만들어 값 변경을 방지할 수 있다.

> 클래스들은 가변적이어야 하는 매우 타당한 이유가 있지 않는 한 반드시 불변으로 만들어야 한다.
>  만약 클래스를 불변으로 만드는 것이 불가능하다면, 가능한 변경 가능성을 최소화하라

### [String 클래스]

String 객체는 한 번 생성되면 변경할 수 없다. 
문자열을 변경하려면 새로운 String 객체를 생성해야한다.

- **안정성** : String 객체를 공유해도 객체의 상태가 변하지 않는다. 따라서 멀티스레드 환경에서 안전하게 사용할 수 있다.
- **문자열 리터럴** : 문자열 리터럴은 자바에서 [문자열 상수 풀(String Constant Pool)](문자열%20상수%20풀(String%20Constant%20Pool).md) 에서 관리되며, 같은 문자열 리터럴은 재사용된다.
	- ex) "hello" 와 "hello" 는 동일한 객체를 참조
- **성능 문제** : 문자열을 자주 수정하게 되면 매번 String 객체가 생성되므로 성능에 주의해야한다.

## Mutable
**Mutable** : 객체의 **상태를 변경할 수 있는 객체**이다.
		 같은 객체를 계속해서 수정할 수 있다.
		 메모리 내의 동일한 객체 주소에서 내부 필드 값만 바뀐다.

불변 객체 종류 : StringBuilder, ArrayList, HashMap
### [StringBuilder 클래스]

String 객체는 같은 문자열을 새로 생성하지 않고 변경할 수 있다.

- **성능 향상** : 객체를 새로 생성하지 않고 기존 객체를 수정할 수 있기 때문에 메모리 사용을 덜 할 수 있다.
- **단일 스레드 환경** : 단일스레드 환경에서만 사용한다.

### [StringBuffer 클래스]

StringBuffer 는 문자열을 변경할 수 있는 가변클래스이다.

- **스레드 안전** : 멀티스레드 환경에서도 사용 가능하므로 동기화가 적용된다.
- **성능** : 동기화로 인해서 StringBuilder 보다 낮다.


## 불변 객체 구현

- 생성자를 제외하고 객체의 상태를 변경하는 메서드(`setter()`)를 사용하지 않는다.

- 클래스 자체를 `final` 로 선언
	- 상속을 막아 하위 클래스에서 상태가 변하는 것을 방지
##### 스프링에서 클래스에 웬만하면 final 을 안붙이는 이유??
- AOP, Transaction 을 이용할 때 `@Service`나 `@Component`클래스를 상속받아 **가짜 객체(프록시)**를 만든다.
- 프록시 객체를 만들기 위해서는 상속을 해야하기 때문에 스프링의 장점과 JPA 를 사용하지 못하게 되어서 보통 final 클래스는 지양한다.

- 모든 필드를 private final 로 구현한다. 
	- **외부 접근을 차단**하고 한번만 초기화

- **방어적 복사**
	- 생성자에서 가변 객체를 인자로 받을 때, **원본을 그대로 참조하지 않고 복사본**을 만들어 저장한다.
	- 외부에서 객체를 변경해도 내부의 객체는 변경되지 않는다.

- **깊은 복사**
	- Getter 메서드에서 가변 객체를 반환할 때도 복사본을 반환하여 외부에서 내부 데이터가 수정되지 않도록 한다.

### 예시 코드 (가변 클래스)

```java
public final class SelectedFoodCategories { // 1. 클래스 final 선언

    private final List<SelectedFoodCategory> selectedFoodCategories;
    private final Map<Long, List<Long>> categoriesGroupByFoodIds;

    public SelectedFoodCategories(List<SelectedFoodCategory> selectedFoodCategories) {
        // 2. 방어적 복사: 외부 리스트와의 참조를 끊음
        this.selectedFoodCategories = List.copyOf(selectedFoodCategories); 
        this.categoriesGroupByFoodIds = groupByFoodIds();
    }

    public List<Long> getCategoryIdsByFoodId(Long foodId) {
        // 3. 반환 시에도 수정 불가능한 리스트로 반환 (이미 Map 생성 시 처리했다면 불필요)
        return categoriesGroupByFoodIds.getOrDefault(foodId, List.of());
    }

    private Map<Long, List<Long>> groupByFoodIds() {
        return selectedFoodCategories.stream()
                .collect(Collectors.collectingAndThen( // 4. 불변 맵 및 불변 리스트로 수집
                        Collectors.groupingBy(
                                SelectedFoodCategory::foodId,
                                Collectors.mapping(SelectedFoodCategory::foodCategoryId, Collectors.toList())
                        ),
                        Collections::unmodifiableMap
                ));
    }
}
```

`List.copyOf()`
- 생성자에서 만약 `this.selectedFoodCategories = selectedFoodCategories` 로 그대로 인자를 넘긴다고 생각을 해보자.
- 이렇게 되면 **외부에서 인자로 받은 리스트를 그대로 참조**하고 있으므로, 외부에서 `add()` 등 변경 메서드를 실행하면 이 클래스 내부의 데이터도 함께 변해버린다. 왜냐하면 외부의 리스트를 그대로 참조하고 있기 떄문이다.
- **불변성 객체로 만들기 위해서**는 외부 리스트와의 참조는 끊고 **새로운 리스트(복사본)으로 생성**해야한다.
- 이 구성을 **방어적 복사**를 할 수 있다.

`Collections.unmodifiableMap()`
- 반환시에도 수정 불가능한 리스트를 반환하기 위해서는 `Collections::unmodifiableMap`으로 리스트의 불변성을 보장하고 반환한다.
- 이 구성을 **깊은 복사**라고 할 수 있다.

> Stream.toList() - 스트림 형태의 리스트를 불변성으로 반환한다.

> Collectors.toList() - 컬렉션 형태의 리스트를 가변성으로 반환한다.



