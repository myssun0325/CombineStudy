# Ch 18: Custom Publishers & Handling Backpressure



- 무슨농담일까요? “What is this backpressure thing? Is that some kind of back pain induced by too much leaning over your chair, scrutinizing Combine code? ”



#### Creating your own publishers

- own publisher를 만들기 위해 살펴볼 3가지 방법
  1. Publisher 네임스페이스안에서 간단한 extension 사용
  2. Subscription(produces values) 을 사용해서 Publisher 네임스페이스 안에서 새로운 타입을 정의한다.
  3. 위와 유사한 방법으로 Subscription(transform values from an upstream publisher)을 가지고 새로운 타입을 정의한다.
     - 차이: 값을 생성하는 subscription이냐, 업스트림으로부터 값을 변형하는 subscription이냐.



> Note: 기술적으로 custom subscription없이 커스텀 publisher를 만드는것은 가능하다. 만약에 이렇게 만들게 되면, subscriber의 요구에 대응하기 어려울 수 있다.(which makes your publisher illegal in the Combine ecosystem.) 빠른 cancellation(early cancellation)도 문제가 될 수 있다. 이런 접근법은 추천하지 않는다. 이번챕터에서 어떻게 publisher를 올바르게 작성하는지 배우게 될 것이다.



#### Publishers as extension methods

- 첫번째 할 일은 이미 존재하는 오퍼레이터들을 사용해서 간단한 오퍼레이터를 구현하는 것이다. 
- `unwrap()`오퍼레이터를 추가다고 생각해보자.(unwrap은 옵셔널 밸류를 unwrap하고 nil을 무시하는 오퍼레이터다)

```swift
extension Publisher {
    func unwrap<T>() -> Publishers.CompactMap<Self, T> where Output == Optional<T> {
        compactMap { $0 }
    }
}

```

- `Publishers.CompactMap`은 제네릭 타입이다. (`CompactMap<Upstream, Output>`)
  - 정의를 보면 알 수 있음. (struct로 되어있음)
  - `Upstream`이 `Self`고 `Output`은 wrapped type이다.
  - 마지막으로 Optional 타입이라는 제약을 걸어준다.
  - 이제 완성된 오퍼레이터를 랩핑된 타입 T에 사용하면 됨.

> 더 복잡한 오퍼레이터를 메서드로 만들 때(such as when using a chain of operators), 서명이 더 빠르게 복잡해 질 수 있다.(?)무슨말이지.. (the signature can quickly become very complicated.) -> 여기서 말하는 시그니쳐는 호출부의 모양을 말하는 것일까요?
>
> 복잡해지는 것을 해결하기 위한 좋은 테크닉은 오퍼레이터의 리턴타입을 AnyPublisher<Output, FailureType>으로 만드는 것이다. 결국 publisher를 반환하게 될것이고 type-erase the signature을 하기 위해 `eraseToAnyPublisher()`로 끝나게 될것이다.



#### Testing your custom operator

```swift
let values: [Int?] = [1, 2, nil, 3, nil, 4]

values.publisher
    .unwrap()
    .sink {
        print("Received value: \($0)")
    }

/*
Received value: 1
Received value: 2
Received value: 3
Received value: 4
*/
```

- publisher를 두가지 그룹으로 나눌 수 있다.
  - "producers"로 동작하는 Publisher, 직접(directly) 값을 제공한다.
  - "transformers"로 동작하는 Publisher, upstream으로부터 제공된 값을 transforming 한다.
- 이번 챕터에서 이 두 가지를 어떻게 사용하는지에 대해서 배우게 된다. 하지만 그 전에 publisher를 구독하게 되었을 때 내부적으로 어떤 일이 일어나는 지에 대한 이해가 필요하다.



#### The subscription mechanism

- Subscription은 Combine의 이름없는 영웅입니다.(?)
  - publisher를 구독할 때, `subscription`을 인스턴스화한다. subscription은 subscriber로부터 요청을 받고 이벤트를 생성한다. (for example, values and completion)

![image-20200705231235815](ch18_customPublishers_handlingBackPressure.assets/image-20200705231235815.png)

(2장에서 배웠던 다이어그램과 각 단계에서 호출하는 메서드를 참고하면 이해하기 좋음)



![image-20200705225011351](ch18_customPublishers_handlingBackPressure.assets/image-20200705225011351.png)

