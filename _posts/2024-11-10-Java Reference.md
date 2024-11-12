---
layout: single
title: Java의 Reference와 GC
excerpt: ""
---

Java의 가비지 컬렉터(Garbage Collector)는 다양한 종류가 있지만 다음과 같은 작업을 공통적으로 수행합니다.

1. Heap 내의 객체 중에서 더 이상 필요 없게 된 객체를 찾아낸다.
2. 찾아낸 객체를 처리하여 Heap의 메모리를 회수한다.

초기 Java는 가비지 컬렉션(GC) 과정이 애플리케이션 코드와 독립적으로 수행하도록 설계되었습니다. 하지만 객체를 다양한 방법으로 처리하기 위해 JDK 1.2부터 `java.lang.ref` 패키지가 도입되어 사용자 코드와 GC와의 제한적인 상호작용을 할 수 있게 하고 있습니다.

Java의 기본 참조 유형인 `strong reference` 외에도 `java.lang.ref` 패키지는 `soft`, `weak`, `phantom` 3가지의 새로운 참조 방식을 각각의 `Reference` 클래스로 제공합니다. 이 3가지 `Reference` 클래스를 사용함으로써 GC에 일정 부분 관여할 수 있습니다.

<br>

## GC의 Reachability

GC는 객체가 가비지인지 판별하기 위해 `reachability`라는 개념을 사용합니다. 어떤 객체에 유효한 참조가 있다면 `reachable`, 없으면 `unreachable`로 구별하고 `unreachable`한 객체를 가비지로 간주 및 GC를 수행합니다. 하나의 객체는 여러 다른 객체를 참조하고, 참조된 다른 객체들도 또 다른 객체들을 참조할 수 있으므로 객체는 참조 사슬을 이룹니다. Root Space은 항상 유효한 최초의 참조로, Root Space에서 시작해 객체가 참조 가능한지 여부를 검사합니다.

JVM의 Runtime Data Area의 구조를 보면 다음과 같습니다.

![img](/assets/images/Reference.png)

Runtime Data Area는 쓰레드가 차지하는 영역과 객체를 생성 및 저장하는 Heap, 클래스 정보를 저장하는 Method Area로 크게 세 부분으로 나눌 수 있습니다. 위 구조에서 참조는 화살표로 표시되어 있습니다.

Heap의 객체 참조 방식은 다음 네 가지 종류로 나눌 수 있습니다.

- Heap 내의 다른 객체에 의한 참조
- Java 스택의 참조(Java 메소드 실행 시 사용하는 로컬 변수나 파라미터들에 의한 참조)
- 네이티브 스택의 참조
- Method Area의 정적 변수에 의한 참조

이것들 중 Heap 내의 다른 객체에 의한 참조를 제외한 나머지 3개가 Root Space로 `reachability` 여부를 결정하는 기준이 됩니다. 

![img](/assets/images/Reference2.png)

Root Space로부터 시작한 참조에 속한 객체들은 `reachable` 객체이고, 이 참조 사슬과 무관한 객체들은 `unreachable` 객체로 GC 대상이 됩니다. 만약 오른쪽 아래처럼 `reachable` 객체를 참조하더라도, 다른 `reachable` 객체가 이 객체를 참조하지 않는다면 `unreachable` 객체가 됩니다.

위 그림에서의 참조는 일반적인 참조로, `strong reference`라고 부릅니다.

## Soft, Weak, Phantom Reference


<br>

참고

[https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/ref/package-summary.html](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/ref/package-summary.html)

[https://d2.naver.com/helloworld/329631](https://d2.naver.com/helloworld/329631)


