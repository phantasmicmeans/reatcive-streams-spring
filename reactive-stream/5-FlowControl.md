## 흐름 제어
푸시모델만 사용하는것으로는 기술적 한계가 있다. 

메시지 기반 통신의 본질은 요청에 응답하는 것이다. Producer가 Subscriber의 처리 능력을 무시하면 시스템에 악영향을 끼칠 수 있기 때문에 매우 까다롭다.

### 느린 프로듀서와 빠른 컨슈머
컨슈머가 매우 빠르게 동작하는 상황에서 프로듀서가 느리게 동작한다고 가정하자. 
이러한 상황은 컨슈머의 능력을 프로듀서가 믿지 못하기에 발생할 수 있다.

상황에 따라 컨슈머의 처리 능력이 동적으로 변할 수도 있다. 가장 쉽게는 프로듀서 수를 늘려서 컨슈머에게 부담을 증가시킬 수 있고 그렇지만..

결론적으로 이러한 문제를 해결하기 위해 필요한 것은 `실제적 요구`이다.
순수 푸시모델은 이러한 메트릭을 제공할 수 없으므로 동적으로 시스템의 처리량을 증가시키는 것이 불가능하다.

### 빠른 프로듀서와 느린 컨슈머
이 경우는 더 복잡하다. 프로듀서는 컨슈머의 능력보다 더 많은 데이터를 전송할 수 있고, 이로 인해 부하를 받는 컴포넌트에게 치명적 오류를 야기할 수 있다.

이 문제를 해결하기 위한 가장 직관적인 방법은 `큐`에 원소를 수집하는 것이다. 이 큐는 프로듀서와 컨슈머 사이에 있을 수도 있고 컨슈머측에 있을수도 있다.
컨슈머의 능력이 느리더라도 큐를 사용하면 처리가 가능하다. 

이러한 큐를 사용하게 되면 적절한 특성을 가진 큐를 선택하는게 중요하다. 

### 무제한 큐
가장 확실한 방법은 큐 사이즈 limit을 없애는 것이다. 생성된 모든 데이터가 큐로 인입되고 추후 subscriber가 이를 사용한다.
무제한 큐의 이점은 메시지 전달에 대한 확신을 가질 수 있다는 것이다. 
모든 메시지는 결국 컨슈머에 전달 될 것이고 컨슈머는 이를 처리 할 것이다. 

But, 실제로 무제한의 리소스를 사용할 수 없으므로 메시지 전달을 계속 수행하면 손상이 올 수 있다. (ex. 메모리 한도 도달)

### 크기가 제한된 드롭 큐
메모리 오버플로우를 방지하기 위해 큐가 차면 신규 메시지를 무시하는 형태로 사용할 수 있다.
그러나 이 큐는 메시지의 중요성이 낮을 때 일반적으로 사용한다. 메시지 손실이 빈번하게 발생한다.

### 크기가 제한된 블록킹 큐
`크기가 제한된 드롭 큐`는 메시지가 중요한 경우 사용불가하다. 결제 시스템 같은 경우 메시지는 반드시 처리되어야하며 위와 같은 큐를 사용할 수 없다. 
대신 제한이 도달하면 메시지 유입을 차단할 수 있다. 유입을 차단하는 기능을 특징으로 하는 큐를 일반적으로 Blocking Queue라고 한다. 
그러나 큐 사이즈의 한계에 도달하면 차단이 시작되고 여유공간이 존재할 때 까지 차단 상태가 된다.

즉 컨슈머의 처리속도가 늦을수록 시스템 전체 처리량이 제한된다고 볼 수 있다. 또한 차단에 의해 비동기 동작을 모두 무효화하므로 효율적으로 리소스를 사용하진 못한다.

결과적으로 `복원력`, `탄력성`, `응답성` 모두를 달성하고자 한다면 좋지 않은 시나리오이다.

위의 예시들처럼 적합한 제어를 추가하지 않은 `푸시 모델`은 다양한 부작용을 발생 시킬 수 있다. 

**Reactive Stream이 시스템 부하에 적절히 대응하는 방법, 즉 BackPressure 메커니즘의 중요성을 언급한 이유이다.**

앞장에서 살펴봤듯이 Reactive Stream은 Publisher, Subscribe, Subscription, Processor 네가지 기본 인터페이스로 이루어져있다.
Publisher-Subscriber는 Observable-Observer와 유사하다는 것도 앞장에서 설명했다.

```java
package org.reactivestreams;

public interface Publisher<T> {
    void subscribe(Subscriber<? super T> var1);
}
``` 

```java
package org.reactivestreams;

public interface Subscriber<T> {
    void onSubscribe(Subscription var1);

    void onNext(T var1);

    void onError(Throwable var1);

    void onComplete();
}
```

```java
package org.reactivestreams;

public interface Subscription {
    void request(long var1);

    void cancel();
}
```

BackPressure 메커니즘을 설명하기 위해 다시 Subscriber의 명세를 보자. 앞장에서의 내용처럼 Subscription은 Publisher와 Subscriber 사이에서 데이터 생성을 제어하기 위한 기본적인 사항을 제공한다.

cancel() 메소드로 발행을 완전히 취소할 수도 있지만, 중요한 개념은 단연코 `request(n)` 메소드이다. 
이제 큐를 두지 않아도 된다. Subscriber는 request 메소드를 통해 Publisher가 보내주어야 하는 데이터의 크기를 결정한다. 
즉, Publisher에서 유입되는 원소의 개수가 처리할 수 있는 제한을 초과하지 않음을 확신할 수 있게 되었다.

기본 메커니즘은 아래 다이어그램으로..

<img src="https://user-images.githubusercontent.com/20153890/98534650-1ec59180-22c8-11eb-8979-4269e038b947.png" width=500>

위 다이어그램은 Subscriber가 요청한 경우에만 새로운 데이터를 보내도록 보장한다. Publisher의 전체 구현은 순수 블록킹부터, Subscriber의 요청에 대해서만 데이터를 생성하는 위와 같은 메커니즘까지 다양하게 있다. 그러나 `큐`는 필요없다. (추후 큐가 등장하기는 하나 일단 이렇게만 이해해도 충분하다)

또한 순수 푸시모델과는 다르게 하이브리드 푸시-풀 모델도 지원한다.
- 순수 푸시모델로 사용하고 싶은 경우는 Long.MAX_VALUE로 request요청을 행하면 된다.
- 순수 풀 모델로 사용하고 싶은 경우는 Subscriber#onNext가 호출될 때 마다 request를 한개씩 요청하면 된다. 

**결론은 Reactive Stream이 시스템 부하에 적절히 대응하는 방법, 즉 BackPressure 메커니즘의 근간은 Subscription이다.**