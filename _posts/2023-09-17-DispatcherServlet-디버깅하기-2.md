---
title: DispatcherServlet Debugging하기 - 2부(Response 영역)
author: mingo
date: 2024-09-17 00:30:00 +0900
categories: [spring, MVC]
tags: [servlet, mvc, dispatcherservlet]
---

----



저번 [**DispatcherServlet Debugging하기 - 1부(Request 영역)**](https://mingovvv.github.io/posts/DispatcherServlet-%EB%94%94%EB%B2%84%EA%B9%85%ED%95%98%EA%B8%B0/)에 이어서
DispatcherServlet의 **Response 영역**을 Debugging해보도록 하자.

우선 복습할 겸, DispatcherServlet 구조를 한번 더 보고..

## | DispatcherServlet 구조

![Desktop View](/assets/img/post/20230915/21.png){: width="800" height="680" }
_DispatcherServlet 구조_

**Request 영역**
1. 핸들러 매핑
2. 핸들러 어댑터
3. 핸들러 호출(Contoller)

**Response 영역**
1. ModelAndView 생성
2. 뷰리졸버
3. 뷰 렌더링

Request 영역의 **3. 핸들러 호출**까지 확인을 했고, 직접 만든 Test Controller로 클라이언트의 HTTP 요청이 매핑되어 인입되었다.
> `http://localhost:8080/profile?name=mingo`

응답값은 직접 만든 객체이며, 임의의 값을 세팅하고 반환하였다.

```java
// Test Controller
@GetMapping("/profile")
public User test(@RequestParam String name) {
    log.info("name is {}", name); // breaking point!!
    return User.builder()
            .name(name)
            .addr("seoul")
            .job("delvoper")
            .build();
}
```

## | Response 영역

Response 영역을 Debugging하기 위한 breaking point는 Test Controller의 log를 남기는 부분으로 설정하였다.

### 👉 1. ModelAndView 생성

![Desktop View](/assets/img/post/20230918/1.png){: width="800" height="680" }
_InvocableHandlerMethod의 doInvoke()_

Contoller에서 `User` 객체를 반환하니, 이전 포스팅의 Request 영역의 마지막 InvocableHandlerMethod 클래스가 나왔다. 
다양한 반환 타입이 있을 수 있으므로 doInvoke() 메서드에서도 Object로 반환하고 있다. (테스트에서는 "User" 객체를 반환하였다.)

흐름을 따라가면 setResponseStatus() 메서드가 있으나 @ResponseStatus 애노테이션을 사용하여 HTTP 응답코드를 설정하지 않았으므로 정상(200) 응답이 반환 될 것이다. 

```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {

    // returnValue에 "User" 객체가 담겨 있음
		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
        
    // 응답코드 설정
		setResponseStatus(webRequest);
    
    ...
  
		try {
      // 응답값 핸들링!!!
			this.returnValueHandlers.handleReturnValue(returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
		}
		catch (Exception ex) {
			if (logger.isTraceEnabled()) {
				logger.trace(formatErrorForReturnValue(returnValue), ex);
			}
			throw ex;
		}
	}
```

ServletInvocableHandlerMethod의 invokeAndHandle() 메서드안에서 handleReturnValue() 메서드를 통해 응답값을 핸들링하는 부분을 찾았다.

메서드를 따라 들어가면,

![Desktop View](/assets/img/post/20230918/2.png){: width="800" height="680" }

주석을 보면 "등록된 `HandlerMethodReturnValueHandler`를 반복하고 이를 지원하는 Handler를 호출합니다." 라고 한다.
`ReturnValueHandler` 또한 애플리케이션 구동 시에 `ArgumentResolver`와 함께 여러 타입을 지원하기 위해 등록되었다. 

![Desktop View](/assets/img/post/20230918/4.png){: width="400" height="340" }
_returnValueHandlers_
`ReturnValueHandler` 객체들을 살펴보면 이름을 통해 어떤 객체가 핸들러로 사용될 지 유추할 수 있다. 
이번 테스트의 경우, @RestController(@ResponseBody)를 사용하여 값을 리턴하였으니 `RequestResponseBodyMethodProcessor`를 `ReturnValueHandler`로 사용한다.
> Controller에서 return값을 view의 논리명으로 하였을 경우, `ReturnValueHandler`는 `ViewNameMethodReturnValueHandler`로 선택된다.

`ReturnValueHandler`를 찾았으니 이어서 `RequestResponseBodyMethodProcessor`가 리턴값(User객체)의 데이터를 어떻게 핸들링하는 과정을 따라가보자.

handler.handleReturnValue() 메서드는 

