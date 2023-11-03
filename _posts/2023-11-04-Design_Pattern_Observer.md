---
title: 🎨 Design Pattern - 옵서버 패턴 빠르게 알아보기
author: mingo
date: 2023-11-04 00:30:00 +0900
categories: [design pattern, behavioral pattern]
tags: [옵서버패턴, strategy, design pattern, behavioral]
---

----

## | 옵서버 패턴
옵서버 패턴은 디자인패턴 중에 행동(behavioral) 패턴으로 분류되는 패턴 중 하나로 **객체간 일대다 종속성을 정의하며 한 객체가 상태가 변경되면
다른 객체들에게 자동으로 상태를 통지하여 상태 변경을 처리할 수 있도록 하는 패턴이다.**

주제(Subject) 객체와 관찰(Observer) 객체로 나누고 주제 객체의 상태 변화를 관찰 객체에게 통지하는 개념이다. 
주제 객체는 내부에서 관찰 객체 목록을 런타임 시점에 조정할 수 있으며 주로 분산 이벤트 핸들링 시스템을 구현하는데 사용된다. 

주제 객체는 `Publisher 출판사`, 관찰 객체는 `Subscriber 구독자`라고도 불리운다.

![Desktop View](/assets/img/post/20230908/10.png){: width="800" height="800" }
_주제 객체와 관찰 객체 관계_

## | 옵서버 패턴 특징

### 👍 느슨한 결합(Loose Coupling)
주제(Subject) 객체와 관찰(Observer) 객체는 서로 독립적으로 존재하며 주제는 관찰자를 모르고 관찰자는 주제의 구체적인 형태를 모르기 때문에 느슨할 결합(Loose Coupling)이라고 할 수 있다.

### 👍 관찰(Observer) 객체가 주제(Subject) 객체를 주기적으로 감시하지 않는다.
주제 객체의 변화를 곧바로 관찰 객체에게 통지되기 때문에 관찰자가 주기적으로 주제의 상태를 확인하는 폴링기법을 사용하지 않아도 된다. 

### 👍 동적으로 관찰자를 추가/제거할 수 있다.
주제 객체 안에서 관리되는 관찰 객체 리스트는 런타임 시점에 추가되거나 제거될 수 있고 변경사항은 바로 적용 가능하다.

### 👎 성능고려
관찰자 목록 모두에게 통지를 하므로 성능을 고려해야 하고 관찰자로 등록이 되어있는 관찰 객체는 **가비지콜렉터의 대상이 되지 않는것**을 항상 염두해야 한다.

## | 옵서버 패턴 구현
```java
// 주제(Subject) or 출판사(Publisher)
static class TemperatureSensor {
    private List<Observer> observers = new ArrayList<>();
    private float temperature;

    public void addObserver(Observer observer) {
        observers.add(observer);
    }

    public void removeObserver(Observer observer) {
        observers.remove(observer);
    }

    public void setTemperature(float temperature) {
        this.temperature = temperature;
        notifyObservers();
    }

    private void notifyObservers() {
        for (Observer observer : observers) {
            observer.update(temperature); // 인터페이스 호출 -> 구독자 알람
        }
    }
}
```
 - 주제(Subject) 객체
 - 관찰(Observer) 객체 목록 관리
 - 다른 객체들의 관심 이벤트를 발행하는 주체
 - 동적으로 구독자가 추가되거나 제거될 수 있음

```java
// 옵서버(Observer) or 구독자 인터페이스
interface Observer {
    void update(float temperature);
}

// 구체적인 옵서버(Concrete Observer) or 구독자 구현체
static class Display implements Observer {
    @Override
    public void update(float temperature) {
        System.out.println("Temperature is now: " + temperature + "°C");
    }
}

// 구체적인 옵서버(Concrete Observer) or 구독자 구현체
static class Mail implements Observer {

    // 생성자 매개변수로 주제 객체를 전달 받아 생성과 동시에 구독
    public Mail(TemperatureSensor sensor) {
        sensor.addObserver(this);
    }

    @Override
    public void update(float temperature) {
        System.out.println("mail send : " + temperature + "°C");
    }
}
```
 - 관찰 객체 인터페이스
 - `Display`, `Mail` 두 가지 구독자 생성
 - 모든 구독자는 같은 인터페이스를 구현 / 출판사는 오직 해당 인터페이스를 통해서만 구독자와 통신 -> 객체간의 **`Loose Coupling`**
 - 인터페이스를 통해 구독자들은 출판사의 상태를 출판사의 구상 클래스와 결합하지 않고도 통지를 받음
 - 구독자가 업데이트를 올바르게 처리할 수 있도록 콘텍스트 정보가 어느 정도 필요하다. 예시로는 temperature를 인터페이스로 정의해서 사용했지만 **실제 출판사 객체 자체를 전달해서 구독자가 필요한 데이터를 뽑아내서 쓰는 경우도 많다.**

```java
TemperatureSensor sensor = new TemperatureSensor();
Display display = new Display();
sensor.addObserver(display); // 구독

Mail mail = new Mail(sensor); // 구독

sensor.setTemperature(25.5f); // 옵서버에게 알림을 보내고 출력
```
 - 구독자 리스트는 동적으로 설정할 수 있음

```text
// 출력
Temperature is now: 25.5°C
mail send : 25.5°C
```

## | 언제 사용하면 좋을까?
옵서버 패턴은 한 객체의 상태가 변경되어 다른 객체들의 상태도 변경되어야 할 필요성이 있을 때 사용하면 된다. 이 패턴 역시 OCP(개방-폐쇄 원칙)을 준수하여 구독자 추가되어도 다른 코드에는 전혀 영향을 미치지 않는다.

## | 마무리
Spring 프레임워크에서 `ApplicationEvent` 클래스(출판사)와  `ApplicationListener` 인터페이스(구독자 인터페이스)를 활용하여 이벤트 처리를 할 수 있다. 이 역시 옵서버 패턴을 기반으로 동작하는 것이다.


