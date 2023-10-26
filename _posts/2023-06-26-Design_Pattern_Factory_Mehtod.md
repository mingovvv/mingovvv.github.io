---
title: 🎨 Design Pattern - 팩토리 메서드 패턴 빠르게 알아보기
author: mingo
date: 2023-06-26 19:30:00 +0900
categories: [design pattern, creational pattern]
tags: [팩터리 메서드 패턴, factory method, design pattern, creational]
---

----

## | 팩토리 메서드 패턴
팩토리 메서드 패턴은 생성 디자인 패턴 중 하나로 **객체를 생성할 때 어떤 클래스의 인스턴스를 만들지 서브 클래스에 위임하는 디자인 패턴**이다.
객체 생성에 대한 구체적인 로직을 클라이언트 코드로부터 분리하여 객체 생성의 유연성과 확장성을 제공한다.

팩토리 메서드 패턴은 아래와 같은 구조로 이루어져 있다.
 - > `Creator` 생성자
   - 객체를 생성하는 추상 클래스 또는 인터페이스를 정의한다. 추상 클래스 또는 인터페이스는 팩토리 메서드를 선언하고 이 메서드를 구현하는 서브 클래스에서 실제 객체 생성을 진행한다.
 - > `ConcreteCreator` 구체적 생성자
   - `Creator`에서 정의한 팩토리 메서드를 구현한다. 이 클래스를 통해 생성된 객체를 클라이언트 코드로 반환한다.
 - > `Product` 제품
   - 생성될 객체를 의미하는 추상 클래스 또는 인터페이스를 정의한다.
 - > `ConcreateProduct` 구체적 제품
   - `Product`를 구현한 구체적인 제품 클래스. 팩토리 메서드 패턴의 목표는 구체적 제품을 생성하는 것이고 각각의 `ConcreteCreator` 클래스를 통해 생성된다.

## | 팩토리 메서드 패턴 특징

### 👍객체 생성의 분리
팩토리 메서드 패턴은 객체 생성을 클라이언트 코드로부터 분리하고 정의된 팩토리 메서드에 위임한다. 
클라이언트는 팩토리 메서드를 통해 객체가 생성되니 실제 생성되는 객체의 생성구조(생성자 파라미터, 클래스 명)에 대하여 언커플링되어 결합도를 낮추며 실제 객체의 생성 로직은 은폐하는 효과가 있다.

### 👍 다형성 활용
팩토리 메서드 패턴은 서브 클래스가 어떤 클래스의 인스턴스를 생성할 지 결정할 수 있으므로 
클라이언트 코드에서는 실제 객체를 추상화한 인터페이스 또는 추상클래스에 의존하는 다형성의 특징을 지니고 있다.

### 👍 유연성 확장
새로운 객체(Product, ConcreateProduct)를 추가하려면 팩토리 메서드를 구현하는 새로운 서브 클래스를 만들어주면 된다.
이것을 통해 OCP 원칙을 준수하며 기존의 코드를 수정하지 않고 새로운 객체를 사용할 수 있다.  

### 👎 클래스 수 증가
팩토리 메서드 패턴을 사용하면 객체 생성을 위한 각각의 팩토리 클래스를 만들어야 한다. 그로 인하여 클래스의 수가 늘어 과도한 복잡성을 띄게 될 수 있다.
여기서 오는 비슷한 코드의 중복도 단점이 될 수 있다.

## | 팩토리 메서드 패턴 구현
```java
public interface Channel {
    void sendMessage();
}
```
 - `Product`의 역할
 - 채널사로 메시지를 전달하는 메소드를 지닌 인터페이스

```java
public class NaverChannel implements Channel {
    @Override
    public void sendMessage() {
        System.out.println("send message to naver");
    }
}
```
 - `ConcreateProduct`의 역할
 - `Channel` 인터페이스의 구현체이며 실제 객체

```java
public abstract class ChannelFactory {
    protected abstract Channel createChannel();
    public Channel create() {
        Channel channel = createChannel();
        channel.sendMessage();
        return channel;
    }
}
```
 - `Creator`의 역할
 - 객체 생성에 대한 역할을 진행하는 인터페이스 또는 추상 클래스

```java
public class NaverChannelFactory extends ChannelFactory {
    @Override
    public Channel createChannel() {
        return new NaverChannel();
    }
}
```
 - `ConcreteCreator`의 역할
 - `Creator`을 상속 또는 구현하여 구체적으로 객체를 생성하는 클래스

```java
public class Test {

    public static void main(String[] args) {

        ChannelFactory naverChannelFactory = new NaverChannelFactory();
        Channel channel = naverChannelFactory.createChannel();
    }

}
```
 - 클라이언트 코드
 - 클라이언트는 실제 객체에 대하여 언커플링 되어있음
 - `NaverChannelFactory` 객체를 통해 채널 객체를 생성하고 반환 객체는 인터페이스 또는 추상 클래스를 참조
 - 만약 또 다른 채널이 추가된다면 `Channel` 인터페이스를 구현한 구체 클래스와 구체 클래스를 구현하기 위한 `ChannelFactory` 추상 클래스를 상속받은 구체 클래스를 추가하면 된다. 확장에는 열려있고 수정에는 닫혀있는 구조.
