---
title: DispatcherServlet Debugging하기 - 1부(Request 영역)
author: mingo
date: 2023-09-17 00:30:00 +0900
categories: [spring, MVC]
tags: [servlet, mvc, dispatcherservlet]
---

----

이번 포스팅에서는 Spring MVC의 핵심이 되는 **DispatcherServlet**를 Debugging해보면서 천천히 흐름을 따라가볼 생각이다.
우선 Debugging을 시작하기 전에 친숙한 **DispatcherServlet**의 큰 그림를 보면서 Request영역과 Response영역을 나눠서 생각해보자.


## | DispatcherServlet 구조

![Desktop View](/assets/img/post/20230915/21.png){: width="800" height="680" }
_DispatcherServlet 구조_

영역별로 커다란 기능을 뽑아보자면,

**Request 영역**
1. 핸들러 매핑
2. 핸들러 어댑터 매핑
3. 핸들러 호출(Contoller)

**Response 영역**
1. 반환 값에 따라 ModelAndView 생성
2. 뷰리졸버 매핑
3. 뷰 렌더링

위와 같은 기능을 생각하면서 소스 어디에 숨겨져 있는지 천천히 Debugging을 하며 따라가보자.

## | DispatcherServlet 관계도

![Desktop View](/assets/img/post/20230915/3.png){: width="800" height="680" }
_DispatchServlet 관계도_
DispatcherServlet은 HttpServlet을 구현하고 있는 servlet이다.

DispatchServlet 관계도를 살펴보면 `HttpServlet` < `FrameworkServlet` < `HttpServletBean` < `DispatcherServlet` 순으로 클래스를 상속받고 있다.
소스가 매우 복잡해서 상속구조를 알고 미리 알고보면 편하다.  

각 클래스에 대해서 가볍게 설명하자면 추가예정



## | 테스트 설정

우선 HTTP 요청을 debugging하기 위해 Contoller를 구현했다.
```java
@Slf4j
@RestController
public class TestController {

    @GetMapping("/profile")
    public User test(@RequestParam String name) {
        log.info("name is {}", name);
        return User.builder()
                .name(name)
                .addr("seoul")
                .job("delvoper")
                .build();
    }

    @Getter
    static class User {

        private String name;
        private String addr;
        private String job;

        @Builder
        public User(String name, String addr, String job) {
            this.name = name;
            this.addr = addr;
            this.job = job;
        }
    }

}
```
`/profile?name=xxx` 쿼리 파라미터로 user명을 받고 임의의 데이터를 설정한 뒤 User 객체를 반환하는 간단한 비즈니스 로직을 만들어 디버깅에 이용했다.

## | Request 영역

### 👉 1. 핸들러(Handler) 매핑
debugging의 breaking point로는 서블릿의 구현체인 HttpServlet의 service() 메서드로 시작했다. 클라이언트의 HTTP 요청으로 서블릿이 호출되면 service()가 호출되기 때문이다.

![Desktop View](/assets/img/post/20230915/4.png){: width="800" height="680" }
_HttpServlet protected service() 메서드_

다음으로 브라우저를 통해 `http://localhost:8080/profile?name=mingo` HTTP GET요청을 쿼리 파라미터와 함께 호출해보았다.

`HttpServlet`의 service() 메서드는 `FrameworkServlet` 에서 @Override 되었다. `FrameworkServlet`의 service() 메서드는 간단하게 HTTP request method를 검증하는 로직이 있고 다시 superclass(HttpServlet)로 역할을 위임한다.
> FrameworkServlet의 service() 메서드에서 허용하는 method는 "DELETE", "HEAD", "GET", "OPTIONS", "POST", "PUT", "TRACE" 이다.

![Desktop View](/assets/img/post/20230915/6.png){: width="800" height="680" }
_HttpServlet_
`HttpServlet`의 service() 메서드가 다시 시작되고 테스트 요청(GET : "/profile") 시, GET method를 사용했기에 doGet() 메서드가 호출된다.
doGet()은 다시 `FrameworkServlet`에서 @Override 하였고 재정의된 doGet()은 내부에서 processRequest()를 호출한다.

![Desktop View](/assets/img/post/20230915/7.png){: width="800" height="680" }
_processRequest() 메서드 내부_

processRequest() 메서드 내부에서 locale(-> LocaleContextHolder)과 request(-> RequestContextHolder) 정보를 설정하는 코드를 찾을수 있다.
RequestContextHolder 객체는 spring2.x 부터 등장하였는데, servlet, filter, interceptor, AOP, controller뿐만 아니라, 전 구간에서 요청 데이터를 담고있는 `HttpServletRequest` 객체에 접근할 수 있는 static method를 제공해준다. (**SCOPE : Request Thread**)

