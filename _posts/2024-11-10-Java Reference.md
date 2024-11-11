---
layout: single
title: Java의 Reference와 GC
excerpt: ""
---

Java의 가비지 컬렉터(Garbage Collector)는 다양한 종류가 있지만 다음과 같은 작업을 공통적으로 수행합니다.

1. Heap 내의 객체 중에서 더 이상 필요 없게 된 객체를 찾아낸다.
2. 찾아낸 객체를 처리하여 Heap의 메모리를 회수한다.

초기 Java는 가비지 컬렉션(GC) 과정이 애플리케이션 코드와 독립적으로 수행하도록 설계되었습니다. 하지만 객체를 다양한 방법으로 처리하기 위해 JDK 1.2부터 `java.lang.ref` 패키지가 도입되어 사용자 코드와 GC와의 제한적인 상호작용을 할 수 있게 하고 있습니다.

Java의 기본 참조 유형인 strong reference 외에도 `java.lang.ref` 패키지는 `soft`, `weak`, `phantom` 3가지의 새로운 참조 방식을 각각의 Reference 클래스로 제공합니다. 이 3가지 Reference 클래스를 사용함으로써 GC에 일정 부분 관여할 수 있습니다.