1. subscriber가 publisher를 **구독(subscribe)** 한다.
2. Publisher가 **Subscription**을 생성하고 subscriber에게 subscription을 준다. (calling `receive(subscription:)`) , (subscriber가 subscription을 받는다.)
3. subscriber가 subscription으로부터 value를 요청한다.  원하는 value가 몇개 인지 요구한다.(by sending it the number of values it wants) == subscription의 `request(_:)` 메서드를 호출한다.
4. subscription은 작업을 시작하고 값을 방출하기 시작한다. 하나씩 방출된 값들을 subscriber에게 보낸다. (subscriber의 `receive(_:)` 메서드를 호출한다.)
5. value를 받으면 subscriber는 새로운 `Subscriber.Demand`를 방출한다. (이전 total demand에 추가된다.)
6. subscription은 요청한 total number에 도달할 때까지 value를 계속 보낸다.



- 만약에 subscription이 subscriber가 요구한 만큼의 많은 value를 보냈다면 value를 더 보내기전에 새로운 demand request를 기다려야 한다.
- Q. 아래 문장 좀 어렵네요.
- 이런 메커니즘을 우회하면서 값을 계속 보낼 수 있지만, 그렇게 하게 되면 subscriber와 subscription의 계약을 깨지게 된다. 뿐만 아니라 Apple의 정에 따라 Publisher tree에서 정의되지 않은 동작이 발생할 수 있다.
  - You can bypass this mechanism and keep sending values, but that breaks the contract between the subscriber and the subscription and can cause undefined behavior in your publisher tree based on Apple's definition.
- 만약에 subscription의 value source가 complete되거나 에러가 발생하면 subscription은 subscriber의 `receive(completion:)` 메서드를 호출하게 된다.



#### Publisher emitting values

- 챕터 11 Timers에서 `Timer.publish()`를 배웠지만 타이머에 Dispatch Queue를 사용하는 것이 불편하다는 것을 알았다. 디스패치의 **DispatchSourceTimer**를 기반으로 자신만의 타이머를 만들어보는건 어떤가요. 그렇게 하기 위해서 `Subscription` 메커니즘에 대한 세부정보를 확인면 됩니다. (문장이 이해가 잘 안가네요..)

```swift
struct DispatchTimerConfiguration {
    // 1
    let queue: DispatchQueue?
    // 2
    let interval: DispatchTimeInterval
    // 3
    let leeway: DispatchTimeInterval
    // 4
    let times: Subscribers.Demand
}
```

1. 타이머가 특정 큐에서 시작하기 위함. 큐를 신경쓰고 싶지 않은 경우를 대비해 옵셔널로 선언. 이 경우엔 선택된 큐에서 시작.
2. 구독시점으로부터 타이머가 시작되는 간격

3. 시스템이 타이머의 이벤트를 지연시킬 수 있는 시간이 있는데 이 지연가능한 시간의 최대 시간을 leeway라고 함.
4. 받고자 하는 타이머 이벤트의 수. 지금은 custom 타이머를 만드는 중이므로 완료하기 전에 유연하고 제한된 수의 이벤트를 전달할 수 있다.



#### Adding the DispatchTimer publisher

- `DispatchTimer` publisher를 생성함으로써 시작할 수 있다. 모든 작업들이 subscription안에서 일어나기 때문에 매우 직관적일것이다.

```swift
extension Publishers {
    struct DispatchTimer: Publisher {
        // 5
        typealias Output = DispatchTime
        typealias Failure = Never
        
        // 6
        let configuration: DispatchTimerConfiguration
        
        init(configuration: DispatchTimerConfiguration) {
            self.configuration = configuration
        }
    }
}
```

5. 현재 시간을 `DispatchTime` value로 방출한다. 물론 절대 실패하지 않는다. (Failure type is Never.)
6. 주어진 configuration의 복사본을 유지한다. 지금 당장은 사용하지 않을 거고 뒤에서 subscriber를 받을 때 사용할 거임.

> Note: 이 코드를 작성할 때 컴파일러 에러가 날것이다. 아직 작성하지 않은 구현부에 대해서.

```swift
// 7
func receive<S>(subscriber: S) where S : Subscriber, Self.Failure == S.Failure, Self.Output == S.Input {
  // 8
  let subscription = DispatchTimerSubscription(
    subscriber: subscriber,
    configuration: configuration
  )
  // 9
  subscriber.receive(subscription: subscription)
}
```

7. function은 제네릭이다. subscriber타입에 매치하기 위해 compile-time specialization가 필요하다.
8. `DispatchTimerSubscription`내에서 발생할 bulk action. `DispatchTimerSubscription`은 곧 작성할 것.
9. 챕터2에서 배웠던 것. subscriber는 subscription을 수신한다. 그리고 이를 통해 value에 대한 request를 보낼 수 있다.



