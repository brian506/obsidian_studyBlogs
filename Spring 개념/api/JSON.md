#spring개념 

#  JSON 이란?

데이터를 구조화해서 문자열로 표현하는 방식
프론트엔드와 백엔드 사이, 혹은 서버와 서버 사이에서 데이터를 주고받을 때 주로 사용

### 직렬화와 역직렬화

**직렬화**  : 자바 객체 -> JSON 문자열로 변환

> 내 서버에서 클라이언트에게 응답값을 반환할 때 보내는 방식

**역직렬화** : JSON 문자열 -> 자바 객체 

> 클라이언트가 서버로 요청했을 때 서버에서 반응하는 방식

___

## 🔎 Jackson 의 주요 어노테이션
#### @JsonProperty("")

만약 토스 서버에서 보내는 JSON 필드명 형태가 secret-key 라고 하자,
그렇지만 나는 필드명을 secretKey 로 정하면 두 필드명이 매핑되지 않는 상황이 발생하게 된다.
그럴 때 @JsonProperty 괄호 안에 JSON 필드명으로 매핑해준다.

```java
@JsonProperty("secret-key")
private String secretKey;
```

#### @JsonIgonreProperties(ignoreUnknown = true)

외부 api에서 특정 기능을 수행했을 때 응답하는 값들이 정해져 있다.
이 응답값들 중 내가 원하는 몇개의 응답값만 DB 에 저장하고 싶을 때 나머지 내 서버에 존재하지 않는 JSON 필드는 무시하고 역직렬화하도록 해주는 어노테이션이다.

```java
@JsonIgnoreProperties(ignoreUnkown = true)
public class User {
	private String name;
    }
````

___

## 🔎 ObjectMapper

> 역직렬화와 직렬화를 하는 Jackson 라이브러리이다.

평소에 @RequestBody 혹은 dto 를 쓸 때는 자동으로 JSON 을 파싱해주지만 HttpClient 를 직접 다루게 될 때는 ObjectMapper 를 써야한다.

Json이 이렇게 있다고 가정하자.

```json
"data": { "menuName": "간장 계란 볶음밥", "totalTime": "15분", "ingredients": [ { "name": "계란", "quantity": "2개" }, { "name": "대파", "quantity": "1대" } ], "steps": [ "1. 계란 2개를 볼에 넣고 잘 풀어줍니다.", "2. 팬을 중불로 예열한 후 대파를 볶아 향을 냅니다.", "3. 계란물을 붓고 스크램블 에그를 만듭니다." ] },
```

아래 객체는 컨트롤러에서 받는 응답 객체이다.
```java
public record RecipeResponse(String menuName,  
                             String totalTime,  
                             List<Ingredient> ingredients,  
                             List<String> steps) {  
}

// 응답 객체를 감싸는 공통 응답 객체
public static <S> ApiResponse<S> success(S data) {  
    return new ApiResponse<>(ResultType.SUCCESS, data, null);  
}
```

```java
public RecipeResponse recommendRecipe(List<String> ingredients) throws IOException {  
    String response = clovaService.getRecipeRecommendation(ingredients);  
    return objectMapper.readValue(response, RecipeResponse.class);  
}


public record RecipeResponse(String menuName,  
                             String totalTime,  
                             List<Ingredient> ingredients,  
                             List<String> steps) {  
}
```

`objectMapper.readValue(response, RecipeResponse.class);` 
- `ObjectMapper` 는 위의 JSON의 key 값(menuName 등)으로 value 값을 매핑하여 아래와 같이 value 값을 추출한다.

```json
{

"result": "SUCCESS",

"data": {

"menuName": "간장 돼지고기 볶음",

"totalTime": "15분",

"ingredients": [

{

"name": "돼지고기",

"quantity": "200g"

},

{

"name": "대파",

"quantity": "1대"

},

{

"name": "간장",

"quantity": "4큰술"

}

],

"steps": [

"1. 돼지고기를 한 입 크기로 깍둑 썰어주세요.",

"2. 대파는 송송 썰어주세요.",

"3. 팬에 중불로 예열한 후 기름을 약간 두르고 돼지고기를 넣어 약 5분간 익혀주세요.",

"4. 돼지고기가 거의 익으면 대파와 간장을 넣고 잘 섞어가며 2분간 더 볶아주세요."

]

},
```
