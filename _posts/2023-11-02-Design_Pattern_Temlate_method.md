---
title: 🎨 Design Pattern - 템플릿 메서드 패턴 빠르게 알아보기
author: mingo
date: 2023-11-02 06:00:00 +0900
categories: [design pattern, behavioral pattern]
tags: [strategy, design pattern, behavioral]
---

----

## | 템플릿 메서드 패턴
템플릿 메서드 패턴은 행동 패턴 중 하나이며, 
**알고리즘 즉, 비즈니스 로직의 흐름 일 부분을 서브 클래스로 캡슐화하여 전체의 구조는 바뀌지 않고 서브 클래스에서 수행되는 로직만 바꾸어 중복코드를 제거하고 확장성을 갖는 디자인패턴**이다.

비즈니스 로직의 전체적인 흐름을 정의하는 메서드(== 템플릿 메서드)를 모아놓고 추상 클래스로 선언한다.
서브 클래스는 추상 클래스를 상속받아 구조는 그대로 이용하고 추가적으로 구현해야 할 부분만 작업한다.

> **Keyword** </br>
> `알고리즘 흐름`, `전체적인 구조`, `서브 클래스 위임`
{: .prompt-tip }
  

## | 템플릿 메서드 패턴 특징

### 👍 중복 코드 제거
로직의 구조를 추상 클래스로 만들고 기본 템플릿으로 제공하기 때문에 각 구체 클래스를 기본 템플릿을 확장하고 필요한 부분만 구현하면 된다.
중복 코드를 방지하고 유지보수를 용이하게 한다.

### 👍 코드 재활용
공통된 알고리즘 구조를 추상 클래스로 정의해두면 동일한 구조를 가져야하는 여러 서브 클래스에서 재사용될 수 있다.

### 👍 유연한 확장성
알고리즘의 구체적인 단계만 정의해두었기 때문에 각각 서브 클래스에서 원하는 단계를 개인 입맛에 맞게 유연한 확장을 할 수 있다. 

### 👎 템플릿 구조 수정되면 서브 클래스를 하나하나 수정해야 한다.
추상 클래스를 구현한 각각의 클래스가 필요하므로 클래스 수가 늘어나고 템플릿 구조가 바뀌면 모든 서브 클래스를 수정해야 할 수도 있다. 

## | 템플릿 메서드 패턴 구현
```java
/**
 * 템플릿 메서드 패턴으로 정의한 추상클래스
 */
public abstract class OrderProcessingTemplate {

    final void processOrder() {
        if (validateOrder()) {
            processPayment();
            shipOrder();
            sendNotification();
        } else {
            System.out.println("주문 유효성 검사 실패. 주문 처리 중지.");
        }
    }

    abstract boolean validateOrder();

    abstract void processPayment();

    abstract void shipOrder();

    abstract void sendNotification();

}
```
 - 알고리즘, 비즈니스 로직의 구조를 만드는 추상클래스 설계

```java
class OnlineOrderProcessing extends OrderProcessingTemplate {

    @Override
    boolean validateOrder() {
        System.out.println("주문 유효성 검사를 수행합니다.");
        // 유효성 검사 로직
        return true; // 간단한 예시로 항상 유효성 검사 통과로 가정
    }

    @Override
    void processPayment() {
        System.out.println("결제를 처리합니다.");
        // 결제 처리 로직
    }

    @Override
    void shipOrder() {
        System.out.println("주문을 배송합니다.");
        // 배송 처리 로직
    }

    @Override
    void sendNotification() {
        System.out.println("고객에게 알림을 전송합니다.");
        // 알림 전송 로직
    }

}
```
 - 추상 클래스 구현체 1

```java
class OfflineOrderProcessing extends OrderProcessingTemplate {
    
    @Override
    boolean validateOrder() {
        System.out.println("오프라인 주문 유효성 검사를 수행합니다.");
        // 유효성 검사 로직 (오프라인 주문의 경우 다르게 구현)
        return true; // 간단한 예시로 항상 유효성 검사 통과로 가정
    }

    @Override
    void processPayment() {
        System.out.println("오프라인 결제를 처리합니다.");
        // 결제 처리 로직 (오프라인 주문의 경우 다르게 구현)
    }

    @Override
    void shipOrder() {
        System.out.println("오프라인 주문을 배송합니다.");
        // 배송 처리 로직 (오프라인 주문의 경우 다르게 구현)
    }

    @Override
    void sendNotification() {
        System.out.println("고객에게 오프라인 알림을 전송합니다.");
        // 알림 전송 로직 (오프라인 주문의 경우 다르게 구현)
    }
    
}
```
 - 추상 클래스 구현체 2
 - 공통되는 메서드가 존재한다면 추상 클래스에서 공통으로 처리할 메서드를 구현하면 된다.

```java
public static void main(String[] args) {

    OrderProcessingTemplate onlineOrder = new OnlineOrderProcessing();
    OrderProcessingTemplate offlineOrder = new OfflineOrderProcessing();
    
    System.out.println("Online 주문 처리:");
    onlineOrder.processOrder();
    
    System.out.println("Offline 주문 처리:");
    offlineOrder.processOrder();

}
```

## | 다른 패턴과 비교

#### 전략 패턴(strategy pattern)과 비교
전략 패턴의 목적은 알고리즘을 캡슐화하고 런타임에 알고리즘을 교체할 수 있도록 설계하는 것이다. 알고리즘 자체가 독립적인 기능을 하여 클라이언트가 선택할 수 있도록 하는 패턴이다.
(클라이언트가 동적으로 선택한 알고리즘을 콘크리트 클래스에서 수행)

반면 템플릿 메서드 패턴은 알고리즘의 전체 구조를 제어하면서 일부 단계를 서브 클래스에게 위임하는 것을 목적으로 한다.

#### 프록시 패턴(proxy pattern)과 비교
프록시 패턴은 객체에 대한 접근을 제어하거나 객체에 추가적인 기능을 제공하기 위해 중간에 대리자(Proxy)를 두는 것이 목적이다.

## | 마무리
템플릿 메서드 패턴은 약간의 차이가 있지만 거의 같은 알고리즘, 비즈니스 로직이 반복되는 여러 클래스가 있을 때에만 사용하는 것이 좋다. 
알고리즘이 변경되면 모든 서브 클래스가 변경되야하는 불상사가 벌어질 수도 있다.
또 템플릿 메서드 패턴은 일반적으로 인터페이스보다 추상 클래스로 설계되는 경우가 더 많다. 
인터페이스를 통하면 다중 상속이 가능하지만 구현이 강제되므로 서브 클래스에서 굳이 사용할 필요 없는 메서드를 구현해야 할 수도 있기 때문이다.