이어서 doService() 메서드가 호출되는데, 이를 DispatcherServlet이 @Override하여 재정의한다. doService() 메서드는 doDispatch() 메서드로 요청을 위임한다. 이제 `DispatcherServlet`이 등장했다.

![Desktop View](/assets/img/post/20230915/8.png){: width="800" height="680" }
_DispatcherServlet의 doDispatch()_

주석을 읽어보면 모든 HTTP method가 핸들링되는 메서드이며, `HandlerMapping`과 `HandlerAdapter`을 통해서 적절한 핸들러(Contoller)를 찾는다고 한다.

![Desktop View](/assets/img/post/20230915/9.png){: width="800" height="680" }
_DispatcherServlet의 doDispatch()_
코드의 흐름에 따라 내려오면 현재 요청에 대한 핸들러를 결정한다는 주석과 getHandler() 메서드를 발견할 수 있다.

![Desktop View](/assets/img/post/20230915/10.png){: width="800" height="680" }
_getHandler 내부_

getHandler() 메서드 내부 코드를 살펴보면, handlerMappings 리스트를 순회하면서 mapping.getHandler() 메서드를 호출하는 코드가 있다. 
mapping.getHandler()는 간단히 요약하면, 테스트 요청(GET : "/profile")에 맞는 핸들러가 있다면 HandlerExecutionChain객체가 반환하고 아니면 null이 반환되는 메서드이다. 이제 
**핸들러 매핑**이 이루어졌다!

그럼 handlerMappings 리스트는 언제, 어떻게 세팅되었을까?

![Desktop View](/assets/img/post/20230915/11.png){: width="800" height="680" }
_handlerMappings 목록_

우선 전체 HandlerMapping를 구현한 객체는 총 14개가 있었지만, Debugging에서 확인되는 list안의 객체는 6개가 있었다. 
여기서 알 수 있는것은 모든 HandlerMapping 객체를 리스트에 담아두는 것은 아니라는 점이다. 

참고로 handlerMappings 리스트가 담겨지는 시점은 `DispatcherServlet`이 생성되는 시점이다.

![Desktop View](/assets/img/post/20230915/12.png){: width="800" height="680" }
_DispatcherServlet의 initStrategies()_

앞으로 자주 언급될 HandlerMapping, HandlerAdapter, ViewResolver 모두 `DispatcherServlet`이 생성될 때 초기화되고 각 메서드를 살펴보면 상세한 코드가 나온다. (더 자세한 코드는 보지 않았으나, 시간이 날때 어떤 방식과 순서로 list를 구성하는지 그리고 HandlerMapping 구현체들의 내부로직을 확인해보면 좋을 것 같다.)

또한 list에 HandlerMapping의 구현체들이 담기는 순서는 각 구현체의 내부 우선순위(order field)를 통해 정해진다. 그러므로 handlerMappings 리스트를 순회하면서 핸들러를 찾을 때, 우선순위가 높은 핸들러부터 매핑시킨다는 의미가 된다.

테스트 요청(GET : "/profile")에 알맞는 HandlerMapping의 구현체로는 `RequestMappingHandlerMapping`인데 
이것은 테스트 시, `@GetMapping("/profile")` == `@RequestMapping` 을 사용하여 Controller의 경로를 설정하였기 때문이다.

여기까지 url을 기반으로 핸들러(Controller)가 매핑되었다. 이어서 핸들러 어댑터 매핑을 알아보자. 

### 👉 2. 핸들러 어댑터(HandlerAdapter) 매핑

![Desktop View](/assets/img/post/20230915/13.png){: width="800" height="680" }

이어서 핸들러 매핑을 통해 찾은 핸들러(Controller)를 실행할 수 있는 어댑터를 찾기 위해 getHandlerAdapter() 메서드가 바로 나온다. (심지어 주석도 너무 친절하다. 요청에 대한 핸들러 어댑터를 결정하는 메서드 ㅎㅎ 디버깅하면서 느끼는건데 Spring 소스는 복잡한 만큼 주석을 잘 남겨주었다는 생각이 든다.) 

![Desktop View](/assets/img/post/20230915/14.png){: width="800" height="680" }

getHandlerAdapter() 메서드는 getHandlerMapping() 메서드와 구조적으로 거의 동일하다. handlerAdapters 리스트를 순회하면서 adapter.supports() 메소드를 통해 매핑된 핸들러를 지원할 수 있는 어댑터를 찾는다.
테스트 요청(GET : "/profile")에 알맞는 어댑터는 핸들러 매핑과 마찬가지로 @RequestMapping를 통한 매핑이기 때문에 구현체로 `RequestMappingHandlerAdapter`가 선택된다. (어댑터 리스트도 우선순위가 존재하여 우선순서대로 정렬되어있다.) 