#### Building your subscription

- subscription의 역할
  - subscriber로부터 최초 요구를 받아들인다. (accept the initial demand)
  - 요구에 따른 timer event 생성
  - subscriber가 value를 받고 demand를 반환할 때마다 demand count에 추가한다. 
  - configuration에서 요청한 값보다 많은 값을 제공하지 않아야 한다. 

- Subscription을 만들어보자

```swift
private final class DispatchTimerSubscription<S: Subscriber>
	: Subscription where S.Input == DispatchTime {
 }
```

- 이 시그니쳐가 주는 정보들
  - 이 subscription은 밖으로 보이지 않고. Subscription 프로토콜을 통해서만 접근할 수 있다. `private`
  - 참조를 넘겨주기 위해 class 타입으로 선언. subscriber는 Cancellable collection을 추가할 수 있을 뿐만 아니라 `cancel()`을 독립적으로 호출할 수 있다. (cancel()호출이나 Cancellable bag 같은걸 추가할 수 있다는 뜻)
  - input value의 타입이 DispatchTime인 subscriber에게 이 subscription이 방출하는 value를 제공한다.



#### Adding required properties to your subscription

```swift
private final class DispatchTimerSubscription<S: Subscriber>: Subscription where S.Input == DispatchTime {
  // 10
  let configuration: DispatchTimerConfiguration
  // 11
  var times: Subscribers.Demand
  // 12
  var requested: Subscribers.Demand = .none
  // 13
  var source: DispatchSourceTimer? = nil
  // 14
  var subscriber: S?
}
```

10. subscriber가 주는 configuration
11. configuration으로부터 복사된 타이머의 최대 횟수. value를 보낼 때마다 감소하는 카운터로 사용한다.
12. subscriber의 현재 demand(value를 보낼 때마다 감소한다.)
13. 타이머 이벤트를 생성하는 internal DispatchSourceTimer
14. subscriber. subscription이 complete이 되지 않는 한 subscriber의 retain할 책임을 갖는다는 것이 명확해진다.
    - **the subscription is responsible for retaining the subscriber for as long as it doesn't complete, fail or cancel.**



> Note: 마지막 14번은 Combine의 소유권(ownership)의 매커니즘을 이해하는데 중요하다. Subscription은 subscriber와 publihser간의 연결이다. 예를 들어 클로저를 홀딩하고 있는 object(like AnySubscriber or sink)
>
> 만약에 subscription에서 참조를 홀드하고 있지 않으면 subscriber는 값을 받지 못하는 것을 볼 수 있을 것이다. subscription이 해제되지마자 모든 것이 중지된다. 내부 구현은 물론 publisher의 세부사항에 따라 달라질 수 있다.



#### Initializing and canceling your subscription

- 이제 subscription에 이니셜라이저를 추가해보자

```swift
init(subscriber: S,
     configuration: DispatchTimerConfiguration) {
  self.configuration = configuration
  self.subscriber = subscriber
  self.times = configuration.times
}
```

- times: maximum number of times the publisher should receive timer events, as the configuration specifies. publisher가 이벤트를 방출할 때마다 이 카운터는 감소한다. zero 도착했을 때 timer는 finished 이벤트와 함께 종료된다.
- 이제 `cancel()`을 구현해보자. cancel메서드는 Subscription에서 반드시 제공해야되는 메서드이다.(required)

```swift
func cancel() {
  source = nil
  subscriber = nil
}
```

- `DispatchSourceTimer`를 nil로 하는것은 running을 멈추기에 충분하다. subscriber프로퍼티를 nil로 설정하게 되면 subscription로부터 해제된다. 더 이상 필요하지 않은 객체를 메모리에서 유지하지 않으려면 반드시 자신의 subscription에서 이 작업을 수행해야 한다.
- You can now start coding the core of the subscription: `request(_:)`



#### Letting your subscription request values

- 2장에서 배운거처럼 한번 subscriber가 subscription을 얻게되면 subscription으로부터 반드시 value를 `request`(요청)해야한다. 이를 구현하기 위해 아래의 메서드를 클래스에 추가한다.

```swift
 // 15
func request(_ demand: Subscribers.Demand) {
  // 16
  guard times > .none else {
    // 17
    subscriber?.receive(completion: .finished)
    return
  }
}
```

15. subscriber로부터 demand를 받는다. **Demand는 누적된다.** 
    - **Demands are cummulative: They add up** to form a total number of values that the subscriber requested.
