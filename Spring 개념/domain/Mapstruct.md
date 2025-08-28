#spring개념 

- Java 객체 변환을 자동화하는 라이브러리
- Entity <-> DTO 변환을 쉽게 처리


```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserMapper INSTANCE = Mappers.getMapper(UserMapper.class);
    
    @Mapping(target = "password", source = "password1")
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "role", ignore = true)
    @Mapping(target = "absencePoint", ignore = true)
    @Mapping(target = "boards", ignore = true)
    @Mapping(target = "comments", ignore = true)
    @Mapping(target = "refreshToken", ignore = true)
    @Mapping(target = "profileImageUrl", ignore = true)
    User toEntity(SignupRequestDto signupRequestDto);
````

### Dto -> entity 일 때

source : 매핑이 될 객체, getter 필요(dto)
target : 매핑할 객체, 생성자와 setter 필요(User)
- User 클래스와 Dto 클래스에서 이름이 같은 필드값들은 자동으로 매핑

1. Dto 의 password1 을 User 의 password 필드값에 매핑
2. Entity 의 id 는 자동생성되므로 "ignore = true"
3. profileImageUrl 은 서비스계층에서 따로 처리하므로 "ignore = true"
4. Entity 에만 있고 DTO 에는 없는 필드값들은 "ignore = true" 하는 것이 좋음
5. Entity 에는 없고 DTO 에만 있는 필드값들은 MapStruct 가 알아서 무시해줌

❇️ Update 관련 매핑(Entity 업데이트)

- @MappingTarget 을 사용하여 기존 엔티티를 수정할 수 있다.
```JAVA
 	@Mapping(target = "id", ignore = true)
    void updateFromDto(UserDto dto, @MappingTarget User entity);
 ````
 - id 는 변경하지 않도록 설정한다.
 
🤷🏻 void 반환이 적합한 이유?
-> 새로운 객체를 수정하지 않고 기존 객체를 수정하므로 반환값이 필요없다.