이어서 선택된 어댑터의 handle() 메서드를 통해 HTTP 요청을 Controller에게 넘기는 handler가 시작된다.

### 👉 3. 핸들러 호출

![Desktop View](/assets/img/post/20230915/15.png){: width="800" height="680" }
_RequestMappingHandlerAdapter의 관계도_

어댑터의 handle() 메서드는 superclass인 `HttpRequestHandlerAdapter`를 거쳐 다시 돌아와 handleInternal() 메서드로 위임된다. 

![Desktop View](/assets/img/post/20230915/16.png){: width="800" height="680" }

여기서 리턴 값이 `ModelAndView` 인 것을 알 수 있는데 간단하게 class를 살펴보자면, 

> ![Desktop View](/assets/img/post/20230915/17.png){: width="800" height="680" }
  _ModelAndView 객체_
  class의 주석을 읽어보면, Spring Web MVC에서 Model과 View를 Contoller에서 하나의 Object로 처리할 수 있도록 설계되어 있다는 것을 알 수 있다. 
  `view` field는 view의 논리명을 받거나 또는 객체의 형태로 받을 수 있고 `model` field의 경우, LinkedHashMap를 상속받은 ModelMap으로 구성되어 뷰 단으로 전달할 데이터를 key-value 형태로 저장해두는 공간이 된다. 


이어서 handleInternal() 메서드를 보면, 세션 존재 여부 / 동기화 체크를 구분하지만, 결과적으로 흐름만 보자면 invokeHandlerMethod() 메서드를 호출하고 있다.

```java
protected ModelAndView invokeHandlerMethod(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

  ...

  ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
  if (this.argumentResolvers != null) {
    // ArgumentResolver 목록 세팅
    invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
  }
  if (this.returnValueHandlers != null) {
    // ReturnValueHandler 목록 세팅
    invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
  }
  invocableMethod.setDataBinderFactory(binderFactory);
  invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

  ...

  // 핸들러 호출
  invocableMethod.invokeAndHandle(webRequest, mavContainer);
  
  ...
        
  return getModelAndView(mavContainer, modelFactory, webRequest);
}
```
invokeHandlerMethod 메서드 내부를 보면 여기서 바로 `ArgumentResolver`, `ReturnValueHandler` 가 등장한다.

![Desktop View](/assets/img/post/20230915/18.png){: width="800" height="680" }

위의 동작방식 도식화된 그림을 보고 다시 테스트 요청(GET : "/profile")을 받는 Controller의 메서드를 생각해보자.
```java
public User test(@RequestParam String name) {
    ...
}
```
`/profile?name=mingo` 경로를 받을수 있는 Contoller를 만들었을 때, 별다른 생각없이 평소 하던 대로 `@RequestParam`를 통해 쿼리 파라미터를 받았다.
Controller의 arguments는 정의된 범위 내에서 개발자가 자유롭게 arguments를 넣어줄 수 있었고 이것을 가능하게 해주는 것이 `ArgumentResolver`다.

`invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);` 로 인하여 세팅된 20 여개의 
`ArgumentResolver`는 supportsParameter()를 호출해서 해당 파라미터를 지원하는지 체크하는 로직을 통해 request 요청에 맞는 핸들러(Contoller)의 arguments를 세팅한다.

이처럼 유연한 파라미터 처리를 위한 로직을 컨닝하기 위해 좀 더 들어가 보자.

![Desktop View](/assets/img/post/20230915/19.png){: width="800" height="680" }
_InvocableHandlerMethod의 getMethodArgumentValues()_

쿼리 파라미터를 값을 배열로 반환하는 메서드를 발견했다.
테스트 요청(GET : "/profile")은 하나의 쿼리 파라미터로 진행해서 배열 args[] 안에  `mingo` 값만 존재하였지만, 다양한 @RequestBody, @HttpSession, @ModelAttribute 등 다양한 arguments를
사용했다면 좀 더 다양한 `ArgumentResolver`가 동작하고 다양한 값이 배열 args[]에 담겼을 것이다.

- Controller에서 사용 가능한 arguments 목록
  : <https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods/arguments.html>

![Desktop View](/assets/img/post/20230915/20.png){: width="800" height="680" }
_InvocableHandlerMethod의 doInvoke()_

이제 Request 영역의 마지막으로 `ArgumentResolver`로 생성한 인자들을 파라미터로 넣고 doInvoke() 메서드를 호출하면 개발자가 작업한 Controller에 도착하게 된다.
```text
i.m.experiment.servlet.TestController    : name is mingo
```
Controller의 log 확인까지 완료되었다.

Response 영역은 다음 포스팅에서 이어서 작성! (DispatcherServlet Debugging하기 - 2부(Response 영역)) 
