---
title: Effective Java - [item26] 로 타입은 사용하지 말라
author: mingo
date: 2023-10-30 19:30:00 +0900
categories: [Effective Java]
tags: [effective java, 5장]
---

----

# 5장 - [아이템 26] - 로 타입은 사용하지 말라

## 한줄 요약
> 로 타입(raw type)은 제네릭이 도입되기 전에 사용하던 코드와의 호환성을 위해 제공될 뿐이지 런타임 중 에러가 발생할 수 있으니 사용하지 말아야 한다. 

## 로 타입은 사용하지 말라
클래스나 인터페이스 선언에 타입 매개변수가 쓰이면 `제네릭 클래스`, `제네릭 인터페이스`라고 부른다. 그리고 이 둘을 `제네릭 타입`이라 한다.
제네릭 타입은 매개변수화 타입에 대하여 어떤 타입을 받을 건지 제네릭 타입을 사용하는 부분에서 정의해야 한다. 

`List` 같이 제네릭 타입에서 매개변수화를 전혀 사용하지 않은 것을 로 타입(raw type)이라고 정의한다. 로 타입의 사용은 제네릭이 도입되기 전(jdk1.5 이전) 코드와
호환되도록 하기 위한 *궁여지책*이라고 한다.

```java
Collection stamp = new ArrayList();
stamp.add(new Stamp()); // 단순 경고 : Unchecked call to 'add(E)' as a member of raw type 'java.util.Collection'  
stamp.add(new Coin()); // 단순 경고 : Unchecked call to 'add(E)' as a member of raw type 'java.util.Collection' 
```
![Desktop View](/assets/img/post/202310/1.png){: width="300" height="150" }

로 타입으로 제네릭 타입을 사용하면 Unchecked call 경고가 뜨지만 정상적으로 컴파일이 되기 때문에 
실제 컬렉션에서 요소를 꺼내와서 타입 캐스팅 시점에서 런타임 error(ClassCastException)가 발생한다.

```java
Collection<Stamp> stamps = new ArrayList<>();
stamps.add(new Stamp());
stamps.add(new Coin()); // 컴파일 에러!!!
```
위와 같이 제네릭 타입에 사용하기를 원하는 타입에 대해서 매개변수화 시켜두면 컴파일러가 인지하고 다른 타입의 접근에 대해서 컴파일 오류를 발생시킨다.
개발자는 즉각 잘못된 부분을 찾아 대응하기 쉽다. 

**Java 진영에서 사용하지 말아야 할 로 타입(raw)을 열어둔 이유, 즉 컴파일 시점에서 잡히지 않도록 설계한 이유는 제네릭이 도입되기 전에 사용되던 코드와의 호환성 때문이다.**
`Collection stamp = new ArrayList();` 같은 기존 코드를 수용하고 제네릭 타입을 사용하는 새로운 코드도 서로 유기적으로 잘 동작해야 하기 때문에 로 타입을 살아남을 수 있엇던 것이다.

로 타입 `List`와 `List<Object>`의 차이는 무엇일까?
로 타입 `List`는 제네릭 자체를 사용하지 않았고 `List<Object>`는 모든 타입을 허용한다는 의미를 컴파일러에게 명확히 전달한다.
```java
List<String> strList = new ArrayList<>();
List<Object> objList = new ArrayList<>();

rawTypeMethod(strList);
rawTypeMethod(objList);

listObjTypeMethod(strList); // 컴파일 에러 발생 = List<String>은 List<Object>의 하위 타입이 아니다!!
listObjTypeMethod(objList);

// 로 타입을 매개변수로 받는 메서드
public static void rawTypeMethod(List list) {
    System.out.println(list.size());
}

// List<Object>를 매개변수로 받는 메서드
public static void listObjTypeMethod(List<Object> list) {
    System.out.println(list.size());
}
```
 - `List<String>`은 `List`의 하위 타입 -> 런타임 에러가 발생할 위험이 있음.
   - 로 타입을 통해 컴파일은 겨우 통과하지만 런타임 시점의 형변환 과정에서 `ClassCastException` 에러를 주의해야 한다.
 - `List<String>`은 `List<Object>`의 하위 타입이 아니다. -> 컴파일 에러가 발생되고 이것은 타입 안정성을 보장해준다는 의미와도 같다.

그럼 콜렉션 원소 타입을 모르는 상태에서 메서드를 작성하려면 어떻게 해야할까?
로타입인 `List`를 사용하면 `ClassCastException` 에러로 인한 런타임 에러를 주의해야 하고 `List<Object>`를 사용하면 컴파일 자체도 되지 않는다.

이럴 땐 **비한정적 와일드 카드 타입(unbounded wildcard type)**을 사용하는 것이 좋다. 
제네릭 타입을 쓰고 싶지만 실제 타입 매개변수가 무엇인지 신경 쓰고 싶지 않을 때 `?` 와일드 카드를 사용하면 된다.
비한정적 와일드 카드 타입을 메서드 매겨변수로 사용하면 어떤 타입도 담을 수 있는 가장 범용적인 타입을 제공하고 타입 안정성까지 가지게 된다.

```java
// 로 타입
List raw = new ArrayList<>();
raw.add(1);
raw.add("가나다");

// 비한정적 와일드 카드 타입
List<?> wild = new ArrayList<>();
wild.add(1); // 컴파일 에러
wild.add("가나다"); // 컴파일 에러
```
로 타입 컬렉션에는 아무값이나 넣을 수 있으나 비한정적 와일드 카드 타입  컬렉션은 null 외에는 그 어떤 원소도 추가 될 수 없다. 타입 불변식을 훼손하지 않도록 설계된 것이다.

런타임에는 제네릭 타입 정보가 **타입 소거** 메커니즘에 따라 지워지므로 instanceof 연산자로 타입을 비교할 때 주의해야 한다.
