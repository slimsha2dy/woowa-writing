# Virtual Thread 받아라~
**Virtual Thread**는 Java 진영의 **경량 스레드 모델**입니다.

기존의 Java의 스레드 모델이 OS 스레드와 1:1로 매핑되던 것과 달리, JVM 내부 스케줄링을 통해 가상 스레드를 OS 스레드에 매핑하는 방식이기 때문에 대규모의 스레드를 동시에 사용할 수 있다는 장점을 갖고 있습니다.

본 글에서는 JDK 21부터 새롭게 도입된 Virtual Thread의 특징, 구조와 사용법, 그리고 주의사항에 대해 알아보도록 하겠습니다.

## 기존 Java Thread와 웹 어플리케이션

기존 Java의 스레드는 OS 스레드를 Wrapping한 객체입니다. 즉, Kernel-level Thread와 User-level Thread가 1:1로 매핑된 형태로, JVM에서 새로운 스레드를 생성하면 시스템 콜을 통해 OS의 스레드를 할당받고 매핑하여 실행하게 됩니다.

![버춸1](https://github.com/user-attachments/assets/8b675a49-92f8-4722-8c92-c3ed92eb0b39)

위의 그림에서 확인할 수 있듯, 모든 **플랫폼 스레드**는 **OS 스레드**와 **1:1 관계**를 맺고 있습니다. `new Thread` 를 통해 플랫폼 스레드를 생성한 후 `thread.start()`를 실행하게 되면 JNI(Java Native Interface)를 통해 직접 OS로 스레드 생성 요청을 하여 직접적으로 1:1 관계가 맺어지고, 이 스레드는 OS의 스케줄러가 관리하게 됩니다.

또한 Spring MVC 웹 어플리케이션에서는 사용자의 요청을 처리할 때 **Thread per request** 방식을 사용해 한 요청 당 한 개의 스레드를 할당해서 처리를 하게 됩니다. 즉, **요청의 처리량**을 높이기 위해서는 사용할 수 있는 **스레드의 개수를 늘려야 할 것**입니다. 하지만 OS 스레드의 생성과 유지 비용은 비싸고, 최대 개수도 OS 스레드에 종속적이므로 기존 플랫폼 스레드 모델로는 동시에 처리할 수 있는 처리량에 한계가 있습니다.

이 때 **스레드 풀**이라는 개념이 등장하게 됩니다. 매 요청마다 스레드를 생성하는 것이 아닌, 스레드 풀에 미리 스레드들을 생성해 놓은 후 요청이 들어오면 이 스레드 풀의 스레드를 사용하고, 요청이 끝나도 매번 스레드를 파괴하지 않고 다음 요청을 위해 다시 스레드 풀로 이동시킵니다. 이러한 방법은 스레드의 생성 비용을 절감할 수는 있지만 비싼 OS의 컨텍스트 스위칭 및 스케줄링 비용과 OS 스레드 개수의 한계로 인한 처리량의 한계는 해결하기 어렵습니다.

기존 플랫폼 스레드 모델의 한계를 정리하면 아래와 같습니다.

1. **스레드 생성 비용이 비싸다**
2. **컨텍스트 스위칭 및 스케줄링 비용도 비싸다**
3. **처리량의 한계**

## Virtual Thread란?

Virtual Thread는 이러한 기존 스레드 모델의 단점을 보완하기 위해 등장한 **경량 스레드** 모델입니다.

여기서 **경량 스레드**란 기존의 스레드의 실행 단위를 더 작게 나눠서 여러 비용을 낮춘 개념으로, **스레드가 경량 프로세스**라고도 불리는 것을 생각한다면 더 쉽게 이해할 수 있을 것 같습니다. Go의 **goroutine**과 코틀린의 **coroutine**이 이 경량 스레드의 일종이라고 볼 수 있습니다. 이러한 경량 스레드는 기존 스레드에 비해 메모리 사용량이 적고, 생성, 컨텍스트 스위칭 및 스케줄링 비용이 싸다는 특징을 갖고 있습니다.

또한 Virtual Thread는 실제 OS 스레드와 1:1로 매핑되는 것이 아니기 때문에 OS 스레드 개수에 종속적이지 않고 JVM Heap 크기가 허용하는 한 제한 없이 생성할 수 있으며, Nonblocking I/O를 지원하기 때문에 기존의 스레드보다 훨씬 높은 수준의 처리량을 기대할 수 있습니다.

## Virtual Thread의 장점

Java의 Virtual Thread 또한 앞서 설명드린 경량 스레드의 장점을 모두 갖고 있습니다.
이에 더해, Virtual Thread는 기존의 Thread를 상속하고 있다는 장점을 갖고 있습니다. 이에 따라 기존 코드에서 Thread를 사용하던 부분에 바로 Virtual Thread를 적용할 수 있게 됩니다.
따라서 Virtual Thread의 장점을 정리하자면 다음과 같습니다.

- **메모리 사용량**
- **스레드 생성 비용**
- **컨텍스트 스위칭 및 스케줄링 비용**
- **최대 개수**
- **Nonblocking I/O 지원**
- **기존 Thread 상속**

이 중 위의 4가지 장점을 종합하면 **많이 생성하고 관리하기 용이하다** 라고 정리할 수 있고, Nonblocking I/O를 지원한다는 것은 결국 **높은 처리량을 기대할 수 있다**고 생각할 수 있습니다.

즉, 이를 전부 종합하면 Virtual Thread는 **대규모의 스레드를 동시에 처리할 수 있다**는 말이 됩니다. 또한 Thread per Request 모델인 Spring MVC에서는 **대규모의 요청을 동시에 처리할 수 있다**는 말과 같을 것이고 이 점이 오늘 소개할 Virtual Thread의 핵심입니다

### 메모리 사용량과 비용

기존 스레드가 최대 2MB의 크기를 갖는 반면 Virtual Thread는 최대 10KB의 크기를 갖고 있습니다. 즉, Virtual Thread는 기존의 스레드에 비해 메모리 사용량이 1% 정도밖에 되지 않으므로 컨텍스트 스위칭에 필요한 데이터 이동량이 적고 이에 따라 컨텍스트 스위칭 비용이 더 적을 것을 예측할 수 있습니다.

또한 앞서 설명했듯, 기존의 스레드는 생성을 위해 시스템 콜이 필요하지만 Virtual Thread는 **JVM 상에서만 스레드 생성이 일어나게** 됩니다. 매번 생성을 위해 커널 영역의 호출이 필요하지 않게 된 만큼 생성 비용에 있어서도 큰 장점을 갖고 있다고 할 수 있습니다.

스케줄링 또한 OS 스케줄러에 의해 관리되던 기존 방식과 달리 JVM의 스케줄러에 의해 관리됩니다. 시스템 콜로 인한 오버헤드가 감소하여 커널 영역의 추가적인 비용이 덜 발생하게 됩니다.

Virtual Thread는 훨씬 더 적은 메모리를 사용하고 OS 스레드에 종속적이지 않기 때문에 **훨씬 많은 수의 스레드 생성**이 가능하므로 요청을 처리할 더 많은 스레드를 생성할 수 있습니다. 또한 스레드를 생성하고 유지하는 비용도 훨씬 싸기 때문에 Thread per request 모델에서 **더 많은 요청을 처리하기에 용이**하다고 할 수 있습니다.

### Nonblocking I/O 지원과 기존 Thread 상속

**Nonblocking I/O**란 I/O 작업으로 인한 Blocking 시간 동안 대기하지 않고 다른 작업을 처리하는 것을 의미합니다.

I/O가 포함된 작업의 경우 CPU 작업 시간보다 I/O Blocking으로 인한 대기 시간이 훨씬 많은 비중을 차지하게 되는데, 이러한 대기 시간이 Thread per request 모델에서는 성능 저하의 원인이 될 수 있습니다. 특히 Micro Service Architecture 구조와 같이 I/O Blocking 대기 시간이 점점 더 길어지는 오늘날의 웹 서버 환경에서는 특히 병목 지점이 될 가능성이 높습니다.

![버춸2](https://github.com/user-attachments/assets/1a2812ad-6a0f-4c62-9d5c-e38751b1475c)


위의 그림에서 확인할 수 있는 것처럼, Nonblocking I/O에서는 이러한 대기 시간 동안 다른 task를 처리하기 때문에 **보다 높은 처리량**을 기대할 수 있습니다.

기존 자바 플랫폼 스레드와 Spring MVC 모델에서는 I/O Blocking이 발생할 경우 해당 스레드는 CPU에 자원을 반납하고 대기 상태에 들어가기 때문에 Nonblocking I/O 방식을 사용할 수 없었습니다. 따라서 자바에서 Nonblocking I/O를 사용하기 위해서는 **Spring Webflux**와 같은 **리액티브 프로그래밍** 모델을 사용해야 했습니다. 하지만 이는 기존에 작성하던 코드와 형태가 달라 러닝 커브가 존재하고 동작 구조로 인해 디버깅에 불편함이 따른다는 단점을 갖고 있습니다. 또한 Reactive하게 동작하는 라이브러리 지원을 필요로 하므로 JPA와 같은 라이브러리를 사용하기 힘들고 R2DBC와 같은 라이브러리를 추가적으로 학습해야 하는 비용도 필요로 합니다.

Virtual Thread는 Nonblocking I/O를 지원하면서 기존 Thread를 상속하고 있기 때문에 기**존 코드의 구조를 유지하면서** Nonblocking I/O를 적용할 수 있습니다. 따라서 Spring Webflux와 달리 **별도의 러닝 커브 및 추가적인 구조 변경 없이** Nonblocking I/O를 적용함으로써 더 높은 처리량을 챙길 수 있습니다.

## **Virtual Thread의 특징**

Virtual Thread는 앞서 설명드린 장점을 어떻게 가질 수 있을까요? 이는 다음과 같은 주요한 특징들로 인해 가능하게 됩니다.

1. **캐리어 스레드와 가상 스레드**
2. **JVM에 의해 스케줄링**
3. **Continuation**

Vitrual Thread는 **캐리어 스레드**와 **가상 스레드**의 형태로 관리되며 **JVM이 이를 스케줄링**합니다. 이에 따라 앞서 설명드린 장점 중 메모리 사용과 여러 비용들에서 장점을 가질 수 있게 됩니다.

또한 Virtual Thread에서는 Runnable 대신 **Continuation**을 사용하기 때문에 **Nonblocking I/O를 지원**할 수 있게 되었습니다. 이에 대한 자세한 내용은 뒤에서 알아보겠습니다.

### 캐리어 스레드와 버추얼 스레드

기존 자바의 플랫폼 스레드가 OS 스레드와 1:1 관계를 갖고 있는 것과 달리, Virtual Thread에서는 각 스레드들이 OS 스레드와 **N:M**의 관계를 갖고 있습니다.

![버춸3](https://github.com/user-attachments/assets/e46264f2-1eed-4233-b3f1-57d14e1ed5eb)

실제 OS 스레드와 1:1로 매핑되는 스레드는 실제로 요청과 매핑되는 Virtual Thread가 아닌 **캐리어 스레드**입니다. 만약 처리해야 하는 **Virtual Thread**가 생겨나면 이 **캐리어 스레드에 할당되고 해제**됩니다. 즉, 기존의 플랫폼 스레드 역할은 캐리어 스레드가 하게 되며, 이 캐리어 스레드가 어떤 스레드를 처리할지 JVM에서 스케줄링하는 방식입니다.

실제로 코드를 확인해 보면 Virtual Thread는 Thread인 carrierThread를 참조하고 있습니다. 스케줄러는 `mount()`와 `unmount()` 메서드를 통해 이 carrierThread에 실제 스레드를 할당하거나 제거함으로써 Virtual Thread가 CPU에서 실행될 수 있도록 합니다.

이런 구조로 인해 새로운 스레드를 생성하거나 컨텍스트 스위칭이 발생해도 시스템 콜 없이 JVM 위에서만 작업이 이루어지게 되므로 생성 비용도 훨씬 적게 됩니다. 또한 OS 스레드와 매핑되기 위한 데이터들도 Virtual Thread가 갖고 있을 필요는 없으므로 메모리 사용량 또한 훨씬 적습니다. 또한 Virtual Thread는 OS 스레드의 개수에 종속받지 않게 되므로 기존의 스레드보다 훨씬 많이 생성할 수 있게 됩니다.

### Continuation

기존의 플랫폼 스레드 작업 단위가 Runnable인 것과 달리 Virtual Thread에서는 **Continuation**이라는 생소한 **작업 단위**를 사용하고 있습니다.

![버춸4](https://github.com/user-attachments/assets/dfd01e1e-e0bd-4d3e-abbc-3d6de94a6c31)

위의 이미지에서 확인할 수 있듯, 실제 `VirtualThread` 클래스를 확인해 보면 필드로 Continuation을 갖고 있고, 생성자의 인자로 받은 Runnable을 통해 Continuation을 생성합니다.

이 Continuation을 간단히 설명하자면 중단이 가능하고 중단 지점부터 재개가 가능한 작업 흐름입니다. 그림을 통해 알아보도록 하겠습니다.

![버춸5](https://github.com/user-attachments/assets/b203492a-2cdd-4a7e-a2b2-e2572b42a538)

기존의 Function은 호출자가 한 번 호출하면 함수가 끝날 때까지 쭉 진행한 후, 끝나고 나면 다시 호출자로 실행 흐름이 넘어오게 됩니다.
하지만 Continuation은 호출자가 함수를 호출한 후, yield를 통해 언제든지 다시 실행 흐름을 넘겨받을 수 있고, 다시 그 중단점부터 함수를 재개할 수 있습니다.

Virtual Thread는 **작업 단위**로 이 **Continuation**을 갖고 있으며, `runContinuation()` 을 실행함으로써 Continuation을 실행할 수 있고, `yieldContinuation()` 를 실행함으로써 Continuation을 중단할 수 있습니다.
JVM의 스케줄러는 이러한 호출자로서 여러 Virtual Thread들을 호출하고 중단하고 실행하면서 스케줄링을 진행할 수 있게 됩니다.
이러한 Continuation을 통해 JVM에서는 실제 OS 스레드의 block 없이 **적은 비용으로 여러 Virtual Thread들을** **스케줄링**할 수 있습니다.

![버춸6](https://github.com/user-attachments/assets/11bfe45b-ff9a-4c47-8cb5-0d3dec056f72)

실제로 block이 발생하는 sleep 메서드를 확인해 보면 중간에 현재 스레드가 Virtual Thread인지를 확인하는 분기 처리를 하고 있습니다. 만약 Virtual Thread가 아니라면 `sleep0` 라는 native 함수를 호출해 block 상태가 됩니다. 하지만 Vitual Thread인 경우 호출하는 메서드를 따라가 보면 native 함수를 호출하는 대신 `yieldContinuation` 을 호출하고 있습니다. 즉, 이 경우에는 실제로 block하는 대신, 현재 Continuation만 yield한 후, 다른 Virtual Thread를 실행할 수 있게 됩니다.

![버춸7](https://github.com/user-attachments/assets/8749429a-0e40-4fcb-b003-9ff9aea85ff2)

sleep 메서드의 흐름을 그림으로 나타내면 위와 같습니다. 기존 Thread인 경우에는 위에서 언급했던 Blocking I/O처럼 동작하는 것을 확인할 수 있고, Virtual Thread인 경우에는 Continuation을 yield한 후 다른 Virtual Thread를 실행하는 식으로 Nonblocking I/O 방식을 지원하는 것을 확인할 수 있습니다.

## **Virtual Thread 사용법**

우선 Virtual Thread를 사용하기 위해서는 java 21 버전을 사용해야 하므로 gradle과 IDE의 설정을 21 버전에 맞춰주어야 합니다.

그리고 다음과 같이 여러 api를 이용해 Virtual Thread를 생성하고 사용할 수 있습니다.

```java
// Thread.ofVirtual
Thread thread1 = Thread.ofVirtual().start(() -> { ... });

Thread thread2 = Thread.ofVirtual().unstarted(() -> { ... });
thread2.start();

// Thread.Builder
Thread.Builder threadBuilder = Thread.ofVirtual().name("Name");
Thread thread3 = threadBuilder.start(() -> { ... });

// ExecutorService
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
executor.submit(() -> { ... });
```

하지만 웹 프레임워크 기반으로 개발을 하는 환경에서 직접적으로 스레드를 생성하는 일이 잦지는 않습니다.

아래와 같은 설정을 `application.yml`에 추가해 주면 Spring Boot 3.2 환경에서 쉽게 Virtual Thread를 사용할 수 있습니다.

```yaml
spring:
    threads:
        virtual:
            enabled: true
```

만약 Spring Boot 3.2 이상이 아니라면, `TomcatProtocolHandlerCustomizer` 를 통해 직접 Virtual Thread를 사용하도록 설정할 수 있습니다.

## 주의할 점

### pinning

코드 내부에서 `synchronized` 키워드를 사용하게 되면 실제 캐리어 스레드가 block 당하게 됩니다. 이로 인해 Virtual Thread의 장점을 활용하지 못하는 상황이 발생하게 되는데 이를 pinning이라고 합니다.

Java util에서 제공하는 `ReentrantLock`을 사용하면 순차적 접근을 보장하면서 캐리어 스레드가 block 당하는 상황을 보장할 수 있습니다. 또한 pinning이 발생하는지 확인할 수 있는 디버거 옵션(`-Djdk.tracePinnedThreads=short, full` )이 존재하므로 이를 활용하여 이런 상황을 방지할 수 있습니다.

하지만 사용하는 라이브러리에서 `synchronized`를 사용하고 있다면 생각치 않게 pinning이 발생할 수 있습니다. 대표적으로 MySQL이 있습니다. 따라서 라이브러리를 사용하는 곳에서 Virtual Thread를 사용할 때는 해당 라이브러리에서 `ReentrantLock`을 지원하고 있는지 확인해야 합니다.

### CPU bound 작업

Virtual Thread는 결국 캐리어 스레드 위에서 돌아가는 만큼, 장점인 Nonblocking I/O를 활용하지 못하는 곳에서는 추가적인 오버헤드만 발생하게 됩니다. 즉, CPU bound 위주의 작업을 처리할 때 Virtual Thread는 오히려 성능 저하만 발생시킬 수 있으므로 사용 시 반드시 I/O 작업의 비중을 잘 확인해야 합니다.

### Thread Local

Virtual Thread는 매번 생성되는데 제한이 거의 없기 때문에 아주 많은 수가 생성될 수 있습니다. 이 때문에 Thread Local 사용 시 메모리를 너무 많이 사용하게 되므로 Virtual Thread를 사용하는 곳에서 Thread Local의 사용은 주의해야 합니다.

### 결론

대부분 서비스들이 여러 라이브러리를 의존하고 있는 복잡한 웹 어플리케이션인 환경에서 아직은 모든 라이브러리들이 pinning에 대한 대응이 되지 않았을 가능성이 큽니다. 또한 Virtual Thread 자체는 수많은 요청을 처리할 수 있지만 인프라와 서비스 간의 병목이 있다면 성능의 개선을 기대하기 힘들고 Connection timeout과 같은 에러가 발생할 수 있는 여지가 있습니다.

Virtual Thread가 갖고 있는 장점도 뚜렷하지만 등장한 지 얼마 되지 않은 기술인 만큼 이에 대한 대응이 부족하고, 따라서 Virtual Thread를 도입하기 위해서는 활용하려는 부분이 어떤 외부 서비스, 인프라, 라이브러리 등을 의존하고 있는지를 확인하는 일이 필수적입니다.

## 참고 자료

[우아한 기술블로그](https://techblog.woowahan.com/15398/)

[카카오페이 기술블로그](https://tech.kakaopay.com/post/ro-spring-virtual-thread/)

[Oracle Virtual Threads](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)

[OpenJDK Project Loom](https://www.baeldung.com/openjdk-project-loom)
