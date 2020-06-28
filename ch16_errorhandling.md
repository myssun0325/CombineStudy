# Ch 16: Error Handling



### Never



### setFailureType

- infallible publisher -> fallible 로 바꿔준다.



### assertNoFailure

- 개발이나 publisher가 failure event로 finish를 할 수 없다는 것을 확인할 때 유용하다.
- upstream으로부터 failure가 흐르는것을 막을 수는 없다. error를 내면 fatalError를 내게 됨.

