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
2. 핸들러 어댑터
3. 핸들러 호출(Contoller)

**Response 영역**
1. ModelAndView 생성
2. 뷰리졸버
3. 뷰 렌더링

위와 같은 기능을 생각하면서 소스 어디에 숨겨져 있는지 천천히 Debugging을 하며 따라가보자.

## | DispatcherServlet 관계도

![Desktop View](/assets/img/post/20230915/3.png){: width="800" height="680" }
_DispatchServlet 관계도_
DispatcherServlet은 HttpServlet을 구현하고 있는 servlet이다.

DispatchServlet 관계도를 살펴보면 `HttpServlet` < `FrameworkServlet` < `HttpServletBean` < `DispatcherServlet` 순으로 클래스를 상속받고 있다.
소스가 매우 복잡해서 상속구조를 알고 미리 알고보면 편하다.  

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
debugging의 breaking point로는 서블릿의 구현체인 HttpServlet의 service() 메서드로 시작했다. 
클라이언트의 HTTP 요청으로 서블릿이 호출되면 service()가 호출되기 때문이다.

![Desktop View](/assets/img/post/20230915/4.png){: width="800" height="680" }
_HttpServlet protected service() 메서드_

다음으로 브라우저를 통해 `http://localhost:8080/profile?name=mingo` HTTP GET요청을 쿼리 파라미터와 함께 호출해보았다.

`HttpServlet`의 service() 메서드는 `FrameworkServlet` 에서 @Override 되었다. `FrameworkServlet`의 service() 메서드는 간단하게 HTTP request method를 검증하는 로직이 있고 다시 superclass(HttpServlet)로 역할을 위임한다.

![Desktop View](/assets/img/post/20230915/6.png){: width="800" height="680" }
_HttpServlet_

`HttpServlet`의 service() 메서드가 다시 시작되고 테스트 요청(GET : "/profile") 시, GET method를 사용했기에 doGet() 메서드가 호출된다.
doGet()은 다시 `FrameworkServlet`에서 @Override 하였고 재정의된 doGet()은 내부에서 processRequest()를 호출한다.

![Desktop View](/assets/img/post/20230915/7.png){: width="800" height="680" }
_processRequest() 메서드 내부_

processRequest() 메서드 내부에서 locale(-> LocaleContextHolder)과 request(-> RequestContextHolder) 정보를 설정하는 코드를 찾을수 있다.
RequestContextHolder 객체는 spring2.x 부터 등장하였는데, servlet, filter, interceptor, AOP, controller뿐만 아니라, 전 구간에서 요청 데이터를 담고있는 `HttpServletRequest` 객체에 접근할 수 있는 static method를 제공해준다. (**SCOPE : Request Thread**)

이어서 doService() 메서드가 호출되는데, 이를 DispatcherServlet이 @Override하여 재정의한다. doService() 메서드는 doDispatch() 메서드로 요청을 위임한다. 이제 `DispatcherServlet`이 등장한다.

![Desktop View](/assets/img/post/20230915/8.png){: width="800" height="680" }
_DispatcherServlet의 doDispatch()_

주석을 읽어보면,
> 핸들러는 `HandlerMappings`를 *순서대로* 적용하여 얻을 수 있습니다. 핸들러-어댑터는 `HandlerAdapters`를 쿼리하여, 핸들러 클래스를 지원하는 첫 번째 핸들러-어댑터를 찾음으로써 얻을 수 있습니다. 
  모든 HTTP 메소드는 이 메소드에 의해 처리됩니다. 어떤 메소드가 허용되는지는 핸들러 어댑터 또는 핸들러 자체에 달려 있습니다.

해석만 보면, 순서대로? 쿼리하여? 약간 모호할 수 있지만, 아래 흐름을 따라가다보면 이해할 수 있을 것이다.
핵심은 `handlerMappings`, `handlerAdapters`를 통해서 핸들러와 핸들러-어댑터를 얻을 수 있다는 내용이다.

![Desktop View](/assets/img/post/20230915/9.png){: width="800" height="680" }
_DispatcherServlet의 doDispatch()_

코드의 흐름에 따라 내려오면, *"현재 요청에 대한 핸들러를 결정한다"*는 주석이 달려있는 getHandler() 메서드를 발견할 수 있다.

![Desktop View](/assets/img/post/20230915/10.png){: width="800" height="680" }
_getHandler 내부_

getHandler() 메서드 내부 코드를 살펴보면, **`handlerMappings` 리스트를 순서대로 순회**하면서 mapping.getHandler() 메서드를 호출하는 코드가 있다.

`handlerMappings` 리스트 부터 알아보자.

해당 리스트는 애플리케이션이 실행될 때, 미리 세팅되는 리스트이며 디버깅은 아래와 같다.
![Desktop View](/assets/img/post/20230915/24.png){: width="400" height="340" }
_handlerMappings 목록_

