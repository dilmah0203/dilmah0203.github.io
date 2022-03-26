---
layout: single
title: "**추상클래스와 인터페이스는 어떻게 구분하여 써야할까?**"
excerpt: ""
---

<br>

**추상클래스 vs 인터페이스**

- 추상클래스와 인터페이스는 인스턴스화 하는 것은 불가능하며,
- 구현부가 있는 메소드와 없는 메소드를 모두 가질 수 있다는 점에서 유사하다.
- 인터페이스에서 모든 변수는 기본적으로 public static final 이며, 모든 메소드는 public abstract인 반면 추상클래스는 static이나 final이 아닌 필드를 지정할 수 있고 public, protected, private 메소드를 가질 수 있다.
- 인터페이스는 다른 여러개의 인터페이스들을 함께 구현할 수 있지만 추상클래스는 상속을 통해 구현되는데 자바는 다중 상속을 지원하지 않으므로 추상클래스를 상속받은 자식클래스는 다른 클래스를 상속받을 수 없다.

<br>

**추상클래스**

```java
public abstract class 클래스이름 {
    //필드 선언
    //추상메소드
    public abstract void method();
    //일반메소드
    public void method2() {
    }
}
```
<br>

**언제 사용할까?**

- 여러 하위 클래스의 공통 기능을 캡슐화할 때

- 상태 변경을 위해 non-static, non-final 필드 선언이 필요할 때


```java
public abstract class Shape {

    abstract void draw();

    abstract double area();
}

public class Rectangle extends Shape {

    @Override
    public void draw() {
        ...
    }

    @Override
    public double area() {
        ...
    }
}

public class Main {

    public static void main(String[] args) {
        Shape shape = new Rectangle();
        shape.draw();
        System.out.println("area :" + shape.area());
    }
}
```

위의 코드는 Rectangle 객체를 생성했지만 상위 타입인 Shape로 객체 참조가 된다. 따라서, 메소드 호출 시 Shape 클래스의 메소드만 사용이 가능해진다. Rectangle과 유사한 다른 클래스를 사용하고 싶을 때에는 객체를 생성하도록 변경해주기만 하면 이후의 코드들은 전혀 수정될 필요가 없다.

다음 예시를 보자.

```java
public abstract class Shape {

    int x;

    public Shape(int x) {
        this.x = x;
    }

    abstract void draw();

    abstract double area();
}

public class Rectangle extends Shape {

    public Rectangle(int x) {
        super(x);
    }

    @Override
    public void draw() {
        ...
    }

    @Override
    public double area() {
        ...
    }
}

public class Triangle extends Shape {

    public Triangle(int x) {
        super(x);
    }

    @Override
    public void draw() {
         ...
    }

    @Override
    public double area() {
         ...
    }
}

```

Rectangle과 Triangle 클래스는 Shape 클래스를 확장한다. 이 클래스들 간의 관계를 Is A 라고 하며 결합도가 높다. int x를 추상 클래스에 선언함으로써 **상태에 관여**할 수 있다는 것이 인터페이스와의 큰 차이점이다.

<br>

**인터페이스**

```java
public interface 인터페이스이름 {
    //상수 필드
    //메소드
}
```

<br>

**언제 사용할까?**

- 구현 클래스들 간에 관련성이 없는 경우
- 특정 데이터 타입의 행동을 명시하고싶은데, 어디서 구현되는지 상관 없는 경우
- 다중상속을 활용할 때

<br>

다음 예시를 보자.

```java
public class Car {

    public void method() {
        System.out.println("car method");
    }
}
  
public class Main {

    public static void main(String[] args) {
        Car car = new Car();
        car.method();
    }
}
```

Main 클래스에서 다음과 같이 객체를 생성했다. 객체는 생성 이후 Car클래스에 선언된 모든 메소드를 사용할 수 있게 된다. 이것은 Car클래스와 Main클래스가 강한 결합을 이루고 있다고 볼 수 있으며 인터페이스를 이용하여 다음과 같이 바꿀 수 있다.

```java
public class Example {

    void auto(Movable m) {
        m.method();
    }

}

public interface Movable {

    public void method();
}

public class Car implements Movable {

    @Override
    public void method() {
        System.out.println("method in Car class");
    }
}

public class Bus implements Movable {

    @Override
    public void method() {
        System.out.println("method in Bus class");
    }
}

public class Main {

    public static void main(String[] args) {
        Example e = new Example();
        e.auto(new Car()); //method in Car class
        e.auto(new Bus()); //method in Bus class
    }
}

```

