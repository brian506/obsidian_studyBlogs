#backend #security #spring 

___

```java
@Configuration  
public class AdminCorsConfig {  
    @Bean  
    public CorsConfigurationSource corsConfigurationSource() {  
        CorsConfiguration configuration = new CorsConfiguration();  
        configuration.setAllowedOrigins(List.of("http://127.0.0.1:5500", "http://localhost:8080","https://d19a6mzn99qmli.cloudfront.net"));  
        configuration.setAllowedMethods(List.of("GET", "POST","PATCH", "PUT", "DELETE", "OPTIONS"));  
        configuration.setAllowedHeaders(List.of("*"));  
        configuration.setAllowCredentials(true);  
        configuration.setExposedHeaders(Collections.singletonList("Authorization"));  
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();  
        source.registerCorsConfiguration("/**", configuration);  
        return source;  
    }  
}
```

- 프론트 및 다른 서버가 해당 서버를 접근할 수 있는 경로를 설정해줘야 한다.