RequestMappingHandlerMapping, BeanNameUrlHandlerMapping 등 여러 개의 핸들러-매핑 객체가 리스트에 담겨있는데 모두 `HandlerMapping` 인터페이스의 구현체이다.

![Desktop View](/assets/img/post/20230915/12.png){: width="800" height="680" }
_DispatcherServlet의 initStrategies()_

프로그램 구동 시점에 initStrategies() 메서드에서 초기화되고 핸들러-매핑을 위한 여러 종류의 핸들러-매핑 객체가 담겨있다. (RequestMapping을 사용하는 객체, bean이름을 사용하는 객체 등등...)
(**추가적으로 여러 종류의 핸들러-어댑터 객체와 뷰 리졸버가 이 시점에 등록된다.**)

> 각 핸들러-매핑 객체는 내부적으로 우선순위(order)가 정해져서 `handlerMappings` 리스트에 초기화 될 때 해당 값의 순서대로 정렬된다. (RequestMappingHandlerMapping 객체가 리스트 처음에 위치한 것은 해당 객체가 0순위라는 의미이다.)

테스트 요청(GET : "/profile")을 처리하는 Controller를 구현할 때, @RequestMapping을 사용하였고 바로 `RequestMappingHandlerMapping` 핸들러-매핑 객체가 애노테이션 기반의 핸들러를 찾는 핸들러-매핑 객체이기 때문에 테스트 요청(GET : "/profile")은 `RequestMappingHandlerMapping` 핸들러-매핑 객체를 사용하게 된다.  

이어서 핸들러-매핑 객체의 getHandler() 메서드의 내부코드를 간단히 요약하면, **클라이언트의 HTTP 요청을 처리할 수 있는 핸들러(Controller의 method)를 uri 기반으로 찾아 반환하는 메서드이다.**

> **핸들러를 찾는과정**
  1. 애플리케이션이 실행될 때, 개발자가 구현한 모든 @Controller(@RestController) + @RequestMapping(@GetMapping...etc) 조합의 메서드를 찾아서 파라미터 타입, 반환 타입, 메소드명 등등 모든 정보를 AbstractHandlerMethodMapping 클래스의 innerclass인 MappingRegistry 클래스에 저장한다.
  2. MappingRegistry 클래스는 내부적으로 HashMap을 구현한 필드를 만들어두고 접근할 수 있도록 한다. (key : `"HTTP method [/uri]"` / value : MappingRegistration 객체)
     ![Desktop View](/assets/img/post/20230915/23.png){: width="400" height="340" }
     _MappingRegistry클래스의 registry 필드_ 
  3. 애플리케이션이 로딩이 되었다면, 개발자가 구현한 모든 핸들러가 1, 2 프로세스를 통해 초기화가 된 상태이며, 이제부터 클라이언트의 HTTP 요청을 처리할 수 있다.
  4. 클라이언트 HTTP 요청의 uri를 가공한 정보(`"HTTP method [/uri]"`)로 MappingRegistration 객체 찾아내어, 일련의 과정을 거쳐 HandlerExecutionChain 객체로 매핑된 핸들러를 꺼내온다.

위의 과정을 `handlerMappings` 리스트를 순회하면서 매핑되는 핸들러(Controller의 method)를 찾아오는 방식이다.

여기까지 HTTP 요청을 기반으로 핸들러(Controller의 method)가 매핑되었다. 이어서 핸들러 어댑터를 알아보자. 

### 👉 2. 핸들러 어댑터(HandlerAdapter)

![Desktop View](/assets/img/post/20230915/13.png){: width="800" height="680" }
_DispatcherServlet의 doDispatcher() 메서드 내부_

이어서 핸들러-매핑을 통해 찾은 핸들러(Controller method)를 실행할 수 있는 어댑터를 찾기 위해 getHandlerAdapter() 메서드가 바로 이어서 나온다. (주석도 너무 친절하다. 요청에 대한 핸들러-어댑터를 결정하는 메서드 ㅎㅎ 디버깅하면서 느끼는건데 Spring 소스는 복잡한 만큼 주석을 잘 남겨주었다.)

![Desktop View](/assets/img/post/20230915/14.png){: width="800" height="680" }
_handlerAdapters_

getHandlerAdapter() 메서드는 getHandlerMapping() 메서드와 구조적으로 거의 동일하다. `handlerAdapters` 리스트를 순회하면서 adapter.supports() 메소드를 통해 핸들러를 지원할 수 있는 어댑터를 찾는다.

핸들러-어댑터 객체는 `HandlerAdapter` 인터페이스의 구현체이며 `handlerAdapters` 리스트는 애플리케이션 구동 시, initStrategies() 메서드를 통해 초기화된다는 것을 아까 전에 확인했었다.
또한, 핸들러-어댑터 객체도 역시 내부적으로 우선순위(order)를 가지고 있어 `handlerAdapters` 리스트에서 정렬된다.  

