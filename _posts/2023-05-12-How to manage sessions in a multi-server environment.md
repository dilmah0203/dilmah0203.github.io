---
layout: single
title : 다중 서버 환경에서 session 불일치 문제와 해결 방법
excerpt : ""
---

사용자가 로그인하기 위해 POST 요청을 보내면 서버는 이 요청을 받고 session id를 발급한다. 그리고 session id를 응답 객체에 실어 사용자에게 재전송하고 사용자는 브라우저의
쿠키 저장소에 session id를 저장해 상태를 유지할 수 있다.

![img](/assets/images/session.png)

위 그림처럼 단일 서버 환경에서는 session을 통한 로그인을 구현할 때 session 불일치 문제를 신경쓸 필요가 없다. 
하지만 서비스가 커짐에 따라 한 대의 서버로 운영하는 것이 불가능해질 경우 서버를 업그레이드 해야하는데 다음과 같은 두 가지 방식이 존재한다.

첫 번째 방법은 **scale-up** 방식이다. **서버 자체의 성능을 늘려** 트래픽을 견딜 수 있게 하는 방식이지만, 여전히 서버 한 대에 모든 트래픽이 집중될 가능성이 있으므로 만일 서버 장애가 생길 시 서버가 복구될 때까지 서비스를 중단해야 하는 상황이 발생할 수 있다.

두 번째 방법은 **scale-out** 방식이다. **서버를 여러대로 늘려**서 각 서버에 로드밸런싱으로 트래픽을 분산하게 한다. 그래서 서버 한 대에 장애가 생겨도 다른 서버에 영향을 주지 않는다. 그러나 이 때 데이터 정합성, **세션 불일치 문제**가 발생할 수 있다. 왜냐하면 여러 대의 서버가 각각 세션 저장소를 독립적으로 갖기 때문에 데이터 불일치가 발생하기 때문이다.

![img2](/assets/images/session2.png)

사용자는 로그인 하기 위해 서버2와 통신한다. 서버1과 서버3은 session id를 가지고 있지 않다. 만약 사용자가 서버1이나 서버3에 요청을 보낼 경우 session id로 조회가 불가능하기 때문에 상태를 유지할 수 없고 로그인이 풀리게 된다.

이러한 문제를 해결하기 위한 방법은 세 가지가 있다.

- Sticky Session
- Session Clustering
- Session Storage

## Sticky Session

![img3](/assets/images/Sticky%20Session.png)

Sticky Session은 사용자의 세션을 처음 생성한 서버가 해당 사용자의 작업을 담당하여 고정된 세션만 사용한다. 즉 `User1`이 A 서버에 처음 로그인 요청을 하여 세션을 생성하였다면, 앞으로의 모든 `User1`의 요청은 A 서버에만 보내지게 된다. 이렇게 특정 사용자의 요청을 하나의 서버로 고정시키기 위해 로드밸런서는 요청을 보낸 사용자의 ip주소나 클라이언트의 요청에 쿠키가 존재하는지 확인 후 작업이 이루어진다. 

결론적으로 Sticky Session방식으로 사용자는 세션이 유지되는 동안 하나의 서버만 사용하게 되므로 세션 불일치 문제가 발생하지 않는다.

**Sticky Session의 단점**

- **세션 정보를 잃어버릴 수 있다.**
  - 특정 서버에 장애가 발생하면 해당 서버로 요청을 보내는 사용자의 세션 정보를 잃어버릴 수 있기 때문에 해당 서버에 고정된 유저는 다시 로그인해야 하는 문제점이 있다.

- **특정 서버에 트래픽이 몰릴 수 있다.**
  - 로드밸런서는 여러 서버에 요청을 적절히 분산하여 부하가 특정 서버에 몰리지 않기 위해 사용한다. 하지만 Sticky Session으로 인해 한 서버에 트래픽이 몰리면 로드밸런서의 원래 목적을 달성할 수 없다.

## Session Clustering

클러스터링은 여러 대의 서버들이 연결되어 하나의 시스템처럼 동작하는 것을 의미하며 Session Clustering 방식은 세션 생성과 변경 등의 작업이 있을 때마다 해당 세션이 속한 모든 서버에 세션 정보를 전파하여 정합성 문제를 해결한다.

