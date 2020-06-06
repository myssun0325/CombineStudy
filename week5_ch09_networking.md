# Ch 9: Networking

- Combine 제공
  - URLSession
  - JSON encoding and decoding through the Codable protocol.



### URLSession extensions

- URLSession이 제공하는 operation
  - **Data transfer tasks** to retrieve the content of a URL.
  - **Download tasks** to retrieve the content of a URL and save it to a file.
  - **Upload tasks** to upload files and data to a URL.
  - **Stream tasks** to stream data between two parties.
  - **Websocket tasks** to connect to websockets.
- Combine은 URLRequest나 URL을 변형해서 dataTask를 하는 API를 제공합니다.

```swift
guard let url = URL(string: "https://mysite.com/mydata.json") else {
    return
}

let subscription = URLSession.shared
    .dataTaskPublisher(for: url) 
		.sink(receiveCompletion: { completion in
        if case .failure(let err) = completion {
            print("Retrieving data failed with error \(err)")
        }
    }, receiveValue: { data, response in
        print("Retrieved data of size \(data.count), response = \ (response)")
    })
```



### Codable support

```swift
let subscription = URLSession.shared
    .dataTaskPublisher(for: url)
    .tryMap { data, _ in
        try JSONDecoder().decode(MyType.self, from: data) }
    .sink(receiveCompletion: { completion in 
        if case .failure(let err) = completion {
        	print("Retrieving data failed with error \(err)")
        }
    }, receiveValue: { object in
        print("Retrieved object \(object)")
    })
```

- `tryMap` 복습 : 에러가 발생할 경우 downstream으로 에러를 방출한다. (ch03,04 참고)

- 위에서 `tryMap`을 아래와 같이 대치

```swift
.map(\.data)
.decode(type: MyType.self, decoder: JSONDecoder())
```

- 안타깝게도 `dataTaskPublisher`는 튜플을 방출하기 때문에, 직접 `decode(type:decoder)`를 쓸 수 없다. 그래서 `map`으로 Data 부분만 가져와야한다.
- 아래와 같이 `decode`를 사용할 때의 유일한 장점은 tryMap을 할 때 클로저에서 매번 JSONDecoder를 생성하지 않고 한번만 인스턴스화 된다는 것입니다.



### Publishing network data to multiple subscribers

- 캐싱 매커니즘을 사용하는 것 외에 또 다른 방법은 `multicast()`를 사용하는 겁니다.
- multicast는 `ConnectablePublisher`를 생성하는데 ConnectablePublisher는 Subject를 통해 값을 방출합니다. Subject를 multiple 구독하고 준비가되면 `connect`메서드를 호출합니다.

```swift
let url = URL(string: "https://www.raywenderlich.com")!
    let publisher = URLSession.shared
        // 1
        .dataTaskPublisher(for: url)
        .map(\.data)
				// 반드시 적절한 타입의 Subject 또는 이미 있는 Subject를 활용해도 된다.
        .multicast { PassthroughSubject<Data, URLError>() }
    // 2 - publisher가 ConnectablePublisher기 때문에 구독하자마자 바로 시작되지 않는다.
    let subscription1 = publisher
        .sink(receiveCompletion: { completion in
            if case .failure(let err) = completion {
                print("Sink1 Retrieving data failed with error \(err)")
            }
        }, receiveValue: { object in
            print("Sink1 Retrieved object \(object)")
        })
    // 3
    let subscription2 = publisher
        .sink(receiveCompletion: { completion in
            if case .failure(let err) = completion {
                print("Sink2 Retrieving data failed with error \(err)")
            }
        }, receiveValue: { object in
            print("Sink2 Retrieved object \(object)")
        })
    // 4 - start working, pushing values to all of its subscribers.
    let subscription = publisher.connect()
```



> **Note**: Make sure to store all of your Cancellables; otherwise, they would be deallocated and canceled when leaving the current code scope, which would be immediate in this specific case.

- 방금했던 이 부분이 Rx처럼 오퍼레이터로 제공되진 않는다. 그래서 18장에서 Custom Publisher & Handling Backpressure를 통해 솔루션을 살펴볼것임.