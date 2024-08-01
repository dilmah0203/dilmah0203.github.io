---
layout: single
title: "**️커스텀 애노테이션과 AOP를 활용한 권한 검사 구현하기**"
excerpt: ""
---

## 리팩토링 전 코드

주문과 배달 관련 API에서 사용자 토큰을 통해 접근 권한을 확인하는 로직이 중복되는 문제가 발생하였습니다. 이외에도 주문 상태 변경과 배달 상태 변경 로직에서도 아래와 같이 사용자의 권한을 확인하는 부분이 반복되었습니다.

```java
@Transactional
    public Order createOrder(JwtAuthentication jwtAuthentication, OrderRequest orderRequest) {
        if (!jwtAuthentication.getRole().equals(Role.CLIENT)) {
            throw new AccessDeniedException("고객만 주문할 수 있습니다.");
        }
    ...
}
```

```java
@Transactional
    public Delivery assignDelivery(Long orderId, Long riderId, JwtAuthentication jwtAuthentication) {
        if (!jwtAuthentication.getRole().equals(Role.RIDER)) {
            throw new AccessDeniedException("배달원만 배달을 할 수 있습니다.");
        }
    ...
}
```

예를 들어, 고객이 주문을 완료한 후 사장님이 배달을 수락하거나 배달 상태를 변경할 때에는 사장님 권한으로 로그인이 되어있어야 합니다. 이러한 불필요한 코드 반복을 줄이고 핵심 비즈니스 로직과 권한 확인 로직을 분리하여 관리하는 것이 더 좋은 방식이라고 생각했습니다.

## AOP란 무엇일까?

