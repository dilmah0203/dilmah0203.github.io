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

### HashMap vs ConcurrentHashMap

`HashMap`은 동기화 관련한 코드가 없기 때문에 multi-thread 환경에서 사용한다면 다음과 같이 전체에 lock을 걸어야 한다.

![img](/assets/images/HashMap.png)

하지만 `ConcurrentHashMap`은 각각의 bucket 별로 동기화를 진행하여 다른 bucket에 속해 있을 경우, 별도의 lock이 없다.

![img2](/assets/images/ConcurrentHashmap.png)

### put()

Map에 원소를 넣는 put(key, value)를 호출하면 `ConcurrentHashMap` 내부적으로 `putVal(key, value, onlyIfAbsent)`로 연결된다.

![img3](/assets/images/ConcurrentHashmap_put()1.png)
![img4](/assets/images/ConcurrentHashmap_put()2.png)

이제 `putVal()` 메소드를 살펴보자.

**1. 빈 bucket에 Node를 삽입하는 경우 `Compare and Swap`을 이용하여 lock을 사용하지 않고 새로운 노드를 bucket에 삽입한다.**

Node는 버킷 안에 저장될 Map 데이터로 필드값으로 key와 value를 가지고 있다. key는 final로 선언되어 한 번 초기화되면 수정이 불가능하다.

![img4](/assets/images/ConcurrentHashmap_put()3.png)

1. table은 ConcurrentHashMap에서 내부적으로 관리하는 Node의 가변 배열로, 무한 loop를 통해 삽입될 bucket을 확인한다.
2. 새로운 Node를 삽입하기 위해, `tabAt()`을 통해 해당 bucket을 가져오고 `bucket == null`로 비어있는지 확인한다.
3. bucket이 비어있는 경우 `casTabAt()`을 통해 Node를 담고 있는 volatile 변수에 접근하고 Node와 기대값 `null`을 비교하여 같으면 Node를 생성해 넣고, 아니면 1번으로 돌아가 재시도 한다.

**2. 이미 bucket에 Node가 존재하는 경우 `synchronized`를 이용해 하나의 thread만 접근할 수 있도록 제어한다. 서로 다른 thread가 같은 bucket에 접근할 때만 해당 block이 잠기게 된다.**

![img5](/assets/images/ConcurrentHashmap.put()4.png)
![img6](/assets/images/ConcurrentHashmap.put()5.png)

1. f는 비어있지 않은 Node<K,V> 타입의 bucket을 의미하고 이것을 통해 동기화 한다.
2. 같은 key값이 들어올 경우 새로운 Node로 교체한다. 
3. 해시 충돌인 경우에는 Separate Chaining에 추가하거나, Tree에 추가한다.
4. bucket의 수(binCount)에 따라 Linked List로 운용할지 Tree로 운용할지 정한다.

3,4는 HashMap의 내부 구현과 같으며

- bucket의 크기가 TREEIFY_THRESHOLD = 8 값보다 큰 경우 Linked List 대신 Tree로 바뀐다.
- bucket의 크기가 UNTREEIFY_THRESHOLD = 6 보다 작아지면 Tree 대신 다시 Linked List로 바뀐다.


CAS 알고리즘은 현재 thread가 가지고 있는 기존값과 메모리가 가지고 있는 값을 비교해 같은 경우 변경할 값을 메모리에 반영하고 true를 반환한다. 다른 경우에는 변경값이 반영되지 않고 false를 반환한 다음 재시도를 하는 방식으로 동작한다.

### get()

`ConcurrentHashMap`에서의 get() 메소드를 살펴보면, synchroized 키워드가 없다. 즉, get()은 가장 최신의 value 값을 return한다.

![img7](/assets/images/ConcurrentHashmap.get().png)

### ConcurrentHashMap 생성자

`ConcurrentHashMap`의 default 생성자를 보면 `initial table size`는 **16**으로 이루어져 있는데 이는 **bucket의 수가 16이고 16개의 thread가 동시 쓰기를 할 수 있다**는 것을 의미한다.

![img8](/assets/images/ConcurrentHashmap2.png)

나머지 생성자의 파라미터는 3가지가 있다.

- **initialCapacity** : 초기 용량을 결정한다.
- **loadFactor** : 초기 hashTable의 크기를 설정하기 위한 용도로 0.75의 값을 가진다. 
- **concurrencyLevel** : 동시에 업데이트를 수행하는 예상 thread의 수

**정리**

`ConcurrentHashMap`은 각 bucket에 lock을 거는 방식이다. 빈 bucket에 Node를 삽입하는 경우 lock을 사용하지 않고 `Compare and Swap`만을 이용하여 새로운 Node를 해시 bucket에 삽입하여 원자성을 보장한다. 그 외의 업데이트(삽입, 삭제 및 교체)는 lock을 이용하지만 각 bucket의 첫 번째 Node를 기준으로 부분적으로 lock을 획득하여 업데이트 한다.

<br>

참고

[https://www.baeldung.com/java-synchronizedmap-vs-concurrenthashmap](https://www.baeldung.com/java-synchronizedmap-vs-concurrenthashmap)

[https://pplenty.tistory.com/17](https://pplenty.tistory.com/17)
