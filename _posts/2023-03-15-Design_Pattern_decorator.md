---
title: 🎨 Design Pattern - 데코레이터 패턴 빠르게 알아보기
author: mingo
date: 2023-03-15 19:30:00 +0900
categories: [design pattern, structural  pattern]
tags: [데코레이터 패턴, decorator method, design pattern, structural]
---

----

## | 데코레이터 패턴
데이레이터 패턴은 구조(structural)패턴 중 하나이며, 기능별로 구분한 **객체의 *합성*을 통해 기능을 동적으로 유연하게 확장할 수 있게 해주는 디자인 패턴**이다.
기본 기능 외에 추가하려는 기능이 있다면, Decorator 클래스로 추가로 생성할 기능을 정의한 후 필요로 하는 객체를 서로 합성하여 기능의 확장을 도와주는 방법이다.

## | 데코레이터 패턴 특징

데코레이터 패턴는 아래와 같은 요소가 있다.
 - `Component(구성요소)` : 데코레이터 패턴의 핵심 인터페이스 또는 추상 클래스이며, 기본 객체와 데코레이터 클래스에 공통된 메서드를 정의한다.
 - `ConcreteComponent(구체적인 구성요소)` : Component 인터페이스를 구현하는 구체적인 클래스로 기본 기능을 제공하는 기본 객체이다. 기본 동작을 제공하며, 데코레이터 패턴의 시작점이며 이 기본 기능을 확장 시키는 것이 목적이다.
 - `Decorator(데코레이터)` : Component 인터페이스를 구현하고 ConcreteComponent(구체적인 구성요소)를 wrappee한다.
 - `ConcreteDecorator(구체적인 데코레이터)` : Decorator 클래스를 상속하고 추가적인 동작을 구현하는 구체적인 클래스 역할을 수행하며 부모로 위임 전/후로 추가적인 기능을 넣을 수 있다. 

### 👍 데코레이터 객체를 통한 다양한 조합 구성
데코레이터 객체를 적절하게 사용하면 상속 및 새로운 객체 생성없이 기능을 확장할 수 있다.

### 👍 기능의 동적인 추가 가능
데코레이터 객체를 조합하여 동적으로 기능을 추가하거나 제거할 수 있다.

### 👍 유연성 증대
추가되는 기능을 데코레이션 패턴으로 확장시켜 클라이언트 측에서 새로운 조합을 만들어 사용할 수 있도록 유연함을 제공한다.

### 👍 개방-폐쇄 원칙 준수
기능 추가 시, 기존 소스에 수정없이 새로운 기능을 추가할 수 있는 OCP를 준수하는 확장성을 가진다.

### 👎 복잡한 구조
데코레이션 클래스가 많이 추가되었을 경우 코드 복잡도가 올라가 이해하기 어려운 설계가 만들어질 수 있다. 

## | 데코레이터 패턴 구현

```java
public interface Notifier {
    int getAmount();
    void sendMessage();
}
```
 - Component(구성요소) 역할
 - 알림 인터페이스이며 금액 조회와 메시지 전송 메서드가 있음

```java
public class BasicNotifier implements Notifier {

    @Override
    public int getAmount() {
        return 0;
    }

    @Override
    public void sendMessage() {
        System.out.println("기본 Notifier는 전화를 통한 알림입니다.");
    }

}
```
 - ConcreteComponent(구체적인 구성요소) 역할
 - 기본 기능을 제공하는 기본 객체
 - Notifier를 구현한 구현체

```java
public abstract class NotifierDecorator implements Notifier {

    protected Notifier notifier;

    public NotifierDecorator(Notifier notifier) {
        this.notifier = notifier;
    }

    @Override
    public int getAmount() {
        return notifier.getAmount();
    }

    @Override
    public void sendMessage() {
        notifier.sendMessage();
    }

}
```
 - Decorator(데코레이터) 역할
 - Notifier를 구현한 구현체

```java
public class FacebookNotifier extends NotifierDecorator {


    public FacebookNotifier(Notifier decoratorNotifier) {
        super(decoratorNotifier);
    }

    @Override
    public int getAmount() {
        return super.getAmount() + 5;
    }

    @Override
    public void sendMessage() {
        super.sendMessage();
        System.out.println("Facebook로 알림을 전송합니다.");
    }

}

public class SlackNotifier extends NotifierDecorator {

  public SlackNotifier(Notifier decoratorNotifier) {
    super(decoratorNotifier);
  }

  @Override
  public int getAmount() {
    return super.getAmount() + 10;
  }

  @Override
  public void sendMessage() {
    super.sendMessage();
    System.out.println("Slack으로 알림을 전송합니다.");
  }

}

public class SMSNotifier extends NotifierDecorator {

  public SMSNotifier(Notifier decoratorNotifier) {
    super(decoratorNotifier);
  }

  @Override
  public int getAmount() {
    return super.getAmount() + 100;
  }

  @Override
  public void sendMessage() {
    super.sendMessage();
    System.out.println("SMS로 알림을 전송합니다.");
  }

}
```
 - ConcreteDecorator(구체적인 데코레이터) 역할
 - NotifierDecorator를 상속받아 구체적인 기능을 추가한 구현체
 - super 키워드를 통해 부모 클래스로 위임하면서 동시에 추가기능이 있다면 수행

```java
public static void main(String[] args) {

    Notifier basic = new BasicNotifier();
    Notifier slack = new SlackNotifier(basic);
    Notifier slackPlusSms1 = new SMSNotifier(slack);

    System.out.println(slackPlusSms1.getAmount());
    slackPlusSms1.sendMessage();

    SMSNotifier slackWithSms2 = new SMSNotifier(new SlackNotifier(new BasicNotifier()));
    System.out.println(slackWithSms2.getAmount());
    slackWithSms2.sendMessage();

}
```

```text
// 출력
110
기본 Notifier는 전화를 통한 알림입니다.
Slack으로 알림을 전송합니다.
SMS로 알림을 전송합니다.
110
기본 Notifier는 전화를 통한 알림입니다.
Slack으로 알림을 전송합니다.
SMS로 알림을 전송합니다.
```

 - 클라이언트 코드
 - 원하는 알림 옵션을 선택해서 동적으로 기능을 동작시킬 수 있다.
 - 위 코드로 보면 basic + slack + sms 알림 기능을 한번에 사용하고 있다.
 - 객체를 조합하여 여러가지 클라이언트의 요구사항을 적절하게 처리할 수 있다.
   - **만약 데코레이터 패턴이 적용되지 않았다면 SlackWithSmsNotifier, SlackWithSmsWithFacebookNotifier 등등.. 클래스가 계속 늘어난다.**
