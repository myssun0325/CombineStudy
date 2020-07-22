# Ch 11: Timers



- Dispatch 프레임워크가 사용가능하기 전에 비동기 작업을 하고 Concurrent를 구현하기 위해 개발자들은 `RunLoop`에 의존했었다.
- `Timer`는 반복/비반복 타이머를 생성하기 위해 사용된다. 이후에 `Dispatch`가 나오고나서 `DispatchSourceTimer`로 이를 구현했다.





### Using RunLoop

- Thread클래스를 사용해서 만든 메인스레드나 다른 스레드는 자체적으로 RunLoop를 가질 수 있다. 현재 스레드에서 `RunLoop.current`를 호출하면 Foundation 자동으로 RunLoop를 생성해준다. 
- RunLoop를 명시적으로 사용하는 경우는 직접 스레드를 정의해서 사용하는 경우 뿐이며, 
- run loop가 어떻게 동작하는지 이해가 없다면 주의해라.(특히 RunLoop를 실행하는 loop문이 있는 경우)
- 런루프에의 작동방식에 대한 이해가 없다면 앱의 메인스레드를 실행하는 기본 main RunLoop를 사용하는게 낫다.

> Note: `RunLoop`클래스는 스레드세이프지 않는다. 그러므로 현재 스레드의 실행 루프(loop) - (run loop)에 대해서만 `RunLoop`의 메서드를 호출해라.

- RunLoop는 17장에서 배우게 될 `Scheduler`프로토콜을 구현한다. 스케쥴러는 상대적으로 low-level의 메서드들을 정의하고 있고, 취소가능한 타이머를(cancellable timers) 생성하는 유일한 방법을 정의한다.
- 아래 타이머는 어떤 value도 주지않고 publisher를 생성하지도 않는다. 

```swift
  let runLoop = RunLoop.main
  let subscription = runLoop.schedule(
  	after: runLoop.now, // start date
  	interval: .seconds(1),
	  tolerance: .milliseconds(100) // 허용오차(?)
  ){
    print("Timer fired")
  }
```

- Combine과 관련해서는 아래와 같이 쓸 때만 유용하게 쓸 수 있다.

```swift
runLoop.schedule(after: .init(Date(timeIntervalSinceNow: 3.0)))
{
	cancellable.cancel()
}
```

- 모든 것들을 고려했을 때, RunLoop는 타이머를 생성하는 best way는 아닌다. 



### Using the Timer class

- Timer는 가장 오래된 타이머이지만, 델리게이트 패턴과 `RunLoop`과의 밀접한 관계 때문에 사용이 까다로웠다.
- Combine은 이를 더 간단하게 사용할 수 있도록 해준다.
- 아래와 같이 사용할 수 있다. 두개의 파라미터는 다음과 같다.
  - 타이머가 붙을(attach) RunLoop (예제에서는 main)
  - run loop mode, 예제에서는 default
  - run loop가 어떻게 동작하는지 모른다면 default value를 쓰는게 좋다.

```swift
let publisher = Timer.publish(every: 1.0, on: .main, in: .common)
```

> Note: DispatchQueue.main 이외의 Dispatch큐에서 이 코드를 실행하게 되면 예기치 않은 결과가 발생할 수 있다. Dispatch 프레임워크는 run loop를 사용하지 않고 스레드를 관리합니다. run loop는 이벤트를 처리하기 위해 run loop의 실행 메서드(run methods) 중에 하나를 호출해야하므로 메인큐 외에서 타이머가 실행되는것을 볼 수 없다.

- timer를 pubilhser하게되면 `ConnectPublisher`를 반환하게 된다. connect() 메서드를 호출하기전까지 타이머는 시작되지 않는다. `autoconnect()`를 쓰게되면 첫번째 subscriber가 구독하자마자 타이머가 시작된다.

```swift
// 1초마다 타이머 반복
let subscription = Timer
    .publish(every: 1.0, on: .main, in: .common)
    .autoconnect()
    .scan(0) { counter, _ in counter + 1 }
    .sink { counter in
        print("Counter is \(counter)")
    }
```



### Using DispatchQueue

- Dispatch 프레임워크에는 `DispatchQueueTimerSource` 이벤트 소스를 갖고 있는데도 Combine은 타이머 인터페이스를 제공하지 않는다. 대신 다른 방법을 사용하여 큐에 타이머 이벤트를 생성한다.

```swift
let queue = DispatchQueue.main 

// 1
let source = PassthroughSubject<Int, Never>()

// 2
var counter = 0

// 3
let cancellable = queue.schedule( after: queue.now,
                                 interval: .seconds(1)) { 
  source.send(counter)
  counter += 1
}

// 4
let subscription = source.sink {
  print("Timer emitted \($0)")
}
```

1. timer value를 보낼 Subject를 생성
2. 타이머가 fire될 때마다 증가시킬 counter
3. 선택된 큐에서 매 초마다 반복되는 작업을 예약합니다. 작업은 바로 시작됩니다.
4. subject를 구독해서 매초마다 받아옵니다.

- As you can see, this is not pretty. It would help to move this code to a function and pass both the interval and the start time.
  - 코드가 별로 이쁘지 않다. 코드를 함수로 옮기고 Interval과 시작시간을 모두 파라미터로 전달하는게 좋다.



### Key points

- 지정된 RunLoop에서 지정된 interval로 값을 생성하는 publisher를 얻으려면 `Timer.publish`를 사용해라.
- Use DispatchQueue.schedule for modern timers emitting events on a dispatchqueue.