16. 첫 번째 테스트는 configuration에 지정된 대로 subscriber에게 충분한 값을 보냈는지 확인하는 것이다. 즉 publisher가 받은 demands와 무관하게 에상했던 value들의 최대값을 보낸 경우다.
17. 이 경우 publisher가 value보낸것을 완료했다고 subscriber에게 알릴 수 있다.

- 계속 구현을 마저해보자.

```swift
// 18
requested += demand

// 19
if source == nil, requested > .none {

}
```

18. 새로운 demand가 추가됨에 따라 the total number of requested values를 증가시킨다.
19. 타이머가 이미 존재하는지 확인하고 아니라면 요청했던 value가 있는지 확인하고, 시작한다.



#### Configuring your timer

- 아래 코드를 if 조건문에 추가한다.

```swift
// 19
if source == nil, requested > .none {
  // 20
  let source = DispatchSource.makeTimerSource(queue: configuration.queue)
  // 21
  source.schedule(deadline: .now() + configuration.interval,
                  repeating: configuration.interval,
                  leeway: configuration.leeway)
}
```

20. configuration의 queue로부터 DispatchSourceTimer를 생성한다.
21. configuration.interval 초마다 시작하기 위해 타이머를 스케쥴링한다.

- 타이머가 시작되면 subscriber에게 이벤트를 보내는데, 이벤트를 사용하지 않더라도 타이머가 중지하지 않는다. subscriber가 구독을 취소하거나 subscription을 할당해제 할 때 까지 계속 실행된다.

- 이제 타이머의 core부분을 코딩할 준비가 되었다. 

22. 타이머의 event handler를 세팅. 타이머가 fire될 때마다 실행되는 클로져. weak self 주의
23. 현재 요청된 값을 확인하다. publisher가 현재  demand가 없을 경우 중지될 수 있다. (나중에 backpressure에 대해 배운다.)
24. value를 방출하기 위해 두 개의 카운터를 감소시킨다.
25. subscriber에게 값을 보낸다.
26. 전송할 value의 총 값이 configuration의 명시된 maximum에 도달하게 되면 publisher가 끝났다고 간주하고 completion event를 방출한다.

#### Activating your timer

- configuration이 끝났으니 이제 timer를 activate시켜보자.
- 그 동안 많은 단계가 있었다. 코드들을 올바르지 못한곳에 넣을 여지가 있다.
- Final.playground과 비교하면서 코드를 
- Last step: Add this extension after the entire definition of DispatchTimerSubscription

```swift
extension Publishers {
    static func timer(queue: DispatchQueue? = nil,
                      interval: DispatchTimeInterval,
                      leeway: DispatchTimeInterval = .nanoseconds(0),
                      times: Subscribers.Demand = .unlimited) -> Publishers.DispatchTimer {
        return Publishers.DispatchTimer(configuration: .init(queue: queue,
                                                             interval: interval,
                                                             leeway: leeway,
                                                             times: times)
        )
    }
}
```



#### Testing your timer

- 대부분의 new timer 오퍼레이터의 파라미터들은 (interval을 제외한) default value를 갖는다.(일반적인 케이스에서 쉽게 사용하기 위해서). 이 defaults들은 멈추지 않는 timer를 생성한다. (has minimal leeway, 특정 지정된 큐가 없다.)
- 테스트 코드를 추가해보자.

```swift
// 27
var logger = TimeLogger(sinceOrigin: true)
// 28
let publisher = Publishers.timer(interval: .seconds(1), times: .max(6))
// 29
let subscription = publisher.sink { time in
    print("Timer emits: \(time)", to: &logger)
}
/*
+1.04550s: Timer emits: DispatchTime(rawValue: 4785726696038)
+2.04384s: Timer emits: DispatchTime(rawValue: 4786725746396)
+3.04480s: Timer emits: DispatchTime(rawValue: 4787726655515)
+4.04486s: Timer emits: DispatchTime(rawValue: 4788726734133)
+5.04381s: Timer emits: DispatchTime(rawValue: 4789725682830)
+6.04484s: Timer emits: DispatchTime(rawValue: 4790726692953)
*/
```

27. TimerLogger -> 챕터10 참고, 그 때랑 다른점: 시간차이나, 경과시간을 로깅할 수 있다.
28. 매 초마다 찍히고 정확히 6번 fire 된다.
29. TimeLogger를 통해 전달받은 value를 로깅한다.

- test canceling your timer.
  - add this code 

```swift
DispatchQueue.main.asyncAfter(deadline: .now() + 3.5) {
    subscription.cancel()
}
```

