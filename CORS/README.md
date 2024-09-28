# WebSocket 및 CORS 관련 트러블슈팅 과정
## 문제 상황
WebSocket을 통해 클라이언트와 서버 간의 통신을 설정하려던 중, CORS 정책 오류와 WebSocket 연결 문제가 발생.
클라이언트에서 provider.html을 로드할 때, 다음과 같은 오류가 발생:
```
Access to XMLHttpRequest has been blocked by CORS policy.
The 'Access-Control-Allow-Origin' header contains multiple values...
```
## 주요 원인
### CORS 설정 충돌:

NGINX와 Spring Boot에서 동시에 CORS 설정을 하고 있어, 중복된 CORS 헤더가 반환됨.
이로 인해 Access-Control-Allow-Origin 헤더에 여러 값이 포함되어, 브라우저가 요청을 차단.
#### Spring Security 설정:

WebSocket 경로 /ws/**에 대해 Spring Security에서 인증을 요구하는 설정이 되어 있었음.
#### NGINX 프록시 설정:

WebSocket 프록시 설정에서 CORS 헤더가 잘못 추가되어, 추가적인 CORS 오류가 발생.
해결 방법
1. Spring Boot CORS 설정 수정
Spring Boot에서 CORS 설정을 중앙에서 관리하기 위해 NGINX에서의 CORS 설정을 제거.
Spring에서 WebSocket 경로 /ws/**를 허용하도록 CORS 및 Security 설정을 수정.
CORS 설정 코드 (Spring Boot):


```
@Override
public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/**")
            .allowedOrigins("http://127.0.0.1:5500")  // 허용할 도메인 명시
            .allowedHeaders("*")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowCredentials(true);  // 인증 정보 허용 시 설정
}
```
2. Spring Security 설정 수정
/ws/** 경로를 인증 없이 접근할 수 있도록 **permitAll()**로 설정.
SecurityConfig 수정 코드:

```
http
    .csrf(csrf -> csrf.disable())
    .authorizeHttpRequests(auth -> auth
        .requestMatchers("/", "/login", "/oauth2/**", "/login/oauth2/code/**", "/public/**", "/ws/**").permitAll()
        .anyRequest().authenticated());
```
3. NGINX 설정 수정
NGINX에서 CORS 관련 설정을 제거하여 Spring Boot에서만 처리되도록 수정.
NGINX 설정 코드:

```
location /ws/ {
    proxy_pass http://localhost:8080/ws/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # WebSocket timeout 설정
    proxy_read_timeout 86400s;
    proxy_send_timeout 86400s;
    
    # CORS 설정 제거
    # add_header 'Access-Control-Allow-Origin' '*';  // 삭제
}
```
4. 브라우저 캐시 문제 확인
브라우저 캐시로 인해 CORS 정책이 올바르게 적용되지 않을 수 있으므로, 캐시를 지우거나 시크릿 창에서 테스트 진행.
최종 결과
CORS 오류 해결 및 WebSocket 정상 연결 확인.
Spring Boot에서의 CORS 및 Security 설정을 통해 중앙 관리 가능하게 됨.
NGINX는 프록시 설정만 남겨두고, CORS 처리는 Spring에서 전담하도록 조정.
참고 사항
Spring Security와 CORS는 반드시 명확하게 구분하여 설정해야 함.
NGINX와 Spring Boot에서 중복된 설정을 피하는 것이 중요.
브라우저 캐시가 문제가 될 수 있으니, 항상 캐시를 지우고 테스트하는 습관이 필요.