# 19. Testing

- **ColorCalc** 라는 앱 예제를 통해 테스트를 배워보자.

### Getting Started

- 앱 간단 설명

  - hex 값을 입력하면 이 앱은 red, green, blue, opacity(alpha) 값을 알려준다.
  - 그리고 해당하는 컬러로 background color를 변경해준다. (매칭되는 컬러가 있다면)
  - 만약에 입력한 컬러가 없다면 흰색으로 배경이 변한다.

- 앱에 있는 이슈

  #### 이슈 1

  1. Action: Launch the app.
  2. Expected: 이름 label이 **aqua** 로 보여야 한다.
  3. Actual: 이름이 Label이 **Optional(ColorCalc.ColoName...)**으로 보인다.

  #### 이슈 2

  1. Action: `←` 버튼을 누름
  2. Expected: 보여지는 헥스값의 마지막 글자가 지워져야 한다.
  3. Actual: 마지막 두 개의 글자가 지워져버린다.

  #### 이슈 3

  1. Action: `←` 버튼을 누름
  2. Expected: 백그라운드가 흰색으로 변해야 한다.
  3. Actual: 백그라운드가 red로 변한다.

  #### 이슈 4

  1. Action: `⊗`버튼을 누름
  2. Expected: 헥스가 `#`으로 클리어되어야함
  3. Actual: 헥스 값이 변하지 않는다.

  #### 이슈 5

  1. Action: 006636 값 입력
  2. Expected: 0, 102, 54, 255 보여져야됨.
  3. Actual: 0, 62, 32, 155 보여짐.



- 몇 가지 Combine operator에 대한 테스트를 할 것이다.

> 이번 챕터는 iOS 유닛테스트에 익숙하다고 가정한다. 



### Testing Combine operators

- **Given** : 조건(condition)
- **When** : 수행할 동작 (an action in performed)
- **Then** : 일어나길 기대하는 결과 (an expected result occurs)
- **ColorCalcTests/CombineOperatorsTests.swift**. 파일을 열자.

```swift
import XCTest
import Combine

class CombineOperatorsTests: XCTestCase {
    
    var subscriptions = Set<AnyCancellable>()
    
    override func tearDown() {
        subscriptions = []
    }
}
```

### Testing collect()

- `collect` 오퍼레이터 테스트
  - This operator will buffer values emiited by an upstream publisher.(complete할 때 까지 기다린다.)

```swift
class CombineOperatorsTests: XCTestCase {
    
    var subscriptions = Set<AnyCancellable>()
    
    override func tearDown() {
        subscriptions = []
    }
    
    func test_collect() {
        // Given
        let values = [0, 1, 2]
        let publisher = values.publisher
        
        // When
        publisher
        .collect()
            .sink(receiveValue: {
                // Then
                XCTAssert($0 == values,
                          "Result was expected to be \(values) but was \($0)")
            })
        .store(in: &subscriptions)
    }
}
```

- 테스트 팁
  1. 다이아몬드 클릭 -> 클래스 내 메서드 single test
  2. **command - U** : 프로젝트 내의 모든 타겟안에 있는 테스트 (각 테스트 타겟은 여러 테스트 클래스를 포함하고 있을 수도 있다. = containing multiple tests)
  3. **Product > Perform Action > Run "TestClassName"** 메뉴 사용 > 키보드단축키: (**Command-Control-Option-U**)
- 테스트 결과 뜸

```
2020-07-20 01:24:35.775976+0900 ColorCalc[15370:453227] Launching with XCTest injected. Preparing to run tests.
2020-07-20 01:24:35.902148+0900 ColorCalc[15370:453227] Waiting to run tests until the app finishes launching.
Test Suite 'Selected tests' started at 2020-07-20 01:24:36.169
Test Suite 'ColorCalcTests.xctest' started at 2020-07-20 01:24:36.169
Test Suite 'CombineOperatorsTests' started at 2020-07-20 01:24:36.170
Test Case '-[ColorCalcTests.CombineOperatorsTests test_collect]' started.
Test Case '-[ColorCalcTests.CombineOperatorsTests test_collect]' passed (0.002 seconds).
Test Suite 'CombineOperatorsTests' passed at 2020-07-20 01:24:36.173.
	 Executed 1 test, with 0 failures (0 unexpected) in 0.002 (0.003) seconds
Test Suite 'ColorCalcTests.xctest' passed at 2020-07-20 01:24:36.173.
	 Executed 1 test, with 0 failures (0 unexpected) in 0.002 (0.004) seconds
Test Suite 'Selected tests' passed at 2020-07-20 01:24:36.173.
	 Executed 1 test, with 0 failures (0 unexpected) in 0.002 (0.005) seconds
```



- Then 코드 바꿔서 테스트 실패 내기

```swi
// Then
XCTAssert($0 == values + [1],
"Result was expected to be \(values + [1]) but was \($0)")
```

- 실패 테스트 결과

```
2020-07-20 01:28:55.721757+0900 ColorCalc[15412:459180] Launching with XCTest injected. Preparing to run tests.
2020-07-20 01:28:55.834542+0900 ColorCalc[15412:459180] Waiting to run tests until the app finishes launching.
Test Suite 'Selected tests' started at 2020-07-20 01:28:56.025
Test Suite 'ColorCalcTests.xctest' started at 2020-07-20 01:28:56.025
Test Suite 'CombineOperatorsTests' started at 2020-07-20 01:28:56.026
Test Case '-[ColorCalcTests.CombineOperatorsTests test_collect]' started.
/Users/mason/Documents/Combine_Asynchronous_Programming_with_Swift_v1.0.3/19-testing-combine-code/starter/ColorCalcTests/CombineOperatorsTests.swift:50: error: -[ColorCalcTests.CombineOperatorsTests test_collect] : XCTAssertTrue failed - Result was expected to be [0, 1, 2, 1] but was [0, 1, 2]
Test Case '-[ColorCalcTests.CombineOperatorsTests test_collect]' failed (0.002 seconds).
Test Suite 'CombineOperatorsTests' failed at 2020-07-20 01:28:56.028.
	 Executed 1 test, with 1 failure (0 unexpected) in 0.002 (0.002) seconds
Test Suite 'ColorCalcTests.xctest' failed at 2020-07-20 01:28:56.028.
	 Executed 1 test, with 1 failure (0 unexpected) in 0.002 (0.003) seconds
Test Suite 'Selected tests' failed at 2020-07-20 01:28:56.028.
	 Executed 1 test, with 1 failure (0 unexpected) in 0.002 (0.003) seconds

```



### Testing flatMap(maxPublishers:)



