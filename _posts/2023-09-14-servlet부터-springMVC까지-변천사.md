---
title: Servlet, JSP, Spring MVC - code로 접근하기 
author: mingo
date: 2023-09-15 04:14:00 +0900
categories: [spring, MVC]
tags: [servlet, jsp, mvc]
---

----

오늘날 java 개발자가 웹 개발을 하면 **spring MVC**를 기반으로 개발을 시작하는 게 대부분이다.
java를 사용하는 동적 웹 프로그래밍이 발전해 온 과정을 간단한 코드를 통해 알아보자.

## 1. only servlet 동적 프로그래밍
servlet은 JSP 표준이 나오기 전에 만들어진 표준으로 java를 통해 웹 프로그래밍을 하기 위하여 만들어졌다.
```java
@WebServlet(name = "userListServlet", urlPatterns = "/users")
public class UserListServlet extends HttpServlet {
    
    // DB 역할
    private UserRepository userRepository = UserRepository.getInstance();

    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        
        response.setContentType("text/html");
        response.setCharacterEncoding("utf-8");
        
        List<User> users = userRepository.findAll();
        PrintWriter w = response.getWriter();

        w.write("<html>");
        w.write("<head>");
        w.write(" <meta charset=\"UTF-8\">");
        w.write(" <title>Title</title>");
        w.write("</head>");
        w.write("<body>");
        w.write("<a href=\"/index.html\">메인</a>");
        w.write("<table>");
        w.write(" <thead>");
        w.write(" <th>name</th>");
        w.write(" <th>job</th>");
        w.write(" </thead>");
        w.write(" <tbody>");
        // 자바 문법을 통한 동적 데이터 처리
        for (User user : users) {
            w.write(" <tr>");
            w.write(" <td>" + user.getName() + "</td>");
            w.write(" <td>" + user.getJob() + "</td>");
            w.write(" </tr>");
        }
        w.write(" </tbody>");
        w.write("</table>");
        w.write("</body>");
        w.write("</html>");
    }
    
}
```
클라이언트로부터 `/users` 경로의 HTTP 요청을 전달받아 처리한다. 요청을 전달받으려면 객체가 servlet으로 등록되어 servlet container의 관리 대상이 되어야 한다.
servlet으로 등록되기 위해서는 `HttpServlet`를 상속받아야 한다.

java 코드를 통해 동적으로 HTML 코드를 생성한다는 점이 특징이며, `HttpServletResponse` 객체를 통해 클라이언트에게 응답하는 방식이다.

HTML을 java 단에서 생성하니 가동성이 매우 떨어지고 비효율적으로 코드가 반복되고 있는데, 이것을 개선하기 위해 JSP, Thymeleaf, Freemarker와 같은 템플릿 엔진이 등장한다.

