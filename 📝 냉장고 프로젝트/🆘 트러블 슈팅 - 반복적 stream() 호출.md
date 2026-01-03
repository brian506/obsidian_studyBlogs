
## 상황

식재료 리스트를 조회할 때 각 식재료에 해당하는 카테고리 정보들을 매핑하는 과정에서 반복적 Map 생성으로 인한 성능 저하를 개선

>`Food`와 `FoodCategory` 는 별도의 테이블

## 문제가 있었던 코드

![](../images/ouaLic%20reterd%20Setecteshe.png)![](../images/reteen%20fossuist.streas%20o%20Streamrood.png)

```java 
    SelectedFoodCategories categories = new SelectedFoodCategories(selectedFoodCategoryManager.findByFoodIds(  
            foods.getContent().stream().map(Food::id).toList()  
    ));  
    return foods.map(food -> food.toRegisteredFood(categories.getCategoryIdsByFoodId(food.id())));  

```

현재 코드의 상황으로는 `food.toRegisteredFood()`로의 도메인 변환 과정에서 
모든 foodId 에 대해서 매번 `Map`을 생성하게 된다.
예시 동작)
1. foodId : 1L, categoryId: 1L,2L,3L 
2. Map 에 `key: foodId, value: categoryId `로 저장
3. 객체 변환
4. 다음 foodId 를 위해서 Map 초기화 후 다시 생성
5. 1번 반복

이렇게 되면 key 값을 해당 foodId 에 대해서 한번만 쓰이게 되고, 각 식재료만큼 Map 이 생성되고 지워진다.
매핑할 때마다 Map 을 만들고 지우면 CPU 는 계속 계산 작업을 하게 되고, 불필요한 메모리가 쌓여간다.

### 시간 복잡도

Map 저장 && 조회(생성) : O(N^2)


## 개선된 코드

![](../images/스크린샷%202025-12-22%2015.31.25.png)

```java 
    SelectedFoodCategories categories = new SelectedFoodCategories(selectedFoodCategoryManager.findByFoodIds(  
            foods.getContent().stream().map(Food::id).toList()  
    ));  
    return foods.map(food -> food.toRegisteredFood(categories.getCategoryIdsByFoodId(food.id())));  

```

1. 이렇게 하면 처음 객체 생성 (`new SelectedFoodCategories())` 할때 `selectedFoodCategories` 인스턴스에 `SelectedFoodCategory` 도메인 객체 리스트가 생성자에 저장되고
2. 일급 컬렉션인 `SelectedFoodCategories` 안의 `Map 에 모든 Key(foodId)에 매핑되는 value(categoryId) 가 한번에 저장된다.
-> 이렇게 한번에 객체 생성에 모든 Key 값에 대한 Map 생성이 한번 이루어지게 된다.
3. 마지막으로 새로운 `toRegisteredFood()` 객체 변환 시엔 저장된 Map 에서 stream 으로 foodId 를 기준으로 변환하게 된다.

### 시간 복잡도

Map 저장 : O(N)
데이터 변환 및 조회 : O(1)

## 코드 리뷰

![](../images/•colleer%20(Cat%20tectars-reeungayt.png)