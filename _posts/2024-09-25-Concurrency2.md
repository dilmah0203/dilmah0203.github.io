---
layout: single
title: "분산락을 사용하여 동시성 문제 해결하기"
excerpt: ""
---

## 문제 상황

제가 지금 진행중인 프로젝트는 동시성 문제를 가지고 있었습니다.

- 두 명의 배달 기사가 동시에 배달 요청된 주문을 수락합니다.
- 주문 상태가 DELIVERY_REQUESTED인 상태에서 두 배달 기사가 동시에 접근하면, 중복 배차 문제가 발생할 수 있습니다.

![img](/assets/images/Concurrent.png)

## 문제 해결 방법

- **낙관적 락**
    - 버전 정보를 이용하여 데이터를 읽을 때는 락을 걸지 않고, 데이터를 업데이트할 때만 충돌을 감지하는 방식입니다. 동시에 접근하는 트랜잭션이 많을 때 성능에 좋지 않을 수 있습니다.
- **비관적 락**
    - 데이터를 읽을 때부터 해당 데이터에 대한 락을 걸어 다른 트랜잭션이 해당 데이터를 변경할 수 없게 합니다. 하지만 이는 성능 저하를 유발할 수 있으며, 락이 해제될 때까지 다른 트랜잭션은 대기해야 합니다.
 
라이더가 동시에 하나 이상의 배달을 받을 수 없게 보장하기 위해, 또 데이터의 정합성을 고려한다면 비관적 락을 사용하는 것이 낫겠다고 생각하였습니다.

현재 프로젝트는 외부 API 호출 시에 서버 비용 절감과 읽기 성능 향상을 위해 캐시를 사용하고 있습니다. 지금은 단일 서버 환경이지만 확장 가능성을 고려했을 때 분산 서버 환경에서 서버간 데이터 일관성 유지를 위해 Redis를 캐시 저장소로 활용하고 있습니다. 단일 서버 환경에서는 비관적 락만으로도 동시성 문제를 해결할 수 있지만 분산 시스템의 경우 분산 락을 사용할 수 있다고 보았고 Redis를 이용하여 분산 락을 구현하기로 하였습니다.

- **분산 락**
    - 저희가 선택한 방식은 Redis를 이용한 분산 락입니다. Lettuce로 분산 락을 사용하기 위해서는 `setnx`, `setex` 등을 이용해 분산 락을 직접 구현해야 합니다. 개발자가 직접 retry, timeout과 같은 기능을 구현해주어야 하는 번거로움이 있습니다. 이에 비해 Redission은 별도의 Lock interface를 지원합니다.
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

`lock() 메소드`를 자세히 살펴보겠습니다.
