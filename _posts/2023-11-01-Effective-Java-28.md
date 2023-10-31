---
title: Effective Java - [item28] 배열보다는 리스트를 사용하라 + 공변과 vs 불공변 비교
author: mingo
date: 2023-11-01 19:30:00 +0900
categories: [Effective Java]
tags: [effective java, 공변, 불공변]
---

----

# 5장 - [아이템 28] - 배열보다는 리스트를 사용하라

## 한줄 요약
>  공변성을 가진 배열은 런타임 에러를 발생시킬 위험이 있으므로 불공변성을 가진 리스트 사용하자

## 배열보다는 리스트를 사용하라
배열과 제네릭 타입에는 두가지 차이가 있다.

### 1. 배열은 공변(covariant)이고 제네릭 타입은 불공변(invariant)이다.
> A 타입이 B 타입의 서브 타입일 때, A 배열은 B 배열의 서브 타입이 될 수 있으면 공변이고 A 리스트는 B 리스트의 서브 타입이 될 수 없으면 불공변이다.

**배열은 공변이**란 말은 즉, `A` 타입이 `B` 타입의 서브 타입 일 때, `A[]`(A 타입의 배열)은 `B[]`(B 타입의 배열)의 서브 타입이라는 것이다. 
```java
static class GrandFather {}
static class Father extends GrandFather {}

public static void main(String[] args) {
  GrandFather[] grandFathers = new Father[1]; // OK
}
```
코드를 보면 `GrandFather` 클래스를 상속한 서브 타입 `Father` 클래스는 배열의 **공변성**으로 인하여 `Father[]` 배열도 `GrandFather[]` 배열의 서브 타입으로 인정되어 다형성을 통한 참조가 가능한 것이다. 

하지만 배열의 공변성이 마냥 좋은 것은 아니다.
```java
Object[] objectArray = new List[1];
objectArray[0] = "str"; // 런타임에 ArrayStoreException 발생...ㅠ
```
이 코드 또한 배열의 **공변성**을 잘 나타낸 부분이다. 공변성의 특성으로 인하여 `List`는 `Object`의 서브 타입이면 `List[]` 도 `Object[]`의 서브 타입이 된다. 
`Object[]` 배열에 String 타입을 주입하는 것은 어색하지 않기에 실행해보면 실제 타입은 `List`라서 런타임 에러가 발생한다. 

반면 **제네릭 불공변**이란, 타입 관계가 변하지 않고 불변하게 유지되는 경우를 말한다. 
즉, `A` 타입이 `B` 타입의 서브 타입 일 때, `List<A>`(A 타입 리스트)는 `List<B>`(B 타입 리스트)의 서브 타입이 아니다.
```java
static class GrandFather {}
static class Father extends GrandFather {}

public static void main(String[] args) {
  ArrayList<GrandFather> fathers = new ArrayList<Father>(); // 컴파일 에러...!!
}
```
`Father`은 `GrandFather`의 서브 타입이지만 `ArrayList<Father>`과 `ArrayList<GrandFather>`은 서브 타입이 아니다. 
이러한 컴파일 에러를 통해 캐스팅 에러를 방지할 수 있는 리스트가 더 자주 사용되어야만 하는 이유 중 하나이다. 


### 2. 배열은 실체화(reify)된다.
> 배열은 런타임에 원소의 타입을 동적으로 인지하여 런타임 에러를 발생시킬 수 있고 제네릭은 타입 소거로 인하여 실체화 될 수 없으므로 컴파일 에러가 발생한다. 컴파일 에러 좋아요~

배열은 런타임에도 자신이 담기로 한 원소의 타입을 동적으로 인지한다. `Long` 타입 배열에 `String` 타입을 넣으려고 하면 에러가 나는 것을 생각하자.

하지만 제네릭은 컴파일 시점에 **타입 소거**가 발생하므로 런타임 시점에 로 타입(raw type)이므로 타입 정보가 없다. 이 말은 제네릭은 실체화되지 않는다는 것을 의미한다.
이 차이로 인하여 배열은 제네릭 타입으로 작성한 `new List<String>[]`, `new List<E>[]`, `new E[]` 코드 컴파일 에러가 발생한다. 제네릭 배열은 타입이 안전하지 않기 때문에 컴파일러가 만들지 못하게 막은 것이다. 

`E`, `List<E>`, `List<String>` 같은 제네릭 타입을 *실체화 불가 타입*이라고 하고 
실체화가 되지 않는다는 것은 가질 수 있는 원소 타입이 적다는 것이고 그것은 타입 안정성 보장으로 귀결된다.

```java
static class Chooser {

    private final Object[] choiceArray;

    Chooser(Object[] choiceArray) {
        this.choiceArray = choiceArray;
    }

    public Object choose() {
        ThreadLocalRandom current = ThreadLocalRandom.current();
        return choiceArray[current.nextInt(choiceArray.length)];
    }

}
```

```java
public static void main(String[] args) {

    Chooser numChooser = new Chooser(new Object[]{1, 2, 3});
    int ch1 = (int) numChooser.choose();
    System.out.println(ch1);

    Chooser strChooser = new Chooser(new String[]{"가", "나", "다"});
    String ch2 = (String) strChooser.choose();
    System.out.println(ch2);

}
```
`Chooser` 클래스의 choose 메서드를 호출할 때마다 반환되는 Object 객체를 원하는 타입으로 형 변환해야 한다. 
혹시나 numChooser에 `new Object[]{1, "나", 3}` 와 같은 Object 배열의 요소가 실수로 들어가도 
컴파일 시점에서는 찾을 수 없고 런타임 시점에 실제 choose() 메서드를 사용해서 형 변환하는 순간 Casting Exception이 발생한다.

`Chooser` 클래스를 제네릭 타입으로 바꿔보자
```java
static class Chooser2<T> {

    private final T[] choiceArray;

    Chooser2(Collection<T> choices) {
        this.choiceArray = (T[]) choices.toArray(); // 경고...!
    }

    public T choose() {
        ThreadLocalRandom current = ThreadLocalRandom.current();
        return choiceArray[current.nextInt(choiceArray.length)];
    }

}
```
Object[]은 T[]로 형변환이 가능하니 형 변환을 진행하였더니,
```text
Unchecked cast: 'java.lang.Object[]' to 'T[]' 
```
형 변환이 안전한지 보장할 수 없다는 경고 문구가 뜨게 된다. 
제네릭은 컴파일 시점에 **타입 소거**가 이루어져 런타임에는 타입을 알 수 없다. 
이 코드는 동작하긴 하지만 컴파일러가 안전성을 보장해주지 않는 코드이며 그것을 컴파일러는 개발자로 하여금 경각심을 느끼도록 경고 문구를 보여주는 것이다.

개발자가 안전하다고 확신이 들면 `@SuppressWarnings("unchecked")` 애노테이션으로 경고를 숨겨도 되고 경고의 원인을 제거해도 된다.

경고를 제거하기 위해서는 리스트를 사용해보자.
```java
static class Chooser3<T> {

    private final List<T> choiceList;

    Chooser3(Collection<T> choices) {
        this.choiceList = new ArrayList<>(choices);
    }

    public T choose() {
        ThreadLocalRandom current = ThreadLocalRandom.current();
        return choiceList.get(current.nextInt(choiceList.size()));
    }

}
```
배열을 리스트로 바꿔서 타입 안정성을 확보한다.