- 3개의 value만 방출되고 말것이다. `cancel` 확인(cancel 구현이 잘됐음을 확인)
- Combine API에서 겉으로 발견하기 힘들지만, 우리가 배웠던거처럼 Subscription에서 대부분의 작업들이 이루어진다.



### Publishers transforming values

- 챕터9에서 배웠던 `share`오퍼레이터에 대한 얘기로 시작.
- `shareReplay()` 만들기. 오퍼레이터를 작성하기 위해 아래의 동작을 수행하는 publisher를 만들것임. 만드려는 Publisher는 아래의 동작을 수행합니다.
  - upstream의 구독을 첫번째 구독자에게 구독시킨다.? (Subscribes to the upstream publisher upon the first subscriber.)
  - 마지막 N개의 값을 구독자들에게 replay합니다.
  - 만약에 completion이 흘렀다면 새로운 구독자에게 completion을 알려줍니다.

- `ShareReplay operator` 코드 예제



#### Implementing a ShareReplay operator

- `shareReplay()`를 구현하기 위해 필요한 것.
  - `Subscription`프로토콜 채택, 이 구독은 각 subscriber들이 받게된다. subscriber들의 demand와 cancellation을 처리하기 위해 각 subscribere들은 독립된 subscription을 받게된다.
  - `Publisher`프로토콜을 채택. subscriber이 share를 해야되기 때문에 클래스로 구현해야됨.(참조필요)

```swift
// 1. subscription을 구현하기 위해서 Publisher, Subscriber가 subscription에 access하고 mutate
// 할 수 있어야 한다.
fileprivate final class ShareReplaySubscription<Output, Failure: Error>: Subscription {
    // 2. replay buffer's maximum capacity (이니셜라이져에서 초기화)
    let capacity: Int
    // 3. subscription동안 subscriber에 대한 참조를 유지. type system을 편리하게 사용하기 위해 type erase
    var subscriber: AnySubscriber<Output, Failure>? = nil
    // 4. publisher가 subscriber로부터 받은 demand를 누적요청을 트래킹함으로써
    // 요청된 수의 값을 정확하게 전달할 수 있다.
    var demand: Subscribers.Demand = .none
    // 5. 보류중인 값(=subscriber에게 전달되거나 버려지기전까지의 값)을 저장합니다.
    var buffer: [Output]
    // 6. completion이 난 경우 이를 통해 새로운 subscriber에게 completion을 준다.
    var completion: Subscribers.Completion<Failure>? = nil
}

```



아래 Note내용을 잘 모르겠네요

> completion 이벤트를 즉시 전달만 해야되고 completion이벤트를 keep하는게 불필요하다고 생각되면, rest assured that's note the case. 구독자는 subscription을 받고 (구독을하고) completion이벤트를 첫 이벤트로 받는다. (만약에 이전에 completion이 방출됐다면) . 첫번째 `request(:_)`가 이 시그널을 만든다. publisher는 이 request가 언제 발생할지 모른다. 그러므로 항상 대비해야하고, subscriber에게 적시에 completion을 전달해야한다.

> Note: If you feel that it’s unnecessary to keep the completion event around when you’ll just deliver it immediately, rest assured that’s not the case. The subscriber should receive its subscription *first*, then receive a completion event — if one was previously emitted — as soon as it is ready to accept values. The first request(_:) it makes signals this. The publisher doesn’t know when this request will happen, so it just hands the completion over to the subscription to deliver it at the right time.



#### Initializing your subscription

- subscription의 정의에 아래 코드를 추가

```swift
    // upstream publisher로부터 여러 value들을 받고, 이 subscription 인스턴스에 세팅한다.
    init<S>(subscriber: S,
            replay: [Output],
            capacity: Int,
            completion: Subscribers.Completion<Failure>?)
            where S: Subscriber,
            Failure == S.Failure,
            Output == S.Input {
                //7. subscriber의 type-erase version을 저장
                self.subscriber = AnySubscriber(subscriber)
                // upstream publisher의 현재 버퍼, maximumCapacity, completion event 저장
                self.buffer = replay
                self.capacity = capacity
                self.completion = completion
    }
```



#### Sending completion events and outstanding values to the subscriber

```swift
    // relays completion events to the subscriber
    private func complete(with completion: Subscribers.Completion<Failure>) {
        // 9. 메서드의 duration동안 subscriber를 유지한다. class에서 nil로 설정한다.
        // 이 방어코드는 subscriber가 잘못된 호출을 했을 경우 모든 issue를 무시하도록 보장한다.
        guard let subscriber = subscriber else { return }
        // 10. completion은 nil로 할당함으로써 한번만 방출된다. 그리고 버퍼를 비운다.
        self.subscriber = nil
        self.completion = nil
        self.buffer.removeAll()
        // relay the completion event to the subscriber
        subscriber.receive(completion: completion)
    }
```

