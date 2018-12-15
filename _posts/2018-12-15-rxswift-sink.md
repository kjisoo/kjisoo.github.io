---
layout: post
title: RxSwift Observable transform 내부동작
tags: [ios, rxswift, sink, producer, observer, observable]
---
RxSwift를 사용하면서 많이 쓰게되는 filter, map등의 함수들의 내부 동작들을 알아봅니다.  
Observable, Observer 개념에 대해 이해하고 있어야 하며,  
ObservableConvertibleType, ObservableType, Observable, Producer, Sink 에 대해 간략하게 살펴봅니다.  
  
{% highlight swift %}
Observable.just(3)
  .filter { $0 % 2 != 0 }
  .map { $0 * 100 }
  .subscribe()
{% endhighlight %}
위 코드의 내부 동작을 아는것이 본 글의 목표입니다.  
  
#### ObservableConvertibleType
ObservableConvertibleType는 `func asObservable() -> Observable<E>`만을 포함하고 있는 단순한 프로토콜 입니다.  
  
#### ObservableType
ObservableType는 `ObservableConvertibleType`를 상속받고 `func subscribe<O: ObserverType>(_ observer: O) -> Disposable where O.E == E` 를 추가한 프로토콜입니다.  
ObservableType부터 subscribee를 통해 이벤트를 구독할 수 있습니다.  
  
#### Observable
Observable는 ObservableType를 따르고 있는 구체클래스 입니다.  
소스코드를 보면 `subscribe`의 내부가 비어있는데, 이는 Observable를 직접 생성하여 subscribe할 수 없기 때문입니다.  
`asObservable()`에 대해서 return self 로 기본 구현이있습니다.  
  
#### Producer
Producer는 Observable를 상속받는 클래스입니다.  
`subscribe`가 구현되어 있으며 filter, map 등을 사용하면 실제 생성되는 타입입니다.  
filter 는 `class Filter<Element> : Producer<Element>` 이지만 Producer는 Observable의 서브타입이기 때문에 Observable<E> 형식을 유지할 수 있습니다.

