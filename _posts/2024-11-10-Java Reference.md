---
layout: single
title: Java의 Reference와 GC
excerpt: ""
---

Java의 가비지 컬렉터(Garbage Collector)는 다양한 종류가 있지만 다음과 같은 작업을 공통적으로 수행합니다.

- Heap 내의 객체 중에서 더 이상 필요 없게 된 객체를 찾아낸다.
- 찾아낸 객체를 처리하여 Heap의 메모리를 회수한다.

초기 Java는 가비지 컬렉션(GC) 작업이 애플리케이션 코드와 독립적으로 수행하도록 설계되었습니다. 하지만 객체를 다양한 방법으로 처리하기 위해 JDK 1.2부터 `java.lang.ref` 패키지가 도입되어 사용자 코드와 GC와의 제한적인 상호작용을 할 수 있게 하고 있습니다.

Java의 기본 참조 유형인 `strong reference` 외에도 `java.lang.ref` 패키지는 `soft`, `weak`, `phantom` 3가지의 새로운 참조 방식을 각각의 `Reference` 클래스로 제공합니다. 이 3가지 `Reference` 클래스를 사용하면 GC에 일정 부분 관여할 수 있습니다.

## GC의 Reachability

GC는 객체가 가비지인지 판별하기 위해 `reachability`라는 개념을 사용합니다. 어떤 객체에 유효한 참조가 있다면 `reachable`, 없으면 `unreachable`로 구별하고 `unreachable`한 객체를 가비지로 간주 및 GC를 수행합니다. 하나의 객체는 여러 다른 객체를 참조하고, 참조된 다른 객체들도 또 다른 객체들을 참조할 수 있으므로 객체는 참조 사슬을 이룹니다. 여기서 Root Space은 항상 유효한 최초의 참조로, Root Space에서 시작해 객체가 참조 가능한지 여부를 검사합니다.

JVM의 Runtime Data Area의 구조를 보면 다음과 같습니다.

![img](/assets/images/Reference.png)

Runtime Data Area는 쓰레드가 차지하는 영역과 객체를 생성 및 저장하는 Heap, 클래스 정보를 저장하는 Method Area로 나눌 수 있습니다. 위 구조에서 참조는 화살표로 표시되어 있습니다.

Heap에있는 객체들에 대한 참조 방식은 다음 네 가지 종류로 나눌 수 있습니다.

- Heap 내의 다른 객체에 의한 참조
- Java 스택의 참조(Java 메소드 실행 시 사용하는 로컬 변수나 파라미터들에 의한 참조)
- 네이티브 스택의 참조
- Method Area의 정적 변수에 의한 참조

이것들 중 Heap 내의 다른 객체에 의한 참조를 제외한 나머지 3개가 `reachability` 여부를 결정하는 기준이 됩니다. 

![img](/assets/images/Reference2.png)

Root Space로부터 시작한 참조에 속한 객체들은 `reachable` 객체이고, 이 참조 사슬과 무관한 객체들은 `unreachable` 객체로 GC 대상이 됩니다. 만약 오른쪽 아래처럼 `reachable` 객체를 참조하더라도, 다른 `reachable` 객체가 이 객체를 참조하지 않는다면 `unreachable` 객체가 됩니다.

위 그림에서의 참조는 일반적인 참조로, `strong reference`라고 부릅니다.

## Soft, Weak, Phantom Reference

`java.lang.ref` 패키지는 `soft reference`, `weak reference`, `phantom reference`를 클래스 형태로 제공하여 `java.lang.ref.WeakReference` 클래스는 객체를 캡슐화한 WeakReference 객체를 생성합니다. 이렇게 생성된 객체는 다른 객체와 달리 GC가 특별하게 처리할 수 있게 합니다.

아래는 WeakReference 클래스가 객체를 생성하는 예시입니다.

~~~java
WeakReference<Sample> wr = new WeakReference<Sample>(new Sample());  
Sample ex = wr.get();  
ex = null; 
~~~

위 코드의 동작 방식은 다음과 같습니다.

`WeakReference` 객체는 `new Sample()`로 생성된 `Sample` 객체를 캡슐화한 객체입니다. 참조된 `Sample` 객체는 `wr.get()`을 통해 다른 참조를 통해 연결되고 이때 `WeakReference` 객체의 참조와 `ex` 참조, 두 개의 참조가 처음 생성한 `Sample` 객체를 가리킵니다.

![img](/assets/images/Reference3.png)

코드 마지막 줄에서 `ex = null;`이 실행되면 처음에 생성한 `Sample` 객체는 오직 `WeakReference` 내부에서만 참조되고 이 객체를 `weakly reachable` 객체라고 합니다.

![img](/assets/images/Reference4.png)

<br>

참고

[https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/ref/package-summary.html](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/ref/package-summary.html)

[https://d2.naver.com/helloworld/329631](https://d2.naver.com/helloworld/329631)

[https://tourspace.tistory.com/42](https://tourspace.tistory.com/42)