```swift
    // emit outstanding values to the subscriber.
    private func emitAsNeeded() {
        guard let subscriber = subscriber else { return }
        //12. demand가 있고 버퍼에 값이 있는 경우에만 emit values
        while self.demand > .none && !buffer.isEmpty {
            // 13. outstanding demand를 한개 감소
            self.demand -= .max(1)
            // 14. subscriber에게 첫번째 (buffer)outstanding value를 보내고 응답으로 새로운 demand를 받는다.
            let nextDemand = subscriber.receive(buffer.removeFirst())
            // 15. 새로운 demand가 .none이 아니라면 total demand에 새로운 demand를 추가한다.
            // 그렇지 않으면 crash가 날꺼다. Combine은 Subscribers.Demand.none을 zero로 취급하지 않는다.
            // .none을 빼거나 더하는건 exception 에러를 발생시킴.
            if nextDemand != .none {
                self.demand += nextDemand
            }
        }
        // 만약에 completion이벤트가 보류중이라면 이 때 보낸다.
        if let completion = completion {
            complete(with: completion)
        }
    }
```

```swift
    // Subscription프로토콜의 요구 메서드 작성.
    func request(_ demand: Subscribers.Demand) {
        // check crash
        if demand != .none {
            self.demand += demand
        }
        emitAsNeeded()
    }
```

> Note: demand가 .none인 경우에도 `emitAsNeeded()` 를 호출해도 이미 발생한 completion 이벤트를 적절하게 relay할 수 있다.



#### Canceling your subscription

```swift
    // cancel the subscription
    func cancel() {
        complete(with: .finished)
    }
```

```swift
    func receive(_ input: Output) {
        guard subscriber != nil else { return }
        // 17. value를 buffer에 추가합니다.
        // 무제한 요구(demand)와 같이 일반적인 케이스에 대해서 최적화를 할 수 있지만
        // 이번 예제에서는 그렇제 하진 않겠다.
        buffer.append(input)
        if buffer.count > capacity {
            // 18. buffer의 보다 요청된 capacitiy가 작다는걸 보장합니다.
            // 무슨 말인지 모르겠는 아랫말..
            // You handle this on a rolling, first-in-first-out basis – as an already-full buffer receives each new value, the current first value is removed.
            buffer.removeFirst()
        }
        // 결과를 subscriber한테 전달합니다.
        emitAsNeeded()
    }
```

#### Finishing your subscription

```swift
    // accept completion events and subscription will complete.
    // subscriber를 삭제하고, 버퍼를 비운다. (good for memory management)
    // completion이벤트를 downstream으로 보낸다.
    func receive(completion: Subscribers.Completion<Failure>) {
        guard let subscriber = subscriber else { return }
        self.subscriber = nil
        self.buffer.removeAll()
        subscriber.receive(completion: completion)
    }
```

#### Coding your publisher

