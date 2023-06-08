# proxy
해당 문서 출처는 [스프링 핵심 원리 - 고급편](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B3%A0%EA%B8%89%ED%8E%B8/dashboard) 기반으로 개발하는 Spring Batch 로 작성되었습니다. 
## 목차
* **[프록시 패턴과 데코레이터 패턴](#프록시-패턴과-데코레이터-패턴)**
  * **[예제 프로젝트 만들기 v1](#예제-프로젝트-만들기-v1)**
  * **[예제 프로젝트 만들기 v2](#예제-프로젝트-만들기-v2)**

## 프록시 패턴과 데코레이터 패턴
### 예제 프로젝트 만들기 v1
다양한 상황에서 프록시 사용법을 이해하기 위해 다음과 같은 기준으로 기본 예제 프로젝트를 만들어보자.   

__예제는 크게 3가지 상황으로 만든다.__   
- v1. 인터페이스와 구현 클래스 : 스프링 빈으로 수동 등록
- v2. 인터페이스 없는 구체 클래스 : 스프링 빈으로 수동 등록
- v3. 컴포넌트 스캔으로 빈 자동 등록

실무에서는 스프링 빈으로 등록할 클래스는 인터페이스가 있는 경우도 있고 없는 경우도 있다. 그리고 스프링 빈을 수동으로 직접 등록하는 경우도 있고, 컴포넌트 스캔으로 자동으로 등록하는 경우도 있다. 이런 다양한 케이스에 프록시를 어떻게 적용하는지 알아본다.   

__v1. 인터페이스와 구현 클래스 : 스프링 빈으로 수동 등록__   
지금까지 보아왔던 `Controller`, `Service`, `Repository` 에 인터페이스를 도입하고, 스프링 빈으로 수동 등록해보자.   

__OrderRepositoryV1__   
```java
package hello.proxy.app.v1;

public interface OrderRepositoryV1 {
    void save(String itemId);
}
```
__OrderRepositoryV1Impl__    
```java
package hello.proxy.app.v1;

public class OrderRepositoryV1Impl implements OrderRepositoryV1 {

    @Override
    public void save(String itemId) {
        //저장 로직
        if(itemId.equals("ex")) {
            throw new IllegalStateException("예외 발생!");
        }
        sleep(1000);
    }

    private void sleep(int millis) {
        try {
            Thread.sleep(millis);
        } catch(InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
__OrderServiceV1__   
```java
package hello.proxy.app.v1;

public interface OrderServiceV1 {
    void orderItem(String itemId);
}
```
__OrderServiceV1Impl__   
```java
package hello.proxy.app.v1;

public class OrderServiceV1Impl implements OrderServiceV1 {

    private final OrderRepositoryV1 orderRepository;

    public OrderServiceV1Impl(OrderRepositoryV1 orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Override
    public void orderItem(String itemId) {
        orderRepository.save(itemId);
    }
}
```
__OrderControllerV1__   
```java
package hello.proxy.app.v1;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.ResponseBody;

@RequestMapping // 스프링은 @Controller 또는 @RequestMapping 이 있어야 스프링 컨트롤러로 인식
@ResponseBody
public interface OrderControllerV1 {

    @GetMapping("/v1/request")
    String request(@RequestParam("itemId") String itemId);

    @GetMapping("/v1/no-log")
    String noLog();
}
```
- `@RequestMapping` : 스프링MVC는 타입에 `@Controller` 또는 `@RequestMapping` 애노테이션이 있어야 스프링 컨트롤러로 인식한다. 그리고 스프링 컨트롤러로 인식해야, HTTP URL이 매핑되고 동작한다. 이 애노테이션은 인터페이스에 사용해도 된다.
- `@ResponseBody` : HTTP 메시지 컨버터를 사용해서 응답한다. 이 애노테이션은 인터페이스에 사용해도 된다.
- `@RequestParam("itemId") String itemId` : 인터페이스에는 `@RequestParam("itemId")` 의 값을 생략하면 itemId 단어를 컴파일 이후 자바 버전에 따라 인식하지 못할 수 있다. 인터페이스에서는 꼭 넣어주자. 클래스에는 생략해도 대부분 잘 지원된다.
- 코드를 보면 `request()`, `noLog()` 두 가지 메서드가 있다. `request()` 는 `LogTrace` 를 적용할 대상이고, `noLog()` 는 단순히 `LogTrace` 를 적용하지 않을 대상이다.

> 주의! - 스프링 부트 3.0 이상    
> 스프링 부트 3.0 이상이라면 정상 동작하지 않는다. 꼭 예제 프로젝트 만들기 v1 마지막에 있는 스프링 부트
> 3.0 변경 사항을 확인해서 코드를 변경하자!

__OrderControllerV1Impl__    
```java
package hello.proxy.app.v1;

import lombok.extern.slf4j.Slf4j;

@Slf4j
public class OrderControllerV1Impl implements OrderControllerV1 {

    private final OrderServiceV1 orderService;

    public OrderControllerV1Impl(OrderServiceV1 orderService) {
        this.orderService = orderService;
    }

    @Override
    public String request(String itemId) {
        orderService.orderItem(itemId);
        return "ok";
    }

    @Override
    public String noLog() {
        return "ok";
    }
}
```
- 컨트롤러 구현체이다. `OrderControllerV1` 인터페이스에 스프링MVC 관련 애노테이션이 정의되어 있다.

__AppV1Config__   
이제 스프링 빈으로 수동 등록해보자.
```java
package hello.proxy.config;

import hello.proxy.app.v1.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppV1Config {

    @Bean
    public OrderControllerV1 orderControllerV1() {
        return new OrderControllerV1Impl(orderServiceV1());
    }

    @Bean
    public OrderServiceV1 orderServiceV1() {
        return new OrderServiceV1Impl(orderRepositoryV1());
    }

    @Bean
    public OrderRepositoryV1 orderRepositoryV1() {
        return new OrderRepositoryV1Impl();
    }
}
```
__ProxyApplication - 코드 추가__    
```java
package hello.proxy;

import hello.proxy.config.AppV1Config;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Import;

@Import(AppV1Config.class)
@SpringBootApplication(scanBasePackages = "hello.proxy.app") //주의
public class ProxyApplication {

	public static void main(String[] args) {
		SpringApplication.run(ProxyApplication.class, args);
	}

}
```
- `@Import(AppV1Config.class)` :  클래스를 스프링 빈으로 등록한다. 여기서는 `AppV1Config.class` 를 스프링 빈으로 등록한다. 일반적으로 `@Configuration` 같은 설정 파일을 등록할 때 사용하지만, 스프링 빈을 등록할 때도 사용할 수 있다.
- `@SpringBootApplication(scanBasePackages = "hello.proxy.app")` : `@ComponentScan` 의 기능과 같다. 컴포넌트 스캔을 시작할 위치를 지정한다. 이 값을 설정하면 해당 패키지와 그 하위 패키지를 컴포넌트 스캔한다. 이 값을 사용하지 않으면 `ProxyApplication` 이 있는 패키지와 그 하위 패키지를 스캔한다. 참고로 `v3` 에서 지금 설정한 컴포넌트 스캔 기능을 사용한다.

> 주의   
> 강의에서는 `@Configuration` 을 사용한 수동 빈 등록 설정을 `hello.proxy.config` 위치에 두고 점진적으로 변경할 예정이다. 지금은 `AppV1Config.class` 를
> `@Import` 를 사용해서 설정하지만 이후에 다른 것을 설정한다는 이야기이다.

> `@Configuration` 은 내부에 `@Component` 애노테이션을 포함하고 있어서 컴포넌트 스캔의 대상이 된다. 따라서 컴포넌트 스캔에 의해 `hello.proxy.config` 위치의 설정 파일들이 스프링 빈으로 자동 등록 되지 않도록 컴포넌스 스캔의 시작 위치를 `scanBasePackages=hello.proxy.app` 로 설정해야 한다. 

__스프링 부트 3.0 변경 사항__    
스프링 부트 3.0 이상이라면 지금까지 작성한 코드에서 2가지를 변경해야 한다.    

스프링 부트 3.0(스프링 프레임워크 6.0)부터는 클래스 레벨에 `@RequestMapping` 이 있어도 스프링 컨트롤러로 인식하지 않는다. 오직 @Controller 가 있어야 스프링 컨트롤러로 인식한다. 참고로 `@RestController` 는 해당 애노테이션 내부에 `@Controller` 를 포함하고 있으므로 인식 된다. 따라서 다음과 같이 변경해야 한다.

__스프링 부트 3.0 미만__   
```java
@RequestMapping //스프링은 @Controller 또는 @RequestMapping 이 있어야 스프링 컨트롤러로 인식
@ResponseBody
public interface OrderControllerV1 {}
```
__스프링 부트 3.0 이상__   
```java
@RestController //스프링은 @Controller, @RestController가 있어야 스프링 컨트롤러로 인식
public interface OrderControllerV1 {}
```
__ProxyApplication - 스프링 부트 3.0 미만__    
```java
@Import(AppV1Config.class)
@SpringBootApplication(scanBasePackages = "hello.proxy.app") //주의
public class ProxyApplication {}
```
__ProxyApplication - 스프링 부트 3.0 이상__    
```java
@Import(AppV1Config.class)
@SpringBootApplication(scanBasePackages = "hello.proxy.app.v3") //주의
public class ProxyApplication {}
```
`scanBasePackages` 부분에서 마지막에 `v3` 가 붙었다. `hello.proxy.app` -> `hello.proxy.app.v3` 이렇게 하는 이유는 스프링 부트 3.0부터는 `@Controller` , `@RestController` 를 사용했는데, 이렇게 하면 내부에 `@Component` 를 가지고 있어서 컴포넌트 스캔의 대상이 된다. 지금 처럼 컴포넌트 스캔도 되고, 빈도 수동으로 직접 등록하게 되면 스프링 컨테이너에 등록시 충돌 오류가 발생한다. 이후에 학습할 `hello.proxy.app.v3` 는 빈을 직접 등록하지 않고 컴포넌트 스캔을 사용하기 때문에 괜찮다.

### 예제 프로젝트 만들기 v2
