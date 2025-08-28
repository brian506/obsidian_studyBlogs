#backend #class #JWT #security #spring 

___

## CustomUserDetails

```java  
@Getter  
public class CustomUserDetails implements UserDetails {  
  
    private Long id;  
    private String email;  
    private String password;  
    private Collection<GrantedAuthority> authorities;  
  
    public CustomUserDetails(Long id,String email,String password,Collection<GrantedAuthority> authorities){  
        this.id = id;  
        this.email = email;  
        this.password = password;  
        this.authorities = authorities;  
    }  
  
    @Override  
    public Collection<? extends GrantedAuthority> getAuthorities() {  
        return authorities;  
    }  
  
    @Override  
    public String getPassword() {  
        return password;  
    }  
  
    @Override  
    public String getUsername() {  
        return email;  
    }  
  
    @Override  
    public boolean isAccountNonExpired() {  
        return true;  
    }  
  
    @Override  
    public boolean isAccountNonLocked() {  
        return true;  
    }  
  
    @Override  
    public boolean isCredentialsNonExpired() {  
        return true;  
    }  
  
    @Override  
    public boolean isEnabled() {  
        return true;  
    }  
}
```

- 사용자의 Credentials 와 Principles 가 담긴 클래스이다.

___

## UserDetailsService

```java
@Service  
@AllArgsConstructor  
public class AdminUserDetailsService implements UserDetailsService {  
  
    private final AdminUserRepository adminUserRepository;  
    private final String ROLE = "ROLE";  
  
    @Override  
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {  
        AdminUser admin = OptionalUtil.getOrElseThrow(adminUserRepository.findByEmail(email),"존재하지 않는 관리자입니다.");  
  
        return new CustomUserDetails(  
                admin.getId(),  
                admin.getEmail(),  
                admin.getPassword(),  
                Collections.singletonList(new SimpleGrantedAuthority(ROLE))  
        );  
    }
```

- 사용자의 권한까지 담긴 정보를 UserDetails 를 통해 불러온다.