테스트 요청(GET : "/profile")에 알맞는 핸들러-어댑터 객체 또한 @RequestMapping를 통한 매핑이기 때문에 `RequestMappingHandlerAdapter`가 supports() 시, true로 반환되어 선택된다. 

그럼 이제 핸들러-어댑터까지 찾아냈다. 어탭터를 이용하여 HTTP 요청을 핸들러(Controller method)로 위임하는 것을 따라가보자.

### 👉 3. 핸들러 호출

![Desktop View](/assets/img/post/20230915/26.png){: width="800" height="680" }
_DispatcherServlet의 doDispatcher() 메서드 내부_

지원하는 핸들러-어댑터 객체는 `ha` 필드에 담겨 있고 해당 객체의 handle() 메서드를 시작으로 HTTP 요청을 핸들러(Controller method)로 전달하는 과정이 진행된다.

![Desktop View](/assets/img/post/20230915/15.png){: width="800" height="680" }
_RequestMappingHandlerAdapter의 관계도_

핸들러-어댑터 객체의 handle() 메서드는 superclass인 `HttpRequestHandlerAdapter`를 거쳐 다시 돌아와 handleInternal() 메서드로 위임된다. 

![Desktop View](/assets/img/post/20230915/16.png){: width="800" height="680" }

여기서 리턴 값이 `ModelAndView` 인 것을 알 수 있는데 간단하게 살펴보자면, 

> ![Desktop View](/assets/img/post/20230915/17.png){: width="800" height="680" }
  _ModelAndView 객체_
  class의 주석을 읽어보면, Spring Web MVC에서 Model과 View를 하나의 Object로 처리할 수 있도록 설계되어 있다는 것을 알 수 있다. 
  `view` field는 view의 논리명을 받거나 또는 객체의 형태로 받을 수 있고 `model` field의 경우, LinkedHashMap를 상속받은 ModelMap객체 형태로 뷰에게 전달할 데이터를 key-value 형태로 저장해두는 공간이 된다. 

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
invokeHandlerMethod 메서드 내부를 보면 여기서 바로 `ArgumentResolver`, `ReturnValueHandler` 가 등장한다. (`ReturnValueHandler` Response 영역에서 알아보자.)
이 것은 역할은 무엇일까?

```java
@GetMapping("/profile")
public User test(@RequestParam String name) { // ArgumentResolver가 동작해서 파라미터를 넣어줌
  ...   
}
```
다시 직접 만든 컨트롤러를 확인해보자.

`/profile?name=mingo` 경로를 받을수 있는 Contoller를 만들었을 때, 별다른 생각없이 `@RequestParam`를 통해 쿼리 파라미터의 값을 통해 받고자 하였다. 이 파라미터는 어디서 세팅해주는걸까?
이것은 사실 내부적으로 `ArgumentResolver`가 동작하여, @RequestParam를 분석하고 String타입의 `name` 필드에 클라이언트의 HTTP 요청에서 받은 name 쿼리 파라미터의 값을 전달해 주었던 것이다. 

![Desktop View](/assets/img/post/20230915/18.png){: width="800" height="680" }
_ArgumentResolver 동작 도식화_

`invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);` 에서 `ArgumentResolver`가 주입된다.
세팅된 20 여개의 `ArgumentResolver` 객체들을 사용해서 호출하려는 핸들러(Controller method)에서 구현된 arguments의 값을 만들어낸다.  

이처럼 유연한 파라미터 처리를 가능하게 해주는 로직을 컨닝하기 위해 좀 더 들어가 보자.

![Desktop View](/assets/img/post/20230915/19.png){: width="800" height="680" }
_InvocableHandlerMethod의 getMethodArgumentValues()_

쿼리 파라미터를 값을 배열[]로 반환하는 메서드를 발견했다.
테스트 요청(GET : "/profile")은 하나의 쿼리 파라미터를 사용했기 때문에 배열 args[] 안에  `mingo` 값만 존재하였지만, 다양한 @RequestBody, @HttpSession, @ModelAttribute 등 다양한 arguments가 존재하는 Contollrer를 만들고
사용했다면 좀 더 다양한 `ArgumentResolver`가 동작하고 다양한 값이 배열 args[]에 담겼을 것이다.

- Controller에서 사용 가능한 arguments 목록
  : <https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods/arguments.html>

![Desktop View](/assets/img/post/20230915/20.png){: width="800" height="680" }
_InvocableHandlerMethod의 doInvoke()_

이제 Request 영역의 마지막으로 `ArgumentResolver`로 생성한 값을 파라미터로 넣고 doInvoke() 메서드를 호출하면 개발자가 작업한 Controller에 도착하게 된다.
```text
i.m.experiment.servlet.TestController    : name is mingo
```
Controller의 log 확인까지 완료되었다.

Response 영역은 다음 포스팅에서 이어서 작성! (DispatcherServlet Debugging하기 - 2부(Response 영역))
