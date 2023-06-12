---
layout: single
title: "ConcurrentHashMap의 내부는 어떻게 구현되어 있을까?"
excerpt: ""
---

## ConcurrentHashMap이란?

`ConcurrentHashMap`은 Java 1.5 버전에서 `HashTable`의 대안으로 나온 Collection이다. `ConcurrentHashMap`이 나오기전까지는 `HashTable` 또는 `synchronized map`을 사용했다. `HashTable`과 `synchronized map`의 경우 모든 메소드에 **synchronized**가 붙어 있어 lock을 걸기 때문에 thread-safe 하지만, 성능 오버헤드를 발생시킨다. 이유는 하나의 thread가 lock을 유지하는 동안
다른 모든 thread는 해당 Map Collection을 이용할 수 없기 때문이다.

그래서 일부에만 lock을 걸어 운용하는 `ConcurrentHashMap`이 등장했고 thread-safe한 다른 Map Collection보다 성능이 좋다. 그렇다면 `ConcurrentHashMap`은 어떻게 구현되어 있을까?

## ConcurrentHashMap 동작 원리

![img](/assets/images/HashMap.png)

###
