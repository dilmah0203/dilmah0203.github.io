---
layout: single
title: "분산락을 사용하여 동시성 문제 해결하기"
excerpt: ""
---

## 문제 상황

제가 지금 진행중인 프로젝트는 동시성 문제를 가지고 있습니다.

- 두 명의 배달 기사가 동시에 배달 요청된 주문을 수락합니다.
- 주문 상태가 DELIVERY_REQUESTED인 상태에서 두 배달 기사가 동시에 접근하면, 중복 배차 문제가 발생할 수 있습니다.
  
<br>

![img](/assets/images/Concurrent.png)

## 문제 해결 방법

- **낙관적 락**
    - 버전 정보를 이용하여 데이터를 읽을 때는 락을 걸지 않고, 데이터를 업데이트할 때만 충돌을 감지하는 방식입니다. 낙관적 락은 데이터 충돌이 발생할 경우 예외 처리를 직접 구현해야 하기 때문에 추가적인 코드 작성이 필요합니다. 만약 동시에 접근하는 트랜잭션이 많아져 충돌이 자주 발생하면 재시도 횟수에 따라 DB I/O가 발생해 부하가 증가합니다.
- **비관적 락**
    - 데이터를 읽을 때부터 해당 데이터에 대한 락을 걸어 다른 트랜잭션이 해당 데이터를 변경할 수 없게 합니다. 하지만 이는 성능 저하를 유발할 수 있으며, 락이 해제될 때까지 다른 트랜잭션은 대기해야 합니다.
 
낙관적 락과 비관적 락을 비교한 결과, 라이더가 동시에 하나 이상의 배달을 수락하지 못하도록 보장하면서 데이터의 정합성을 유지하려면 비관적 락을 사용하는 것이 더 적합하다고
판단하였습니다. 현재는 단일 서버 환경이지만, 확장 가능성을 고려하여 분산 서버 환경에서도 데이터 일관성을 유지할 수 있도록 Redis를 활용한 분산 락을 구현하기로 하였습니다. 

- **분산 락**
    - 저희가 선택한 방식은 Redis를 이용한 분산 락입니다. Lettuce로 분산 락을 사용하기 위해서는 `setnx`, `setex` 등을 이용해 분산 락을 직접 구현해야 합니다. 개발자가 직접 retry, timeout과 같은 기능을 구현해주어야 하는 번거로움이 있습니다. 이에 비해 Redission은 별도의 Lock interface(RLock)를 지원합니다.
    - Lettuce는 분산락 구현 시 `setnx`, `setex`과 같은 명령어를 이용해 지속적으로 Redis에게 락이 해제되었는지 요청을 보내는 스핀락 방식으로 동작하기 때문에 요청이 많을수록 Redis의 부하는 커지게 됩니다.
    - 이에 비해 `Redisson`은 Pub/Sub 방식을 이용하여 락이 해제되면 락을 기다리던 쓰레드들은 락이 해제되었다는 신호를 받고 락 획득을 시도합니다.

~~~java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DistributedLock {
    String key();
    TimeUnit timeUnit() default TimeUnit.SECONDS;
    long waitTime() default 5L;
    long leaseTime() default 3L;
}
~~~

`@DistributedLock`은 특정 메소드에 분산 락을 적용하기 위해 정의된 커스텀 어노테이션으로 이 어노테이션을 통해 락의 키, 시간 단위, 대기 시간, 유지 시간등을 지정합니다.

~~~java
@Component
public class AopTransactionExecutor {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public Object proceed(final ProceedingJoinPoint joinPoint) throws Throwable {
        return joinPoint.proceed();
    }
}
~~~

`Redisson`을 사용해 Redis 분산 락을 관리하고, `AopTransactionExecutor` 클래스를 사용해 트랜잭션을 처리하고 메소드를 실행합니다. 

~~~java
@Aspect
@Component
@RequiredArgsConstructor
public class DistributedLockAop {

    private final RedissonClient redissonClient;
    private final AopTransactionExecutor aopTransactionExecutor;
    
    private static final String REDISSON_LOCK_PREFIX = "LOCK:";

    @Around("@annotation(com.project.food_ordering_service.global.annotaion.DistributedLock)")
    public Object lock(final ProceedingJoinPoint joinPoint) throws Throwable {
        Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
        DistributedLock distributedLock = method.getAnnotation(DistributedLock.class);

        String key = buildLockKey(distributedLock, joinPoint);
        RLock rLock = redissonClient.getLock(key); // (1)

        try {
            boolean available = rLock.tryLock(distributedLock.waitTime(),
                    distributedLock.leaseTime(), distributedLock.timeUnit()); // (2)
            if (!available) {
                return false;
            }
            return aopTransactionExecutor.proceed(joinPoint); // (3)
        } catch (InterruptedException e) {
            throw new InterruptedException();
        } finally {
            rLock.unlock(); // (4)
        }
    }

    private String buildLockKey(DistributedLock distributedLock, ProceedingJoinPoint joinPoint) {
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        Object dynamicKey = DynamicValueParser.getDynamicValue(methodSignature.getParameterNames(),
                joinPoint.getArgs(), distributedLock.key());
        return REDISSON_LOCK_PREFIX + dynamicKey;
    }
}
~~~

`@DistributedLock` 애노테이션 선언 시 수행되는 aop 클래스입니다. `@DistributedLock` 애노테이션의 파라미터 값을 가져와 분산락 획득을 시도하고 애노테이션이 선언된 메소드를 실행합니다.