Movable 인터페이스를 상속받은 Car과 Bus클래스에 다른 동작을 선언하고 이 동작을 사용하는 Main 클래스에서는 객체만 바꿔주면 되기 때문에 다른 클래스에 영향을 미치지 않고 독립적인 프로그래밍이 가능해진다. 인터페이스를 통해 간접적인 관계를 맺음으로써 서로에게 영향을 주지 않는다.

<br>

**JDK 8의 인터페이스에 추가된 기능**

- default method

  JDK 8 이전엔 인터페이스에서 구현을 정의할 수 없었다. 그래서 새로운 메소드를 추가하려면 인터페이스를 구현하는 모든 클래스들이 해당 메소드를 구현해야했다.
  하지만 default method로 추가가 가능해지면서 기존 인터페이스를 구현했던 클래스에서 메소드를 override하지 않아도 된다.
  기존 코드에 영향을 주지 않고 새로운 메소드를 가질 수 있도록 이전 인터페이스에 대한 호환성을 제공한다.

```java
public interface Calculator {

    final int first = 10; //상수 필드

    public int plus(int x, int y); //메소드

    public int minus(int x, int y);

    default int multiply(int x, int y) {
        return x * y;
    }
}

public class Example implements Calculator {

    @Override
    public int plus(int x, int y) {
        return x + y;
    }

    @Override
    public int minus(int x, int y) {
        return x - y;
    }
}

public class Main {

    public static void main(String[] args) {
        Calculator calc = new Example();
        int value = calc.multiply(2, 3);
        System.out.println(value); //6
    }
}

```

<br>

- static method

  객체 생성 여부와 상관없이 사용이 가능하며, override가 불가능하고 상속되지 않는다.
  상수를 선언할 때는 static block에서 초기화할 수 없고, 선언과 동시에 초기화해야한다.
  간단한 기능을 가지는 유틸리티성 인터페이스를 만들 때 사용할 수 있다.

```java
public interface Calculator {

    final int first = 10; //상수 필드

    public int plus(int x, int y); //메소드

    public int minus(int x, int y);

    static int multiply(int x, int y) {
        return x * y;
    }
}

public class Ex implements Calculator {

    @Override
    public int plus(int x, int y) {
        return x + y;
    }

    @Override
    public int minus(int x, int y) {
        return x - y;
    }
}

public class Main {
    public static void main(String[] args) {
        int value = Calculator.multiply(5, 3);
        System.out.println((value));
    }
}
```

<br>

**정리**

추상클래스와 인터페이스는 추상메소드를 사용하고, 객체 생성이 불가능하다.

**추상클래스**는 추상메소드를 포함하는 것을 제외하면 일반 클래스와 다르지 않다. 추상클래스를 상속 받음으로써 자식 클래스들 간의 공통 기능을 구현하고 확장시킨다. 추상클래스는 is ~A(~이다) 이고, 여러 클래스들의 공통점을 찾아 추상화 시켜서 사용하는 것이 개발에서 이득일 때 사용한다.

**인터페이스**는 추상클래스보다 추상화 정도가 강해 몸통이 있는 일반 메소드나 멤버변수를 가질 수 없다. 추상 메소드를 정의해놓고 구현하는 클래스에서 각 기능들을 override하여 여러가지 형태로 구현할 수 있다. 상속을 받아 기능을 확장시키는 것이 목적인 추상클래스와는 달리, 구현 객체가 같은 동작을 한다는 것을 보장하기 위한 목적으로 사용한다.

<br>

참고

[https://docs.oracle.com/javase/tutorial/java/IandI/createinterface.html](https://docs.oracle.com/javase/tutorial/java/IandI/createinterface.html)

[https://docs.oracle.com/javase/tutorial/java/IandI/abstract.html](https://docs.oracle.com/javase/tutorial/java/IandI/abstract.html)

[https://www.geeksforgeeks.org/difference-between-abstract-class-and-interface-in-java/](https://www.geeksforgeeks.org/difference-between-abstract-class-and-interface-in-java/)

[https://www.journaldev.com/1607/difference-between-abstract-class-and-interface-in-java](https://www.journaldev.com/1607/difference-between-abstract-class-and-interface-in-java)