- Publisher는 대게 값 타입으로 struct로 구현한다. 하지만 가끔 Publishers.Multicast(multicast()를 리턴하는 애와 같이 class로 구현해야한다. 이런 publisher에게 클래스가 필요하지만, 규칙의 예외는 아니지만, 구조체를 사용하는 경우가 대부분입니다.

```swift

extension Publishers {
    // 20. 오퍼레이터의 single instance를 subscriber들에게 share. 그래서 클래스 사용. It’s also generic, with the final type of the upstream publisher as a parameter.
    final class ShareReplay<Upstream: Publisher>: Publisher {
        // 21. doesn't change the output or failure types of the upstream publisher
        typealias Output = Upstream.Output
        typealias Failure = Upstream.Failure
    }
}
```

#### Adding the publisher's required properties

```swift
extension Publishers {
    // 20. 오퍼레이터의 single instance를 subscriber들에게 share. 그래서 클래스 사용. It’s also generic, with the final type of the upstream publisher as a parameter.
    final class ShareReplay<Upstream: Publisher>: Publisher {
        // 21. doesn't change the output or failure types of the upstream publisher
        typealias Output = Upstream.Output
        typealias Failure = Upstream.Failure
        
        // 22. 동시에 여러 subscriber에게 feeding할꺼라서 lock을 사용.
        // you'll need a lock to guarantee exclusive access to your mutable variables.
        private let lock = NSRecursiveLock()
        // 23. upstream에 대한 reference 유지
        private let upstream: Upstream
        // 24. maximum recording capacity of your replay buffer
        private let capacity: Int
        // 25. storage for the recorded values.
        private var replay = [Output]()
        // 26. 다수의 subscriber에게 feed하기 때문에 그들에 event를 알려주기 위해사용.
        // 각 subscriber는 각 전용의 subscription으로부터 값을 받는다. (subscriber랑 1:1)
        private var subscriptions = [ShareReplaySubscription<Output, Failure>]()
        // 27. replay value even after completion,
        private var completion: Subscribers.Completion<Failure>? = nil
    }
}
```

- In the end, you’ll see it’s not that much, but there is housekeeping to do, like using proper locking, so that your operator will run smoothly under all conditions. > 문장 이해안감

#### Initializing and relaying values to your publisher

```swift
        // initializer
        init(upstream: Upstream, capacity: Int) {
            self.upstream = upstream
            self.capacity = capacity
        }
        
        private func relay(_ value: Output) {
            // 28. mutable
            lock.lock()
            // 꼭 defer를 써야하는건 아니지만, 나중에 메서드를 변경하거나 early return 문을 추가할 경우에 대해 unlock하는걸 까먹는걸 방지하기 위한 연습.
            defer { lock.unlock() }
            
            // 29. upstream이 아직 complete 되지 않은 경우에만
            guard completion == nil else { return }
            
            // 30. Adds the value to the rolling buffer and only keeps the latest values of capacity. These are the ones to replay to new subscribers.
            // rolling buffer = 순환 버퍼?
            replay.append(value)
            if replay.count > capacity {
                replay.removeFirst()
            }
            
            // 31. Relays the buffered value to each connected subscriber.
            subscriptions.forEach {
                _ = $0.receive(value)
            }
        }
```



#### Letting your publisher know when it's done

```swift
        private func complete(_ completion: Subscribers.Completion<Failure>) {
            lock.lock()
            defer { lock.unlock() }
            // 32. 추후 subscriber들에 대한 completion event 저장
            self.completion = completion
            // 33. Relaying it to each connected subscriber
            subscriptions.forEach {
                _ = $0.receive(completion: completion)
            }
        }
```

- This standard prototype for receive(subscriber:) specifies that the subscriber, whatever it is, must have Input and Failure types that match the publisher’s Output and Failure types. 

```swift
        func receive<S: Subscriber>(subscriber: S) where Failure == S.Failure,
            Output == S.Input {
                lock.lock()
                defer { lock.unlock() }
        }
```

#### Creating your subscription

- 위의 receive메서드를 보충.

```swift
        // prototype for receive(subscriber:)
        func receive<S: Subscriber>(subscriber: S) where Failure == S.Failure,
            Output == S.Input {
                lock.lock()
                defer { lock.unlock() }
                // 34. 새로운 subscription은 subscriber에 대해 참조하고 현재 replay buffer의 값, capacity, outstanding completion event를 받는다.
                let subscription = ShareReplaySubscription(
                    subscriber: subscriber,
                    replay: replay,
                    capacity: capacity,
                    completion: completion)
                
                // 35. keep the subscription around to pass future events to it.
                subscriptions.append(subscription)
                // 36. subscriber에게 subscription을 보냄.
                subscriber.receive(subscription: subscription)
        }
```

#### Subscribing to the publisher and handling its inputs

- 이제 upstream publisher를 구독할 준비가 되었다. 

```swift
        // prototype for receive(subscriber:)
        func receive<S: Subscriber>(subscriber: S) where Failure == S.Failure, Output == S.Input {
            lock.lock()
            defer { lock.unlock() }
            // 34. 새로운 subscription은 subscriber에 대해 참조하고 현재 replay buffer의 값, capacity, outstanding completion event를 받는다.
            let subscription = ShareReplaySubscription(
                subscriber: subscriber,
                replay: replay,
                capacity: capacity,
                completion: completion)
            
            // 35. keep the subscription around to pass future events to it.
            subscriptions.append(subscription)
            // 36. subscriber에게 subscription을 보냄.
            subscriber.receive(subscription: subscription)
            // 37. upstream으로부터 1번만 구독
            guard subscriptions.count == 1 else { return }
            // 38. 간단한 AnySubscriber사용. 클로저를 인자로 받고 즉시 .unlimited 리퀘스트를 보낸다.
            let sink = AnySubscriber(
                receiveSubscription: { subscription in
                    subscription.request(.unlimited)
               // 39. Relay values you receive to downstream subscribers.
            }, receiveValue: { [weak self] (value: Output) -> Subscribers.Demand in
                    self?.relay(value)
                    return .none
                // 40. complete publisher with the completion event
            }, receiveCompletion: { [weak self] in
                    self?.complete($0)
            })
            // finish off the definition of this method
            upstream.subscribe(sink)
        }
```

> **Note**: You could initially request .max(self.capacity) and receive just that, but remember that Combine is demand-driven! If you don’t request as many values as the publisher is capable of producing, you may never get a completion event!



#### Adding a convenience operator

- 체이닝을 하기 위한 코드 추가

```swift
extension Publisher {
    func shareReplay(capacity: Int = .max) -> Publishers.ShareReplay<Self> {
        return Publishers.ShareReplay(upstream: self, capacity: capacity)
    }
}
```

#### Testing your subscription

```swift

// 41
var logger = TimeLogger(sinceOrigin: true)
// 42. 다른 시간에 값을 보내는것을 시뮬레이션하기 위한 subject
let subject = PassthroughSubject<Int,Never>()
// 43. subject를 share하고 2개의 값만 replay하도록.
let publisher = subject.shareReplay(capacity: 2)
// 44. 초기값
subject.send(0)

let subscription1 = publisher.sink( receiveCompletion: {
    print("subscription2 completed: \($0)", to: &logger)
  },
  receiveValue: {
    print("subscription2 received \($0)", to: &logger)
  }
)
subject.send(1)
subject.send(2)
subject.send(3)

let subscription2 = publisher.sink( receiveCompletion: {
    print("subscription2 completed: \($0)", to: &logger)
  },
  receiveValue: {
    print("subscription2 received \($0)", to: &logger)
  }
)
subject.send(4)
subject.send(5)
subject.send(completion: .finished)

var subscription3: Cancellable? = nil
DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
    print("Subscribing to shareReplay after upstream completed")
    subscription3 = publisher.sink(receiveCompletion: {
      print("subscription3 completed: \($0)", to: &logger)
    },
    receiveValue: {
      print("subscription3 received \($0)", to: &logger)
    }
)
}
```



#### Verifying your subsription

..skip..



## Handling backpressure

- 유체역학(?)에서 역압(backpressure)은 파이프를 통해 흐르는 유체에 저항하는 힘입니다.
- Combine에선 publisher로부터 나온 value의 흐름에 저항을 의미한다.(?)
- 여기서 저항은 무엇인가? 가끔 subscriber가 publisher로부터 방출된 값을 처리해야하는 경우가 있다.
  - 센서로부터 들어오는 input data. (Processing high-frequency data, like input from sensors.)
  - 대용량 파일 전송
  - 복잡한 UI 업데이트
  - user input을 기다림
  - 더 일반적인 경우, 들어오는 데이터가 subscriber가 keep up with 할 수 없는 데이터인 경우,
    - More generally, processing incoming data that the subscriber can’t keep up with at the rate it’s coming in.

- Combine에서 제공하는 publisher-subscriber 메커니즘은 flxible합니다. **push** ㄱㅏ 아닌 **pull**ㅁㅔ커니즘을 사용합니다. subscriber는 publisher에게 값의 방출을 요구하고, 얼마나 수신할지를 수신할지를 결정합니다.
- 이 요청 메커니즘은 adaptive합니다. demand는 subscriber가 값을 받을 때 마다 업데이트 됩니다. 이를 통해 subscriber가 데이터를 더 이상 수신하지 않으려는 경우`closing the tap`을 통해 backpressure를 다룰 수 있게 해준다.  나중에 더 많은 데이터를 수신하기 위해서 준비가 되었을 때 "opening it" 된다.

> Note: demand는 추가하는 방법으로만 조절할 수 있다. (이전의 2장에서 살펴봤던 demand를 조절하는 방법을 상기해보자.) 새로운 `.max(N)` 나 `.unlimited` 를 반환해서 subscriber가 새로운 값을 받을 때마다 demand를 증가시킬 수 있다. 또는 `.none` 을 리턴해서 demand가 증가하지 않아야 한다고 명시할 수 도 있다.
>
> 그러나 subscriber가 새로운 max demand까지의 value를 받기 위해 "on the hook" 상태에 있습니다.(?)
>
> 예를 들어, 만약에 이전의 max demand가 3 value를 받고, subscriber가 값을 하나만 받고 .none을 반환했어도 "close the tap" 되지 않을 것이다. subscriber는 publisher가 value를 방출한 준비가되면 여전히 최대 2개의 값을 더 받을 수 있다.

- 위를 해결하기 위한 3가지 방법
  - 요구를 통해 흐름을 제어하는거
  - Buffer로 담아두기
  - Drop해버리는거
- subscription 통해 처리, publisher를 