1. 락의 이름으로 RLock 인스턴스를 가져옵니다.
2. 정의된 waitTime까지 락 획득을 시도하고 leaseTime이 지나면 락을 해제합니다.
3. DistributedLock 애노테이션이 선언된 메소드를 별도의 트랜잭션으로 실행합니다.
4. 종료시 무조건 락을 해제합니다.

`@DistributedLock`이 선언된 메소드는 Propagation.REQUIRES_NEW 옵션을 지정해 별도의 트랜잭션으로 동작합니다. 그리고 반드시 **트랜잭션 커밋 이후 락이 해제됩니다.**

> 왜 트랜잭션 커밋 이후 락이 해제되어야 할까?

**락의 해제 시점이 트랜잭션 커밋 시점보다 빠르면 데이터 정합성이 깨질 수 있습니다.** 예를 들어, 트랜잭션이 커밋되기 전에 락이 해제되면 다른 트랜잭션이 동시에 접근할 수 있게 되어, 데이터의 정합성이 유지되지 않을 수 있습니다. 따라서 트랜잭션 커밋이 완료된 후에 락을 해제함으로써 데이터 정합성을 보장할 수 있습니다.

`lock()` 메소드를 자세히 살펴보겠습니다.

~~~java
@Around("@annotation(com.project.food_ordering_service.global.annotaion.DistributedLock)")
public Object lock(final ProceedingJoinPoint joinPoint) throws Throwable {
    Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
    DistributedLock distributedLock = method.getAnnotation(DistributedLock.class);
    
    String key = buildLockKey(distributedLock, joinPoint);
    RLock rLock = redissonClient.getLock(key);
		
    try {
        boolean available = rLock.tryLock(distributedLock.waitTime(),
                distributedLock.leaseTime(), distributedLock.timeUnit());
        if (!available) {
            return false;
        }
        return aopTransactionExecutor.proceed(joinPoint);
    } catch (InterruptedException e) {
        throw new InterruptedException();
    } finally {
        rLock.unlock();
    }
}
~~~

- @Around 애노테이션을 통해 `@DistributedLock`이 붙은 메소드에 대해 `lock() 메소드`를 적용하도록 설정합니다.
- ProceedingJoinPoint joinPoint : AOP에서 메소드 호출 대상 객체에 대한 정보를 담고 있는 객체로 `joinPoint.proceed()`가 호출되면 타겟 메소드를 실행할 수 있습니다.
- MethodSignature : 현재 호출된 메소드 정보를 가져올 수 있는 객체로 실제 메소드와 메타데이터(애노테이션)에 접근할 수 있습니다. `method.getAnnotation()`을 통해 메소드에 붙어있는 `@DistributedLock` 애노테이션을 가져와 락을 설정할 때 사용할 정보를 알 수 있습니다.
- buildLockKey() 메소드에 넘겨진 파라미터를 분석해서 Redis에 저장될 락의 키를 생성합니다. 이 키는 분산 락을 구분하는데 사용됩니다.
- Redission의 RLock은 Redis에서 분산 락을 관리하는 객체로 `getLock(key)`를 통해 Redis의 특정 키에 대한 락을 가져옵니다. 락이 존재하지 않으면 새로운 락을 만듭니다.
- tryLock() : 락을 걸 때 사용되는 메소드로 앞에서 설정했던 **시간 단위,** **대기 시간, 유지 시간 정보**를 이용합니다. waitTime동안 락이 풀리기를 기다리며, 락을 획득하면 leaseTime 동안 유지됩니다.
- 락 획득 실패시 false를 반환해 메소드 실행을 하지 않고, 락을 성공적으로 획득하면 `aopTransactionExecutor.proceed()`가 호출되어 원래의 메소드를 실행합니다. 이 메소드는 @Transactional(propagation = Propagation.REQUIRES_NEW)으로 설정된 별도의 트랜잭션에서 실행됩니다.
- finally 블록에서 락을 해제합니다. 메소드 실행이 완료되면 무조건 락이 해제되도록 보장합니다. 이는 락을 획득한 후 실행 중에 예외가 발생하더라도 락이 적절히 해제되도록 하기 위함입니다.

아래는 배달 할당 과정에서 분산락이 적용된 코드입니다.

~~~java
@Service
@RequiredArgsConstructor
public class DeliveryService {

    private final DeliveryRepository deliveryRepository;
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;

    @DistributedLock(key = "#orderId")
    public Delivery assignDelivery(Long orderId, Long riderId) {
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new CustomException(ErrorInformation.ORDER_NOT_FOUND));
        ...
    }
}  
~~~

`@DistributedLock` 애노테이션이 달린 `assignDelivery()` 메소드가 호출되면 @Around 어드바이스가 실행되어 분산 락을 설정합니다. REDISSON_LOCK_PREFIX와 `@DistributedLock` 애노테이션에서 지정한 key 및 메소드의 인자값을 기반으로 락의 키를 생성합니다. 이 키는 Redis에 저장된 락을 구분하는 데 사용됩니다.

- RedissonClient를 사용하여 Redis에서 RLock 객체를 가져옵니다.
- `rLock.tryLock()` 메소드를 호출하여 락을 시도합니다.
- 락을 성공적으로 획득하면 `aopTransactionExecutor.proceed(joinPoint)`를 호출하여, 메소드를 실행합니다.
- AopTransactionExecutor는 @Transactional(propagation = Propagation.REQUIRES_NEW)로 설정되어 있어, 새로운 트랜잭션을 시작하고 메소드를 실행합니다.
- `assignDelivery()` 메소드가 실행된 후, finally 블록에서 `rLock.unlock()`을 호출하여 락을 해제합니다. 이는 메소드 실행이 완료된 후 락이 반드시 해제되도록 보장합니다. 예외가 발생하더라도 락은 적절히 해제됩니다.
