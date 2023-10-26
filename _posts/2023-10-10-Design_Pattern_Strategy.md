---
title: 🎨 Design Pattern - 전략 패턴 빠르게 알아보기
author: mingo
date: 2023-10-10 19:30:00 +0900
categories: [design pattern, behavioral pattern]
tags: [전략패턴, strategy, design pattern, behavioral]
---

----

## | 전략 패턴
전략 패턴은 **공통 관심사 또는 행위를 캡슐화 하여 인터페이스로 정의하고 관심사를 구현한 여러가지 방식 또는 알고리즘을 
각각의 `전략(strategy)` 클래스로 만들어 클라이언트에게 제공**하여 런타임 중에 전략을 동적으로 변경할 수 있는 유연함과 확장성을 가지도록 설계된 디자인 패턴이다.  

## | 전략 패턴 특징

### 유지보수 향상
전략 패턴은 객체지향의 원칙 중 하나인 개방-폐쇄 원칙(Open-Closed Principle, OCP)를 지키고 있는 디자인패턴이다. 
새로운 전략이 추가된다면 전략 클래스를 생성하면 되고 다른 전략 클래스 또는 콘텍스트에도 영향이 없으니 수정사항이 없어 확장에 열려있고 수정에 닫혀 있게 된다. 
이로 인해 유지보수성이 아주 뛰어나다.

### 코드 재활용
각각의 전략은 독립적으로 구현되어 있기 때문에 동일한 전략을 재활용하여 사용할 수 있다.

### 전략을 통한 유연함
클라이언트 코드를 통해 동적으로 전략이 바뀌어 들어올 수 있기 때문에 시스템 전반적인 유연성을 제공한다.

### 코드의 응집성
콘텍스트 클래스는 어떤 알고리즘이 사용되었는지, 어떤 전략이 선택되었는지 알 필요가 없고 클라이언트로부터 전달받은 전략을 작동시키기만 하면 된다.
이 말은 즉, 객체의 사용의 책임과 구현의 책임을 명확하게 구분하고 있기 때문에 단일책임 원칙(Single Responsiblity Principle, SRP)도 지킨다고 볼 수 있다.

## | 전략 패턴 구현
```java
public interface PartnerCompany {

   void callApi(Long id, int amount);

}
```
 - 파트너사로 API 호출하는 것을 공통 관심사로 캡슐화

```java
public class Amazon implements PartnerCompany {

    private static String HOST = "www.amazon.com";
    private static String PORT = "8080";

    @Override
    public void callApi(Long id, int amount) {
        AmazonBody amazonBody = AmazonBody.of(id, amount);
        Unirest.post(HOST + ":" + PORT).body(amazonBody);
    }

}

public class Naver implements PartnerCompany {

  private static String HOST = "www.naver.com";
  private static String PORT = "443";

  @Override
  public void callApi(Long id, int amount) {
    NaverBody naverBody = NaverBody.of(id, amount);
    Unirest.post(HOST + ":" + PORT).body(naverBody);
  }

}
```

```java
@Getter
public class AmazonBody {

    private Long saleId;
    private int saleAmount;

    @Builder
    public AmazonBody(Long saleId, int saleAmount) {
        this.saleId = saleId;
        this.saleAmount = saleAmount;
    }

    public static AmazonBody of(Long id, int amount) {
        return AmazonBody.builder()
                .saleId(id)
                .saleAmount(amount)
                .build();
    }

}

@Getter
public class NaverBody {

  private Long productId;
  private int productPrice;

  @Builder
  public NaverBody(Long productId, int productPrice) {
    this.productId = productId;
    this.productPrice = productPrice;
  }

  public static NaverBody of(Long id, int amount) {
    return NaverBody.builder()
      .productId(id)
      .productPrice(amount)
      .build();
  }

}
```
 - 파트너사를 구현한 전략(strategy) 클래스
 - 전략 클래스는 회사의 공유한 host와 port, body 필드를 가지고 있다.

```java
// 클라이언트 코드
public static void main(String[] args) {

    ApiInvoke apiInvoke = new ApiInvoke();
    Naver naver = new Naver();
    Amazon amazon = new Amazon();

    apiInvoke.setPartnerCompany(naver);
    apiInvoke.invoke(1L, 2000);

    apiInvoke.setPartnerCompany(amazon);
    apiInvoke.invoke(3L, 4000);
}

// 콘텍스트 클래스
static class ApiInvoke {

    private PartnerCompany partnerCompany;

    public void setPartnerCompany(PartnerCompany partnerCompany) {
        this.partnerCompany = partnerCompany;
    }

    public void invoke(long id, int amount) {
        partnerCompany.callApi(id, amount);
    }

}
```
 - 클라이언트 코드를 통해 전략을 선택한다.
 - 콘텍스트 클래스는 클라이언트 코드로부터 전달받은 구체적인 전략(strategy)에 대해서 알 필요가 없다. 사용 방법만 알면 된다. -> **전략 클래스와 강학 분리**
 - 새로운 컴퍼니가 추가되어도 컴퍼니 전략(strategy)과 사용하는 클라이언트 코드만 추가하면 된다. 확장에는 열려있고 수정에는 닫혀있는 OCP원칙이다.

## | 마무리
전략패턴은 디자인패턴의 꽃이라고도 많이 표현된다. 그만큼 널리 사용되고 있어서 프레임워크나 라이브러리 소스를 열다 보면 쉽게 발견할 수 있다. 
예를 들어 스프링의 RequestMappingHandlerMapping 같은 객체도 전략 패턴의 콘텍스트 클래스 역할을 수행한다.
전략패턴을 공부하다 보면 항상 사내 여러가지 코드를 리팩토링하고 싶은 욕구가 차오른다. (위 예제코드도 사실 회사 로직을 바꾸고 싶은 부분을...) 
상황과 목적에 맞는 구체적인 설계를 통해 바꿔가는 것도 재미있는 과제가 될 것 같다.
