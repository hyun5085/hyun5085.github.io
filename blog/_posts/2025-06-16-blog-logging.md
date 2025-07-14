---
layout: post
title: Interceptor vs AOP 스터디 노트
description: >
  Spring Boot에서 HTTP 레벨과 메서드 레벨 공통 처리를 위해
  Interceptor와 AOP를 언제, 어떻게 사용하면 좋은지 정리한 스터디 노트입니다.
image: /assets/img/blog/logging.png
sitemap: false
---

# Interceptor vs AOP 스터디 노트

Spring Boot 애플리케이션에서 **Interceptor**와 \*\*AOP(Aspect-Oriented Programming)\*\*를
적절히 활용하기 위한 개념, 예시, 장단점, 그리고 사용 시나리오를 정리합니다.

---

## 1. Interceptor

### 1.1 정의

* Spring MVC의 `HandlerInterceptor` 인터페이스를 구현하여
  컨트롤러(Handler) 호출 전·후에 공통 로직을 삽입하는 컴포넌트
* `WebMvcConfigurer.addInterceptors()`를 통해 등록

### 1.2 주요 메서드

| 메서드                 | 실행 시점               | 용도                    |
| ------------------- | ------------------- | --------------------- |
| `preHandle()`       | 컨트롤러 호출 이전          | 인증·인가, 요청 파라미터 검사, 로깅 |
| `postHandle()`      | 컨트롤러 실행 후, 뷰 렌더링 이전 | 모델 조작, 응답 커스터마이징      |
| `afterCompletion()` | 뷰 렌더링 후             | 리소스 정리, 예외 로깅         |

### 1.3 예시 코드

```java
public class LoggingInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        System.out.println("[Request] " + req.getMethod() + " " + req.getRequestURI());
        return true;
    }
    @Override
    public void afterCompletion(HttpServletRequest req, HttpServletResponse res, Object handler, Exception ex) {
        System.out.println("[Response] status=" + res.getStatus());
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoggingInterceptor())
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/auth/**");
    }
}
```

### 1.4 장단점

* **장점**

  * HTTP 요청/응답 흐름 제어에 최적화
  * URL 패턴별 공통 로직 적용 용이
* **단점**

  * Spring MVC에 종속적
  * 메서드 실행 내부 로직에는 적용 불가

---

## 2. AOP (Aspect-Oriented Programming)

### 2.1 정의

* 핵심 비즈니스 로직(core concern)과 횡단 관심사(cross-cutting concern)를 분리하는 기법
* Spring AOP는 프록시 기반으로 메서드 실행 전·후, 예외 발생 시점 등에 Advice 적용

### 2.2 주요 개념

| 용어       | 설명                                  |
| -------- | ----------------------------------- |
| Advice   | 실행할 공통 로직 (Before, After, Around 등) |
| Pointcut | Advice를 적용할 메서드 집합을 지정하는 표현식        |
| Aspect   | Pointcut + Advice를 하나로 묶은 모듈        |

### 2.3 예시 코드

```java
@Aspect
@Component
public class PerformanceAspect {
    @Around("execution(* com.example.service.*.*(..))")
    public Object measure(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();
        long elapsed = System.currentTimeMillis() - start;
        System.out.println(pjp.getSignature() + " executed in " + elapsed + "ms");
        return result;
    }
}
```

### 2.4 장단점

* **장점**

  * 메서드 실행 전·후, 예외 처리 등 세밀한 공통 로직 적용 가능
  * 서비스, DAO 등 모든 Bean에 적용할 수 있음
* **단점**

  * 포인트컷 표현식 복잡도 증가
  * 프록시 한계(`this.method()` 호출에 미적용)

---

## 3. 언제 사용하면 좋은가?

| 사용 시나리오           | Interceptor               | AOP                           |
| ----------------- | ------------------------- | ----------------------------- |
| **HTTP 요청/응답 제어** | 로그인 세션 확인, CORS 처리, 요청 로깅 | -                             |
| **URL 패턴별 필터링**   | 특정 엔드포인트 전용 기능 적용         | -                             |
| **메서드 실행 시간 측정**  | -                         | 비즈니스 로직 성능 모니터링               |
| **트랜잭션 관리**       | -                         | `@Transactional` 붙인 메서드 전후 처리 |
| **예외 처리 & 로깅**    | 기본 예외 로깅                  | 서비스/DAO 레벨 예외 세부 로깅           |
| **서비스 확장성**       | Spring MVC 웹 레이어 한정       | 애플리케이션 전반(웹/서비스/DAO) 적용       |

---

> **정리:**
>
> * HTTP 레벨 공통 처리: **Interceptor**
> * 메서드 레벨 공통 처리: **AOP**
    >   두 기술을 목적에 맞게 조합하면, 응집도 높은 코드와 재사용 가능한 모듈을 설계할 수 있습니다.
