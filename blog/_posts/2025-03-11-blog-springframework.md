---
layout: post
title: Spring 핵심 개념 및 활용 가이드
description: >
  Spring Framework의 주요 기능과 활용 예제를 한 번에 정리합니다.
  IoC/DI, Validation, 보안(Authentication/Authorization), JPA 매핑, 메시지 변환, 예외 처리 등 필수 개념을 다룹니다.
image: /assets/img/blog/java.png
sitemap: false
---
# Spring 핵심 개념 및 활용 가이드

## IoC/DI와 Bean 관리

* **IoC(Inversion of Control)**: 제어의 역전, 컨테이너가 객체 생성·주입
* **DI(Dependency Injection)**: 의존성 주입, 생성자/Setter/필드 주입
* **Bean 등록 방식**:

    * `@Component`, `@Service`, `@Repository` + `@ComponentScan`
    * `@Configuration` + `@Bean`
    * `@Qualifier` / `@Primary`로 빈 선택

```java
@Configuration
public class AppConfig {
  @Bean
  public UserService userService() {
    return new UserServiceImpl();
  }
}
```

## 입력 검증(Validation)

* **Bean Validation**: `javax.validation` API 활용

    * 어노테이션: `@NotNull`, `@NotBlank`, `@Size`, `@Email` 등
* **컨트롤러 적용**:

    * `@Valid` + `BindingResult` 사용
    * `@ModelAttribute` vs `@RequestBody`

```java
@PostMapping("/users")
public ResponseEntity<?> create(@Valid @RequestBody UserDto dto, BindingResult br) {
  if (br.hasErrors()) {
    // 에러 처리
  }
  // 생성 로직
}
```

## 보안: 인증(Authentication) & 인가(Authorization)

* **세션 vs 토큰**: `HttpSession` / JWT 기반 토큰
* **JWT 구성**: 헤더(Header), 페이로드(Payload), 서명(Signature)
* **필터 기반 처리**: `OncePerRequestFilter` 사용 예

```java
public class JwtAuthFilter extends OncePerRequestFilter {
  @Override
  protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
      throws ServletException, IOException {
    String token = resolveToken(req);
    if (validate(token)) setAuthentication(token);
    chain.doFilter(req, res);
  }
}
```

## JPA: 매핑과 영속성

* **엔티티(Entity)**: `@Entity`, `@Table`, `@Id`, `@GeneratedValue`
* **연관 관계**: `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`
* **패치 전략**: `FetchType.LAZY` vs `EAGER`
* **Cascade & orphanRemoval**: 영속성 전이, 고아 객체 관리

```java
@Entity
public class Order {
  @Id @GeneratedValue
  private Long id;

  @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
  private List<OrderItem> items = new ArrayList<>();
}
```

## 메시지 컨버터(Message Conversion)

* **HttpMessageConverter**: 요청/응답 메시지를 객체로 변환

    * JSON: `MappingJackson2HttpMessageConverter`
    * String, ByteArray 변환기 등
* **컨트롤러 사용**: `@RequestBody` / `@ResponseBody`, `@RestController`

```java
@GetMapping("/data")
public DataDto getData() {
  return new DataDto("example");
}
```

## 예외 처리(Exception Handling)

* **@ControllerAdvice** + `@ExceptionHandler`
* **ResponseEntityExceptionHandler\`** 확장

```java
@ControllerAdvice
public class GlobalExceptionHandler {
  @ExceptionHandler(EntityNotFoundException.class)
  public ResponseEntity<String> handleNotFound(EntityNotFoundException ex) {
    return ResponseEntity.status(HttpStatus.NOT_FOUND).body(ex.getMessage());
  }
}
```

## JPQL 및 Spring Data JPA 활용

* **JPQL**: 객체 지향 쿼리 언어, `@Query`로 정의
* **페이징 & 정렬**: `Pageable`, `Sort`
* **Query Methods**: 메서드 이름으로 쿼리 생성

```java
public interface UserRepository extends JpaRepository<User, Long> {
  List<User> findByStatusOrderByCreatedDateDesc(String status);

  @Query("SELECT u FROM User u JOIN FETCH u.roles WHERE u.id = :id")
  User findWithRoles(@Param("id") Long id);
}
```

> 이 가이드를 통해 Spring의 핵심 기능을 한눈에 파악하고, 실제 프로젝트에 즉시 적용해 보세요!