또 `func run<O : ObserverType>(_ observer: O, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where O.E == Element`가 빈 메서드로 추가되어 있습니다.  
위 Filter class는 run 메서드도 구현되어 있습니다.  
Producer의 subscribe 가 중요하기 때문에 코드를 살펴보면,  
{% highlight swift %}
// 이해를 위해 일부 코드를 제거했습니다.
override func subscribe<O : ObserverType>(_ observer: O) -> Disposable where O.E == Element {
  let disposer = SinkDisposer() // 1
  let sinkAndSubscription = self.run(observer, cancel: disposer) // 2
  disposer.setSinkAndSubscription(sink: sinkAndSubscription.sink, subscription: sinkAndSubscription.subscription) // 6
  
  return disposer // 7
}
// Filter의 run
override func run<O: ObserverType>(_ observer: O, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where O.E == Element {
  let sink = FilterSink(predicate: _predicate, observer: observer, cancel: cancel) // 3
  let subscription = _source.subscribe(sink) // 4
  return (sink: sink, subscription: subscription) // 5
}
{% endhighlight %}
1. Observable를 subscribe 할때 받는 그 disposable입니다. dispose될때 여러 transform operator 들이 연쇄적으로 disposed됩니다.  
2. subscribe시 받는 Observer, 1에서 생성한 disposable입니다. (subscribe(), onNext등을 이용시 익명 observer가 생성되어 전달됩니다.)   
3. FilterSink를 생성합니다. FilterSink는 Disposable를 따르는 Sink와 ObserverType 2개의 프로토콜을 따르고 있습니다. _predicate는 filter의 클로저, observer, cancel는 2에서 준 observer, disposable입니다.  
4. _source는 filter 이전 Observable 인데, Observable.just(1).filter 의 경우 just(1) 이 _source에 해당합니다. sink를 ObserverType으로서 상위 Observable의 subscribe를 일으킵니다. **따라서 생성은 순차적으로, 구독은 역순으로 진행되게 됩니다.**  
5. sink는 자신의 Sink를 구독하기 위해서 나온 Disposable, subscription은 이전 Observable을 구독하면서 나온 Disposable입니다.  
6. disposBag 으로 이해하면 되겠습니다.  
7. 최종적으로 subscribe 시 받게되는 disposable입니다.  
Producer 에서는 subscribe를 발생되면 run 을 통해서 Sink를 생성하는데, 생성된 Sink의 Observer는 이전 Observable의 subscribe로 전달한다고 이해하면 되겠습니다.   
  
#### Sink
Sink는 Disposable을 따르는 클래스입니다.  
생성자로 Observer를 받고 observer.on 의 역할을 `func forwardOn(_ event: Event<O.E>`에서 수행합니다.  
해당 메서드 내에서는 disposed가 되지 않았다면, _observer.on(event) 전달하는 역할만 있습니다.  
추상화를 하기 위한 클래스라 생각듭니다.  
  
이해에 필요한 내용은 모두 나왔습니다. 이제부터 처음 나왔던 흐름을 따라가볼 차례입니다.  
#### Observable.just(3)
{% highlight swift %}
extension ObservableType {
  public static func just(_ element: E) -> Observable<E> {
    return Just(element: element)
  }
}
{% endhighlight %}
just, map 등 ObservableType을 extension 하여 구현되어 있습니다. just는 static입니다.  
Just의 구체클래스를 숨기고 Observable타입을 유지하는 영리한 방법이라 생각합니다.  
내부에선 받은 element를 Just의 생성에 사용하고, Just를 돌려줍니다.  
  
{% highlight swift %}
class Just<Element> : Producer<Element> {
  private let _element: Element
  
  init(element: Element) {
    _element = element
  }
  
  override func subscribe<O : ObserverType>(_ observer: O) -> Disposable where O.E == Element {
    observer.on(.next(_element))
    observer.on(.completed)
    return Disposables.create()
  }
}
{% endhighlight %}
Just는 Producer로 받은 element를 저정하고 있다가 subscribe가 일어나면 받은 element를 observer에게 넘겨주고 끝이 납니다.  
  
#### .filter { %0 % 2 != 0}
filter 도 마찬가지로, extension을 통해 구현되어 있습니다.  
{% highlight swift %}
extension ObservableType {
  public func filter(_ predicate: @escaping (E) throws -> Bool) -> Observable<E> {
      return Filter(source: asObservable() /* point */, predicate: predicate)
  }
}
{% endhighlight %}
위에서 포인트는 asObservable()입니다. 해당 Observable은 just부터 내려왔기 때문에 asObservable()는 곧 Observable.just(3)입니다.  
predicate는 { $0 % 2 != 0 } 의 클로저입니다.  
  
Filter를 보면,  
{% highlight swift %}
class Filter<Element> : Producer<Element> {
  typealias Predicate = (Element) throws -> Bool
  
  private let _source: Observable<Element>
  private let _predicate: Predicate
  
  init(source: Observable<Element>, predicate: @escaping Predicate) {
    _source = source
    _predicate = predicate
  }
  
  override func run<O: ObserverType>(_ observer: O, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where O.E == Element {
    let sink = FilterSink(predicate: _predicate, observer: observer, cancel: cancel)
    let subscription = _source.subscribe(sink)
    return (sink: sink, subscription: subscription)
  }
}
{% endhighlight %}
생성시에는 Observable과 클로저를 source, predicate에 저장을 해두고, 실제 구독이 일어나야 run 내에서 Sink를 생성합니다.  
FilterSink로 클로저, observer 를 전달하는데 observer는 subscribe에서 생성될 observer입니다.  
source(Observable.just(3))을 sink를 이용해 subscribe 합니다.  
  
{% highlight swift %}
class FilterSink<O : ObserverType>: Sink<O>, ObserverType {
  typealias Predicate = (Element) throws -> Bool
  typealias Element = O.E
  
  private let _predicate: Predicate
  
  init(predicate: @escaping Predicate, observer: O, cancel: Cancelable) {
    _predicate = predicate
    super.init(observer: observer, cancel: cancel)
  }
  // ObserverType 타입
  func on(_ event: Event<Element>) {
    switch event {
    // Point
    case .next(let value):
      do {
        let satisfies = try _predicate(value)
        if satisfies {
          forwardOn(.next(value))
        }
      }
      catch let e {
        forwardOn(.error(e))
        dispose()
      }
    case .completed, .error:
      forwardOn(event)
      dispose()
    }
  }
}
{% endhighlight %}
.next에서 받은 클로저를 이용해 next하거나 스킵합니다.  
  
#### map { $0 & 100 }
map 은 filter 와 유사하게 되어있습니다.  
Map Sink에서 filter는 next를 하거나 안했지만, map에서는 클로저의 결과를 next로 전달합니다.  
{% highlight swift %}
func on(_ event: Event<SourceType>) {
  switch event {
  case .next(let element):
    do {
      // _transform 이 map에 전달된 클로저
      let mappedElement = try _transform(element)
      forwardOn(.next(mappedElement))
    }
    catch let e {
      forwardOn(.error(e))
      dispose()
    }
  case .error(let error):
    forwardOn(.error(error))
    dispose()
  case .completed:
    forwardOn(.completed)
    dispose()
  }
}
{% endhighlight %}

#### subscribe()
최종적으로 subscribe가 일어나야 run이 호출됩니다.  
let a = Observable.just(3)  
let b = a.filter { $0 % 2 != 0 }  
let c = b.map { $0 * 100 }  
let d = c.subscribe()  
생성은 a, b, c, d순으로 생성됩니다.  
구독은 c, b, a순으로 일어나게 됩니다.  
c.subscribe() 에서 observer 가 없으므로 AnonymousObserver 가 생성되어 c에게 전달되고,  
c는 MapSink를 통해 만들어진 Observer를 이용해 b를 구독하고,  
b는 FilterSink를 통해 만들어진 Observer를 이용해 a를 구독하고,  
a는 저장된 element를 받은 observer를 통해 전달하고 스트림을 종료(completed)시킵니다.  
  
아마 이 글을 통해서 한번에 이해하는건 불가능에 가까울거라 생각합니다.  
하지만 RxSwift의 코드를 볼 예정이라면 도움이 되었으면 좋겠습니다.  
Rxswift의 코드를 보면 Protocol과 Extension을 잘 쓰고  
접근제한자를 라이브러리를 안전하게 사용할 수 있게 적절하게 사용되어 있어 보기 좋았습니다.  