> servlet 관련 참조 : [**✍ 서블릿(Servlet) 이해하기**](https://mingovvv.github.io/posts/%EC%84%9C%EB%B8%94%EB%A6%BF-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0/)

## 2. only jsp 동적 프로그래밍
```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ page import="java.util.List" %>
<%@ page import="mingovvv.servlet.domain.UserRepository" %>
<%@ page import="mingovvv.servlet.domain.User" %>
<%
    UserRepository userRepository = UserRepository.getInstance();
    List<User> users = userRepository.findAll();
%>
<html>
<head>
  <meta charset="UTF-8">
  <title>Title</title>
</head>
<body>
  <a href="/index.html">메인</a>
  <table>
    <thead>
      <th>name</th>
      <th>job</th>
    </thead>
    <tbody>
    <%
      for (User user : users) {
        out.write(" <tr>");
        out.write(" <td>" + user.getName() + "</td>");
        out.write(" <td>" + user.getJob() + "</td>");
        out.write(" </tr>");
      }
    %>
    </tbody>
  </table>
</body>
</html>
```
JSP만으로 요청을 처리하는 샘플이다. jsp파일은 src/main/webapp/jsp/users.jsp 경로에 생성되어 있으며 이 경로가 곧 클라이언트의 요청 경로가 된다.

HTML 파일 내에서 `<%@ ... %>` 태그는 Java import문을, `<% ... %>` 태그는 Java 로직을 처리할 수 있게 해준다.  

only jsp로 웹 동적 프로그래밍을 만들어 보니 only servlet으로 처리하던 방법보다는 가독성이 좋아졌지만 하나의 파일에 너무 많은 책임이 몰려있다는 점은 아쉽다.
jsp파일 상단에서 DB에서 user 목록을 가져오는 코드가 있는데, 이것은 실제 비즈니스 로직 영역을 담당하는 부분이다.
비즈니스 로직을 수행하는 영역과 뷰 영역을 담당하는 HTML과 함께 하나의 jsp 파일에 노출되어 있다는 것은 유지보수 측면에서 개발자를 힘들게 할 것이다.
비즈니스 로직 영역인 DB 메서드 명을 바꾸더라도, 뷰 영역인 table 태그의 색상을 넣어도 jsp파일을 수정 해야하기 때문이다.

그럼 특성이 맞게 영역을 분리해보는게 어떨까?

## 3. servlet + jsp
```java
@WebServlet(name = "userListServlet", urlPatterns = "/users")
public class UserListServlet extends HttpServlet {

    // DB 역할
    private UserRepository userRepository = UserRepository.getInstance();

    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {

        List<User> users = userRepository.findAll();
        request.setAttribute("users", users);

        String viewPath = "/WEB-INF/views/users.jsp";
        RequestDispatcher dispatcher = request.getRequestDispatcher(viewPath);
        dispatcher.forward(request, response);

    }
    
}
```

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"%>

<html>
  <head>
    <meta charset="UTF-8">
    <title>Title</title>
  </head>
  <body>
    <a href="/index.html">메인</a>
    <table>
        <thead>
        <th>name</th>
        <th>job</th>
        </thead>
        <tbody>
        <c:forEach var="user" items="${users}">
            <tr>
                <td>${user.name}</td>
                <td>${user.job}</td>
            </tr>
        </c:forEach>
        </tbody>
    </table>
  </body>
</html>
```
only servlet 또는 only jsp만으로는 비즈니스 로직 영역과 뷰 영역을 분리시키지 못하므로 이를 해결하고자 등장한 것이 바로 MVC 패턴이다. 

> **MVC 패턴은?**
- M(Model)
  : Controller에서 비즈니스 로직을 수행하고 나온 결과 데이터를 View로 전달하기 위해 담아두는 영역.
- V(View)
  : Model에 담겨진 결과 데이터(동적 데이터)를 바탕으로 클라이언트에게 전달할 화면을 렌더링하는 작업을 수행하는 영역.
- C(Controller)
  : HTTP 요청을 받아 파라미터를 검증하고 비즈니스 로직을 수행하는 영역.
{: .prompt-info }

먼저 servlet을 사용한 java code를 보자.

비즈니스 로직을 처리하는 Controller의 역할(DB 조회)을 수행하고 HttpServletRequest 객체의 setAttribute() 메서드를 사용해서
View로 넘길 동적 데이터 담아둔다. 동적 데이터를 전달할 View 경로를 파리미터로 RequestDispatcher 객체를 가져와 forward() 메서드를 통해
HttpServletRequest, HttpServletResponse를 넘긴다. HttpServletRequest 객체 속에는 동적으로 생성된 데이터가 들어가 있으니 View 단에서 데이터를 읽고 렌더링 한다.
(실제 비즈니스 로직은 Controller 를 통한 Service 영역에서 처리하는 것이 일반적이지만, 간단한 흐름을 보기 위함으로 생략한다.)

이어서 jsp code를 보자.

only jsp 방식에서 존재하던 비즈니스 영역의 로직뿐만 아니라 list를 반복문으로 출력하던 java code가 깔끔하게 사라졌다. 여기서는 java code 대신 `jstl`이라는 jsp 개발을 도와주는 태그 라이브러리를 통해 반복문을 구현했다.

(`<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"%>` 태그를 jsp 파일 상단에 작성하고 `jstl` 라이브러리가 프로젝트에 포함되어 있어야 한다.) 

MVC 패턴을 통해 Controller(비즈니스 처리 영역), View(뷰 렌더링), Model(데이터 전달) 구분하여 유지보수를 쉽게 만들고 OOP의 SRP(Single Responsiblity Principle)도 어느정도 지켜지는 것으로 보인다. 

여기서 MVC 패턴을 차용해서 좀 더 갈고 닦아 발전시킨 Spring MVC가 등장한다.

> **/WEB-INF**   
`/WEB-INF` 하위 경로는 외부에서 직접 파일을 호출할 수 없도록 막혀있다. 즉 https://도메인/WEB-INF/views/users.jsp 이 불가능하다. 이것은 View 영역의 접근은 Controller 영역을 통해서만 이루어져야 하는 MVC 패턴을 적절하게 지킨 방법이다.
{: .prompt-warning }

## 4. spring MVC

```java
@Controller
public class UserController {

    // DB 역할
    private UserRepository userRepository = UserRepository.getInstance();

    @GetMappong("/users")
    public String getList(Model model) {
      List<User> users = userRepository.findAll();
      model.addAttribute("users", users);
      return "users";
    }
    
}
```
위 소스는 오늘날 가장 많이 쓰이고 알려진 형태의 Spring MVC 형태이다. 눈에 띄는 차이점은 servlet을 사용하기 위한 `HttpServlet` 상속을 받는 부분이 사라졌다.
그럼 servlet은 어디로 간 것일까?

servlet은 이미 존재하고 있는데 그것은 Spring MVC가 앞단에서 모든 HTTP 요청을 받아줄 목적으로 `DispatcherServlet`라는 이름으로 만들어 둔 것이다. (앞에서 모든 모든 요청을 받기 때문에 FrontController라고도 불리운다.)


![Desktop View](/assets/img/post/20230915/1.JPG){: width="600" height="600" }
_DispatcherServlet 계층도_

모든 요청은 `DispatcherServlet`을 통해서 이루어지며 url에 매핑되는 Contoller에게 요청을 위임함으로써 Controller단은 servlet에 종속적이지 않게된다.
![Desktop View](/assets/img/post/20230915/2.JPG){: width="600" height="600" }

소스를 이어서 확인해보면 @GetMappong 애노테이션을 통해 클라이언트가 호출한 경로의 HTTP 요청을 받는다는 것을 알 수 있다.
호출 경로와 매핑되는 getList() 메서드를 살펴보면 메서드의 argument인 `Model`과 return 타입이 String인 것이 눈에 띈다.
`Model`은 `DispatcherServlet`가 생성해서 제공하는 객체이며, View로 전달할 데이터를 보관해주는 MVC의 Model 역할을 그대로 수행한다.(이름도 같은 Model..)

Contoller의 return 값은 View의 논리명(jsp파일명)을 의미하고 설정에 따른 prefix와 suffix를 조합해서 물리명(jsp파일 전체경로)으로 변환한다. 
> **springboot에 적용된 기본 설정값**
  prefix = /WEB-INF/views/   
  suffix = .jsp

물리명의 파일을 찾아 `Model` 속 동적 데이터를 바탕으로 HTML을 렌더링하고 클라이언트에게 응답한다.

>
  - Controller에서 사용 가능한 arguments 목록 
  : <https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods/arguments.html>
  - Controller에서 사용 가능한 return 목록 
  : <https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods/return-types.html>
  {: .prompt-tip }

----

간단한 소스와 함께 servlet에서 spring MVC까지 정리를 흐름을 정리해보았다. 
Spring MVC로 코드를 작성하면 개발자가 직접 sevlet 코드를 다룰 일이 없지만 Java 웹 프로그래밍의 전반적인 흐름을 이해하기 위해서는 꼭 이해하고 넘어가는 것이 중요하다.  
