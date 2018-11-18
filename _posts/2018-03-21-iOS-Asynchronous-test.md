---
layout: post
title: iOS 비동기 코드에 대한 테스트 방법
tags: [iOS, Swift, RxSwift, test, RxTest, RxBlocking, XCTest, Quick, Nimble]
redirect_from:
  - /blog/ios-비동기-코드에-대한-테스트-방법들
  - /blog/ios-비동기-코드에-대한-테스트-방법들/
---
비동기(Asynchronous)를 테스트 하기 위한 방법들을 정리해봅니다.  
[XCTestCase](https://developer.apple.com/documentation/xctest/xctestcase), [Quick](https://github.com/Quick/Quick/) & [Nimble](https://github.com/Quick/Nimble/), Rx를 다룰것이며, 더 좋은 테스트 방법들이 많이 공개되어있으니, 참고용으로 보면 좋겠습니다.  
해당 게시글의 프로젝트는 [https://github.com/kjisoo/ios_async_testcase](https://github.com/kjisoo/ios_async_testcase)에서 다운받을 수 있습니다.  

### Closure
아래는 테스트할 서비스 코드 입니다.  
{% highlight swift %}
class FooService {
  private var workItem: DispatchWorkItem?

  /// Complete is executed after a delay.
  ///
  /// Complete is executed after a delay.
  /// When executed, param is passed to complete.
  /// If you call this method, the previous call is automatically canceled.
  ///
  /// - Parameters:
  ///     - param: param to be passed to complete
  ///     - delay: delay of complete execution
  ///     - complete: Closure to be executed after delay
  ///
  public func execute(param: Any, delay: Double, complete: @escaping ((Any) -> Void)) {
    self.cancel()
    workItem = DispatchWorkItem(block: {
      complete(param)
    })
    DispatchQueue.main.asyncAfter(deadline: .now() + delay, execute: workItem!)
  }
  
  /// Cancels the last execute.
  ///
  /// Cancels the last execute.
  /// If there is no last execute, nothing happens.
  ///
  public func cancel() {
    workItem?.cancel()
  }
}
{% endhighlight %}
execute로 파라미터와 딜레이, 완료 클로저를 넘겨주면, 딜레이 후에 클로저에 파라미터를 넘겨줍니다.  
실행 전에 execute를 다시 호출하거나, cancel를 호출할경우 이전 요청을 취소합니다.  
테스트할 항목들은,  
 - 파라미터를 클로저에 잘 넘겨주는지
 - 클로저가 실행이 되는지
 - 클로저가 실행되기 전에 요청을 취소했을때 클로저가 실행이 안되는지
 - 클로저가 실행되기 전에 다시 요청을 했을 경우 이전 클로저는 무시되고, 새로운 클로저가 실행되는지
execute에 실행이 취소되었을때 받을 수 있는 피드백이 없기 때문에 테스트가 어려울 수 있습니다.

#### 1. XCTestCase - XCTestExpectation
클로저 실행 확인  
{% highlight swift %}
func testPassParam() {
  let executeExpectation = XCTestExpectation(description: "in closure")
  let text = "Test param string"
  fooService.execute(param: text, delay: 1) { (param) in
    executeExpectation.fulfill()
    XCTAssert(text == (param as! String))
  }
  wait(for: [executeExpectation], timeout: 2)
}
{% endhighlight %}
XCTestExpectation를 사용해 인스턴스를 만들고, 클로저 내부에서 fulfill를 호출해줍니다.  
타임아웃 이내에 fulfill이 채워지지 않는다면, 해당 테스트 케이스는 실패하게 됩니다.  
wait에는 enforceOrder파라미터가 있는데 해당 값이 true면 expectations가 순서에 맞게 fulfill되어야 합니다.  

##### 클로저가 실행되지 않는게 올바른 동작일때
{% highlight swift %}
func testCancelExecuteClosure() {
  let executeExpectation = XCTestExpectation(description: "in closure")
  executeExpectation.isInverted = true
  fooService.execute(param: "param", delay: 1) { (_) in
    executeExpectation.fulfill() // test fail
  }
  fooService.cancel()
  wait(for: [executeExpectation], timeout: 2)
}
{% endhighlight %}
클로저가 실행되지 않는 상황이 되었을때 피드백이 있으면 좋겠지만, 그렇지 않은 경우들도 있습니다.  
해당 케이스들도 테스트를 할 수 있는데, XCTestExpectation에 isInverted옵션을 true로 설정해주면 됩니다.  
해당 값이 true면 동작이 반대로 되는데, 타임아웃 시간내에 fulfill이 있으면 해당 테스트케이스는 실패하게 됩니다.  

##### 위 두 케이스를 혼합
{% highlight swift %}
func testDoubleExecuteClosure() {
  let executeExpectation = XCTestExpectation(description: "in closure")
  let notExecuteExpectation = XCTestExpectation(description: "never execute")
  notExecuteExpectation.isInverted = true
  fooService.execute(param: "param", delay: 1) { (_) in
    notExecuteExpectation.fulfill()
    XCTFail() // 중복
  }
  fooService.execute(param: "param", delay: 1) { (_) in
    executeExpectation.fulfill()
  }
  wait(for: [executeExpectation, notExecuteExpectation], timeout: 2)
}
{% endhighlight %}
2개의 XCTestExpectation를 만들고, 실행되면 안되는 expectation에는 isInverted속성을 true로 줍니다.  

#### 2. Quick & Nimble
##### 클로저 실행 확인 - waitUntil
{% highlight swift %}
it("Compares parameters equally using waitUntil.") {
  let text = "Test param string"
  waitUntil(timeout: 2) { done in
    fooService.execute(param: text, delay: 1, complete: { (param) in
      expect((param as! String)).to(equal(text))
      done()
    })
  }
}
{% endhighlight %}
waitUntil를 사용할 수 있습니다.  
위 케이스와 마찬가지로, 시간내에 done을 호출해야합니다. 호출하지 않을경우 해당 케이스는 실패하게 됩니다.  

##### 클로저 실행 확인 - toEventually
{% highlight swift %}
it("Compares parameters equally using toEventually.") {
  let text = "Test param string"
  var inClosureText: String = ""
  fooService.execute(param: text, delay: 1, complete: { (param) in
    inClosureText = param as! String
  })
  expect(inClosureText).toEventually(equal(text), timeout: 2)
}
{% endhighlight %}
 waitUntil외에서 toEventually라는 기능을 제공해줍니다.  
타임아웃시간 동안 일정한 간격으로 값을 확인하여 값을 비교합니다. 예를들어, 0.1초 간격으로 두 값이 같은지 확인하고 같아지는 순간 해당 케이스는 성공하게 됩니다.  
제한 시간동안 값이 여러번 바뀔 수 있다면 해당 기능의 사용을 자제해야 합니다.  
최초의 상태가 A이고 기대하는 상태가 B 일때,  A --> B --> A 와 같이 상태가 변한다면, B 상태가 되었을때 해당 테스트는 성공하게 됩니다.  

실행되지 않는 케이스에 대해서, isInverted처럼 편하게 사용할 수 있는 기능을 찾지 못했습니다.  


#### Rx
아래는 Rx에서 테스트할 코드입니다.
{% highlight swift %}
class RxFooService {
  private let cancelSubject = PublishSubject()
  
  /// After the delay, the param is emitted via the observer.
  ///
  /// After the delay, the param is emitted via the observer.
  /// If you call this method, the previous observer is automatically disposed.
  ///
  /// - Parameters:
  ///     - param: param to be passed to observer
  ///     - delay: delay for emitted
  /// - Returns: Observer to receive param after delay
  ///
  public func execute(param: Any, delay: Double, scheduler: SchedulerType = MainScheduler.instance) -> Observable {
    self.cancel()
    return Observable
      .create({ (observer) -> Disposable in
        observer.onNext(param)
        return Disposables.create()
      })
      .delay(delay, scheduler: scheduler)
      .takeUntil(self.cancelSubject)
  }
  
  /// Cancels the last execute.
  ///
  /// Cancels the last execute.
  /// If there is no last execute, nothing happens.
  ///
  public func cancel() {
    self.cancelSubject.onNext(Void())
  }
}
{% endhighlight %}
동일하게, 특정 시간 후에 파라미터를 옵저버를 통해 전달해줍니다.  

##### 값 얻어오기 - RxBlocking
{% highlight swift %}
func testPassParam() {
  let text = "Test param string"
  
  let firstParam = try! fooService.execute(param: text, delay: 1.0).toBlocking().first()
  
  XCTAssertEqual((firstParam as! String), text)
}
{% endhighlight %}
toBlocking를 사용해 옵저버를 동기화시켜 사용할 수 있습니다.  

##### 스트림 흐름 비교하기 - RxTest
{% highlight swift %}
func testDoubleExecuteBlock() {
  let scheduler = TestScheduler(initialClock: 0)
  let observer = scheduler.createObserver(String.self)
  
  scheduler.scheduleAt(100) {
    _ = self.fooService.execute(param: "first", delay: 10, scheduler: scheduler)
      .map { $0 as! String }
      .subscribe(observer)
  }
  scheduler.scheduleAt(105) {
    _ = self.fooService.execute(param: "second", delay: 10, scheduler: scheduler)
      .map { $0 as! String }
      .subscribe(observer)
  }
  scheduler.start()
  let expectedEvents: [Recorded<Event>] = [
    completed(105),
    next(115, "second")
  ]
  
  XCTAssertEqual(observer.events, expectedEvents)
}
{% endhighlight %}
TestScheduler를 만들고, 관찰할 옵저버를 만듭니다.  
이후 만들어지는 옵저버블들을 구독하게 되면, 해당 옵저버에 이벤트가 기록됩니다.  
가상시간 1은 1초인데, 100에 이벤트를 발생, 105에 새로운 이벤트를 발생하여 이전 스트림으 완료되어 complete(105)가 생깁니다.  
그리고  10초 후 second를 받아 next(115, "second")가 생깁니다.  
Rx를 사용하는 거의 모든 케이스를 커버가능합니다.  

Rx를 사용한다면, Rx + RxTest, RxBlock, ... 테스트를 위한 라이브러리 + XCTest or Quick  
Rx를 사용하지 않는다면, XCTest or Quick + 테스트를 위한 라이브러리 
가능하면, 같은 계열을 사용하여 일관성을 맞추는것이 좋겠습니다.  
  

끝
