---
layout: single
title : "**상속과 조합**"
excerpt : ""
---

<br>

**상속을 사용하는 이유**

- 기존 클래스의 필드와 메소드를 재사용함으로써 코드가 간결해진다
- 유지보수가 쉬워지며 개발시간이 단축된다

<br>

**상속의 단점**

**캡슐화를 깨트리고 결합도가 높아진다**

객체지향 프로그래밍에서는 결합도는 낮고 응집도는 높을수록 좋다. 하지만 상속을 이용하면 캡슐화는 깨지고 결합도는 높아진다. 이유는  상위 클래스의 구현이 바뀌면 이 클래스를 상속받은   하위클래스에도 영향을 미치기 때문이다.
따라서 변화에 유연하게 대처하기 어려워진다.


```java
public class Beverage {

    private int price;

    public Beverage(int price) { //생성자
        this.price = price;
    }

    public int calculatePrice() {
        return price;
    }
}

public class Coffee extends Beverage {

    public Coffee(int price) {
        super(price);
    }
}


```
<br>

커피와 토스트라는 세트 메뉴를 추가해야하며, 커피가 제공될 경우에는 할인이 되어야 하는 상황이라고 가정하자. 
이를 구현하기 위해 Coffee클래스에 discountPrice() 메소드를 추가한다.

```java
public class Coffee extends Beverage {

    public Coffee(int price) {
        super(price);
    }

    public int discountPrice() {
        return 1000;
    }
}


public class CoffeeToastSet extends Coffee {

    public CoffeeToastSet(int price) {
        super(price);
    }

    @Override
    public int calculatePrice() {
        return super.calculatePrice() - super.discountPrice();
    }
}

```
CoffeeToastSet클래스는 할인 금액을 뺀 원래의 금액을 계산하기 위해 Coffee 클래스에서 discountPrice()를 제공해야 함을 알고 있어야 한다. 즉, 상속을 위해서는 부모 클래스의 내부 구조를 잘 알고있어야 한다. 따라서 부모 클래스는 자식 클래스에게 노출되어 캡슐화가 약해지고 두 클래스는 강하게 결합되어 부모 클래스가 변경될 때 자식클래스도 변경될 가능성이 높아진다.

<br>

**유연성 및 확장성이 떨어진다**

예를들어, 부모 클래스인 Beverage 클래스에 음료의 잔 수를 반환하는 새로운 메소드를 추가해야하는 상황이라고 가정하자. 이를 위해 새로운 변수를 추가하게 된다. 만약, 잔수에 따른 할인율이 다르다면 Coffee클래스와 CoffeToastSet클래스도 변경되어야 할 것이다. 


```java
public class Beverage {

    //중복된 부분 생략
    private int count;

    public int getCount() {
        return count;
    }
}
```


<br>

**조합이란**

상속은 is-a 관계라면 조합은 has-a 관계다. 객체가 변경되더라도 영향을 최소화할 수 있기 때문에 변경에 안정적이며 구현을 효과적으로 캡슐화할 수 있다. 상속은 클래스를 통해 강하게 결합되지만 합성은 느슨하게 결합되므로 설계가 유연해진다.


 ```java
public class Job {

    private int salary;

    public int getSalary() {
        return salary;
    }

    public void setSalary(int salary) {
        this.salary = salary;
    }
}


public class Person {

    private Job job;

    public Person() {
        this.job = new Job();
        job.setSalary(1000);
    }

    public int getSalary() {
        return job.getSalary();
    }
}


public class Test {

    public static void main(String[] args) {
        Person person = new Person();
        int salary = person.getSalary();

        System.out.println("Salary : " + salary);
    }
}


```
Person 객체를 사용하여 급여(salary)를 받는다. 위 코드는 Job 객체의 변경에 영향을 받지 않는다. 

<br>

정리

상속은 컴파일 시점에 부모와 자식 클래스가 강하게 결합되는 반면, 조합은 느슨한 결합으로 코드 재사용을 위해서는 조합을 사용하는 것이 설계에 있어서 유연하다.


<br>

참고 <br>

[https://www.journaldev.com/1325/composition-in-java-example](https://www.journaldev.com/1325/composition-in-java-example)


