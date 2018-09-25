---
layout: post
title: 공변성과 반공변성 그리고 불변성
tags: [swift, covariance, contravariance]
---
리스코프 치환 원칙에서 공변성(covariance) 과 반공변성(contravariance)과 연관된 내용이 나온다.  
이번 글에서는 공변성, 반공변성, 불변성에 관하여 swift코드를 통해서 다뤄본다.  
  
### 뜻
공변성과 반공변성은 상대적인 성질을 갖고 기준에 따라 달라진다.  
B가 A를 상속받고, C가 B를 상속받았다면, B를 기준으로  
C는 B의 하위타입으로 **공변성**이다.  
B는 A의 하위타입으로 **반공변성**이다.  
class를 기준으로 subtype이면 공변성, supertype이면 반공변성 으로 이해하면 편하다.  
다르게 표현하면, B Type을 파라미터로 받는 함수가 있다면, C Type은 옳은 파라미터지만, A Type은 옳지 않은 파라미터다.  
제네릭, 클로저도 적용이 가능한데, 예를 들면 `Array<B>`는 `Array<A>`의 서브타입인가 `(B) -> Void`는 `(A) -> Void`의 서브타입인가?  

### 예제
만약 A타입으로 선언된 변수에 B타입의 인스턴스를  
명시적인 타입캐스팅 없이 할당할 수 있다면 B타입은 A타입의 서브타입이라고 할 수있겠다.  
#### 클래스
{% highlight swift %}
class Animal {}

class Cat: Animal {}

class Dog: Animal {}
class Maltese: Dog {}
{% endhighlight %}
위와 같은 클래스가 있다면,  
`let animal: Animal = Cat()`나  
`let animal: Animal = Dog()`은 허용되는 표현이다.  
  
#### 클로저
`var printDog: (Dog) -> Void` 클로저에 할당 가능한 타입은?  
우선 Cat은 Dog과 슈퍼타입만 갖을 뿐 연관이 없다.  
`(Animal) -> Void`의 클로저를 대입한다면?  
printDog에서 Dog만의 메서드를 사용할 수 있다.  따라서 Animal은 불가능하다.  
`(Maltese) -> Void`의 경우 
Martese는 Dog의 서브타입으로 Dog만의 메서드를 사용할 수 있다. 따라서 할당이 가능하다.  
  
`var returnDog: (Void) -> Dog`의 경우를 보면,  
`(void) -> Animal` 모든 Dog는 Animal의 서브타입이다. 따라서 할당이 가능하다.  
`(void) -> Maltese` 모든 Dog은 Maltese이 아니다.  
만약 Dog을 상속받은 Pomeranian를 돌려주는 클로저였다면 Maltese로는 커버가 불가능하다.

두가지를 합쳐본다면, `var someDog: (Dog) -> Dog`에 `(Martese) -> Animal` 를 할당이 가능하다.  
파라미터는 반공변적으로, 리턴 타입은 공변적이다.  

#### 제네릭
{% highlight swift %}
class Wrapper<T> {
  func setT(value: T)
}
{% endhighlight %}
`let wrapper: Wrapper<Any> = Wrapper<Int>()` 이러한 표현이 불가능 하다.  
프로토콜이나 제네릭은 공변성을 지원하지 않은 **불변성** 이다.  
하지만 기본으로 지원해주는 Array같은 컬렉션 타입은 공변성을 지원한다.  
`let array: Array<Any> = Array<Int>()` 가 허용된다.  
허용되는 이유를 추측 해보자면, Struct type 으로 래퍼런스 타입이 아닌 값을 복사한다.  
{% highlight swift %}
let dogArray = Array<>()
dogArray.append(Maltese())
let animalArray = dogArray
animalArray.append(Cat())

// dogArray는 Maltese
// AnimalArray는 Maltest, Cat
{% endhighlight %}
만약 배열의 래퍼런스가 복사되는 언어였다면, animalArray에 cat을 넣는 순간 dogArray에도 cat이 생기게 되기 때문에 문제가 생긴다.  
하지만 배열의 래퍼런스가 아닌 값이 복사되는 형태에서는 animalArray에 cat을 넣어도 dogArray와는 무관하다.  
따라서 안정성이 보장되기 때문이라고 추측한다.  
  
### 참조
[공변성과 반공변성은 무엇인가?(번역)](https://www.haruair.com/blog/4458)  
[공변성과 반공변성은 무엇인가?(원문)](https://www.stephanboyer.com/post/132/what-are-covariance-and-contravariance)  
[Covariance and Contravariance in Swift](https://medium.com/@aunnnn/covariance-and-contravariance-in-swift-32f3be8610b9)  