WAS에 따라서 Session Clustering 방식이 다른데, 스프링 부트의 내장 WAS인 Tomcat의 Session Clustering 방식은 다음과 같다.

![img3](/assets/images/Session%20Clustering2.png)

즉 Tomcat은 Session Clustering을 구현하는 방법으로 all-to-all 세션 복제 방식을 사용하며, 해당 방식은 하나의 세션 저장소에 변경이 발생하면 변경된 사항이 다른 모든 세션에 복제된다.

사용자의 세션이 새로 생성되거나 업데이트될 때마다 Tomcat에서 `DeltaManager`가 다른 모든 서버에 해당 세션의 정보를 복제한다. 따라서 Sticky session의 문제점인 특정 서버에만 트래픽이 집중되는 문제를 해결할 수 있게 되었다.
![img4](/assets/images/Session%20Clustering.png)

**Session Clustering의 단점**

- **많은 메모리가 필요하다.**
  - 데이터가 변경될 때마다 세션을 전파하는 작업을 수행하므로 서버의 수가 늘어날수록 많은 메모리를 필요로 한다. 그렇기 때문에 위 공식 문서 내용과 같이 4대 이상의 대규모 클러스터는 권장되지 않는다.

- **성능 저하가 발생한다.**
  - 마찬가지로 서버 수에 비례하여 트래픽이 증가하므로 성능 저하가 생길 수 있다. 

- **세션 불일치 문제가 발생할 가능성이 존재한다.**
  - 세션 데이터 복제 작업 중에 모든 서버에 세션이 전파되기까지의 시간차로 인하여 세션 불일치 문제가 발생할 가능성이 존재한다.

- **새로운 서버가 생겨나면 클러스터링을 해주어야 한다.**
  - 새로운 서버를 띄울 때 마다 기존에 존재하던 WAS에 새로운 서버를 클러스터링 해주어야 한다.

## Session Storage

Session Storage 방식은 독립된 세션 저장소를 구성하여 해당 저장소에 세션 데이터를 저장하고, 다른 서버들이 이러한 독립된 세션 저장소에서 세션 데이터를 읽어오는 방식이다.

![img5](/assets/images/Session%20Storage.png)

Session Storage가 분리되면 서버가 늘어나도 외부 저장소의 정보만 각각의 서버에 입력해 주면 데이터를 읽어올 수 있다. 그렇기 때문에 Sticky Session의 문제점인 특정 서버로 트래픽이 몰리는 문제가 발생하지 않는다. 또한 독립된 저장소에서 세션을 공유하므로 세션이 재설정되어도 세션 저장소의 데이터만 수정하면 되어 WAS 간 불필요한 네트워크 통신 과정을 진행하지 않아도 된다. 저장소로는 MySQL, OracleDB 등 RDBMS나 Redis와 Memcached와 같은 In-memory DB를 사용한다.

**Session Storage의 단점**

- **세션 저장소가 다운되면 세션 정보를 잃어버릴 수가 있다.**
  - 하나의 독립된 세션 저장소에서 세션 정보를 저장하고 있기 때문에 해당 저장소가 다운되면 모든 서버가 세션을 공유 받을 수 없게 된다. 그래서 동일한 세션 저장소를 하나 더 구성하여 복제해야 할 필요가 있다.

<br>

참고

https://tomcat.apache.org/tomcat-8.5-doc/cluster-howto.html(https://tomcat.apache.org/tomcat-8.5-doc/cluster-howto.html)

https://velog.io/@dailyzett/%EC%84%B8%EC%85%98-%EB%B6%88%EC%9D%BC%EC%B9%98-%EC%8B%9C-%ED%95%B4%EA%B2%B0-%EB%B0%A9%EB%B2%95%EB%93%A4
(https://velog.io/@dailyzett/%EC%84%B8%EC%85%98-%EB%B6%88%EC%9D%BC%EC%B9%98-%EC%8B%9C-%ED%95%B4%EA%B2%B0-%EB%B0%A9%EB%B2%95%EB%93%A4)

https://velog.io/@sileeee/%EC%84%B8%EC%85%98-%EB%B6%88%EC%9D%BC%EC%B9%98-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0%EB%B0%A9%EB%B2%95
(https://velog.io/@sileeee/%EC%84%B8%EC%85%98-%EB%B6%88%EC%9D%BC%EC%B9%98-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0%EB%B0%A9%EB%B2%95)
