---
layout: single
title: "ConcurrentHashMap의 내부는 어떻게 구현되어 있을까?"
excerpt: ""
---

## ConcurrentHashMap이란?

**ConcurrentHashMap**은 Java 1.5에서 도입된 Map 인터페이스의 구현체로 멀티쓰레드 환경에서 안전하게 사용할 수 있다. 이전에는 HashTable이나 Collections.synchronizedMap()을 사용해 쓰레드 안전성을 확보했지만 모든 메소드에 **synchronized**가 붙어 있어 lock을 걸기 때문에 성능 저하를 일으켰다. 이유는 하나의 쓰레드가 lock을 유지하는 동안 다른 모든 쓰레드는 해당 Map Collection에 접근할 수 없기 때문이다.

그래서 일부에만 lock을 걸어 운용하는 ConcurrentHashMap이 등장했고 thread-safe한 다른 Map Collection보다 성능이 좋다. 그렇다면 ConcurrentHashMap은 어떻게 구현되어 있을까?

### ConcurrentHashMap 생성자

`ConcurrentHashMap`의 default 생성자를 보면 `initial table size`는 **16**으로 이루어져 있는데 이는 **bucket의 수가 16이고 16개의 thread가 동시 쓰기를 할 수 있다**는 것을 의미한다.

![img8](/assets/images/ConcurrentHashmap2.png)

나머지 생성자의 파라미터는 3가지가 있다.

- **initialCapacity** : 해시 맵의 초기 용량을 결정한다.
- **loadFactor** : 해시 테이블의 크기를 설정하기 위한 용도로 0.75의 값을 가진다. 테이블의 크기가 75%에 도달할 때 버킷의 수를 동적으로 늘리기 시작한다.
- **concurrencyLevel** : 동시에 업데이트를 수행하는 예상 thread의 수

## ConcurrentHashMap 동작 원리

### HashMap vs ConcurrentHashMap

`HashMap`은 동기화 관련한 코드가 없기 때문에 multi-thread 환경에서 사용한다면 다음과 같이 전체에 lock을 걸어야 한다.

![img](/assets/images/HashMap.png)

하지만 `ConcurrentHashMap`은 맵 전체가 아니라 각각의 bucket 별로 동기화를 진행하여 다른 bucket에 속해 있을 경우, 별도의 lock이 없다.

![img2](/assets/images/ConcurrentHashmap.png)

### ConcurrentHashMap의 put() 메소드

Map에 원소를 넣기 위해 put(key, value)를 호출하면 `ConcurrentHashMap` 내부적으로 `putVal(key, value, onlyIfAbsent)` 메소드가 호출되고 이 메소드에서 세부적인 작업이 이루어진다.

![img3](/assets/images/ConcurrentHashmap_put()1.png)
![img4](/assets/images/ConcurrentHashmap_put()2.png)

이제 `putVal()` 메소드를 살펴보자.

**1. 빈 bucket에 Node를 삽입하는 경우 `Compare and Swap(CAS)`을 이용하여 lock을 사용하지 않고 새로운 노드를 bucket에 삽입한다.**

Node는 bucket 안에 저장될 Map 데이터로 필드값으로 key와 value를 가지고 있다. 

![img4](/assets/images/ConcurrentHashmap_put()3.png)

- table은 ConcurrentHashMap에서 내부적으로 관리하는 Node 객체를 관리하는 가변 배열로, table 배열이 비어있으면 initTable() 메소드를 통해 초기화한다.
- 새로운 Node를 삽입하기 위해, tabAt()을 통해 해당 bucket을 가져오고 bucket == null로 비어있는지 확인한다.
- bucket이 비어있는 경우 casTabAt() 메소드를 사용하여 Compare-And-Swap(CAS) 연산을 통해 새로운 Node를 bucket에 넣는다.

**2. 이미 bucket에 Node가 존재하는 경우 `synchronized`를 이용해 하나의 thread만 접근할 수 있도록 제어한다. 서로 다른 thread가 같은 bucket에 접근할 때만 해당 block이 잠기게 된다.**

![img5](/assets/images/ConcurrentHashmap.put()4.png)
![img6](/assets/images/ConcurrentHashmap.put()5.png)

- 같은 key값이 들어올 경우 기존 Node를 새로운 Node로 교체한다. 
- 해시 충돌이 발생하면 Separate Chaining 방식으로 Node를 추가하거나, 특정 조건에 따라 Tree로 변환한다.
  - bucket의 크기가 TREEIFY_THRESHOLD = 8 값보다 큰 경우 Linked List 대신 Tree로 바뀐다.
  - bucket의 크기가 UNTREEIFY_THRESHOLD = 6 보다 작아지면 Tree 대신 다시 Linked List로 바뀐다.

**CAS 알고리**즘은 현재 thread가 가지고 있는 기존값과 메모리가 가지고 있는 값을 비교해 같으면 변경할 값을 메모리에 반영하고, true를 반환한다. 다를 경우 변경값이 반영되지 않고 false를 반환한 다음 재시도를 하는 방식으로 동작한다.

### get()

`ConcurrentHashMap`에서의 get() 메소드를 살펴보면, synchroized 키워드가 없어도 최신의 value 값을 return할 수 있다.

![img7](/assets/images/ConcurrentHashmap.get().png)

<br>

### 정리

- ConcurrentHashMap은 각 bucket에 lock을 거는 방식을 사용한다.
- 빈 bucket에 Node를 삽입하는 경우 lock을 사용하지 않고 `Compare and Swap`만을 이용하여 새로운 Node를 해시 bucket에 삽입하여 원자성을 보장한다.
- 그 외의 업데이트(삽입, 삭제 및 교체)는 lock을 이용하지만 각 bucket의 첫 번째 Node를 기준으로 부분적으로 lock을 획득하여 업데이트 한다.

<br>

참고

[https://www.baeldung.com/java-synchronizedmap-vs-concurrenthashmap](https://www.baeldung.com/java-synchronizedmap-vs-concurrenthashmap)

[https://pplenty.tistory.com/17](https://pplenty.tistory.com/17)
