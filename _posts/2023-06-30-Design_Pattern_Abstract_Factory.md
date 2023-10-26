---
title: 🎨 Design Pattern -  추상 팩토리 패턴 빠르게 알아보기
author: mingo
date: 2023-06-30 23:30:00 +0900
categories: [design pattern, creational pattern]
tags: [추상 팩토리 패턴, abstract factory, design pattern, creational]
---

----

## | 추상 팩토리 패턴
추상 팩토리 패턴은 **관련된 객체들의 집합을 생성하는 인터페이스를 제공하며, 
이를 통해 다양한 객체 생성 팩토리 클래스가 구체적인 객체를 생성할 수 있게 하는 디자인 패턴**이다. 
이 패턴은 객체 생성에 관한 복잡성을 추상화하고, 클라이언트 코드로부터 구체적인 클래스의 결합을 분리하여 
시스템의 유연성과 확장성을 향상시킨다.

팩토리 메서드 패턴의 묶음으로 생각하면 된다. 
팩토리 메서드 패턴은 객체 생성에 집중하고 추상 팩토리는 **관련성이 높은 객체들의 구현을 묶는 것**에 초점을 맞춘다.

## | 추상 팩토리 패턴 특징

### 👍 객체 관련성 응집도 향상
추상 팩토리 패턴은 관련된 객체의 집합을 생성하는데 사용되므로 객체 간의 관련성을 유지하고 일관성 있는 객체 그룹을 생성할 수 있다.
각 추상 팩토리는 특정 객체 집합을 생성하는 역할을 가지므로, 단일 책임 원칙(Single Responsibility Principle, SRP)을 준수한다.

### 👍 클라이언트 코드의 독립
클라이언트 코드는 구체적인 클래스를 알 필요가 없고 팩토리 인터페이스를 통해 객체가 생성된다.

### 👍 유연성 확장
새로운 종류의 객체 집합을 추가하려면 해당 객체 집합에 대한 새로운 추상 팩토리를 구현하기만 하면 된다.  

### 👎 복잡성
각각의 추상 팩토리, 구체 팩토리, 객체 등 수많은 클래스로 인하여 복잡도가 올라간다.

## | 추상 팩토리 패턴 구현
```java
public interface Button {
    void render();
}

public class DarkButton implements Button {
  @Override
  public void render() {
    System.out.println("어두운 테마 버튼을 렌더링합니다.");
  }
}

public class LightButton implements Button {
  @Override
  public void render() {
    System.out.println("밝은 테마 버튼을 렌더링합니다.");
  }
}
```
 - Button의 렌더링 행위를 인터페이스로 설계하고 어두운 버튼, 밝은 버튼을 각각 인터페이스를 구현하여 구체화 한다.

```java
public interface TextBox {
    void display();
}

public class DarkTextBox implements TextBox {
  @Override
  public void display() {
    System.out.println("어두운 테마 텍스트 상자를 표시합니다.");
  }
}

public class LightTextBox implements TextBox {
  @Override
  public void display() {
    System.out.println("밝은 테마 텍스트 상자를 표시합니다.");
  }
}
```
 - TextBox의 디스플레이를 행위를 인터페이스로 설계하고 어두운 텍스트 박스, 밝은 텍스트 박스를 각각 인터페이스를 구현하여 구체화 한다.

```java
public interface ThemeFactory {

    Button createButton();
    TextBox createTextBox();

}

public class DarkThemeFactory implements ThemeFactory {
  @Override
  public Button createButton() {
    return new DarkButton();
  }

  @Override
  public TextBox createTextBox() {
    return new DarkTextBox();
  }
}

public class LightThemeFactory implements ThemeFactory {
  @Override
  public Button createButton() {
    return new LightButton();
  }

  @Override
  public TextBox createTextBox() {
    return new LightTextBox();
  }
}
```
 - ThemeFactory 팩토리를 통해 관련된 `객체들의 집합을 생성하는 인터페이스` 정의
 - '어두운테마 팩토리', '밝은테마 팩토리'라는 `객체 집합 인터페이스를 구현한 팩토리`를 통해 실제 사용되는 객체를 생성

```java
public static void main(String[] args) {
    ThemeFactory darkThemeFactory = new DarkThemeFactory();
    Button darkButton = darkThemeFactory.createButton();
    TextBox darkTextBox = darkThemeFactory.createTextBox();

    darkButton.render();
    darkTextBox.display();

    ThemeFactory lightThemeFactory = new LightThemeFactory();
    Button lightButton = lightThemeFactory.createButton();
    TextBox lightTextBox = lightThemeFactory.createTextBox();

    lightButton.render();
    lightTextBox.display();
}
```
 - 클라이언트 코드를 의미
 - 클라이언트 코드는 구현체가 아닌 인터페이스에 의존하고 있다.
 - 공통화된 기능이 있는 객체 집합은 각각 어두운 버튼, 어두운 텍스트박스 / 밝은 버튼, 밝은 텍스트박스로 구현되어 있지만 클라이언트는 어떤 객체를 통해 구현되었는지 알 수 없고 알 필요도 없다.
