---
layout: post
title: swift로 하는 protocol programming
tags: [swift, protocol, interface, ios, dependency, di]
---
swift의 POP(protocol oriented programming)의 글들을 보면, 프로토콜을 통해 수평적인 확장을 지향하지만,  
이번 글에서 다룰 프로토콜 프로그래밍은 전통적 인터페이스를 설계하고 사용하는 과정을 봅니다.  
프로토콜 선언, 구현보다는 의존성에 주의를 기울이면 프로토콜의 존재 이유와 장점에 대해 더 쉽게 이해할 수 있을거라 생각됩니다.  
앞으로는 프로토콜과 인터페이스를 동일한 의미로 작성합니다.  

### 프로토콜 프로그래밍이 중요한 이유
구현이 아닌 인터페이스에 맞추어 개발하라는 말이 있듯이 그만큼 인터페이스(프로토콜)은 중요합니다.  
  
의도한 행위만을 노출 시킴으로서 그외 상태에 대한 캡슐화가 이루어 집니다.  
`func getUser(id: Int) -> User` 을 갖는 프로토콜이 있다면, 이를 구현하는 구현체는 User를 네트워크에서 가저오든, 파일에서 가저오든 상관이 없고 이를 구현하기 위해 사용되는 변수 혹은 메서드들은 노출되지 않으며 노출시킬 필요가 없습니다.  
(프로토콜이 이러한 사고를 지향할 수 있게 도와준다고 생각돼 추가했습니다.)  

구현체가 아닌 프로토콜에 의존적이면, 구현을 바꾸기가 쉬워집니다.  
위와 같이 User를 네트워크에서 가저오던 구현체를, 파일에서 가저오든 인터페이스만 충족된다면, 사용되는 곳에서는 신경쓰지 않습니다.  
혹은 더 좋은 성능으로 개발된 구현체를 만들어 기존 구현체를 대체할 수 있습니다. 심지어 런타임에도.  

추가적으로, Erich Gamma와의 대화에 객체 간 협업 증대, 공동의 어휘 등의 내용이 있습니다.  

### 주의
프로토콜의 구현을 바꾸는건 쉬운 장점이 있지만,  
프로토콜 자체를 바꾸는 일은 모든 구현체, 해당 프로토콜을 이용하는 오브젝트, 테스트 코드에 영향을 주기 때문에 어려운 작업 입니다.  
따라서 프로토콜을 설계할때 적절한 권한(역할)을 부여해야 하여 변경을 최소화 해야합니다.  
extension을 통해 부분 기본 구현으로, 조금 더 유연한 개발이 가능해보입니다.  
  
### 예제
User의 id를 받아 User의 이미지들을 돌려주는 시나리오를 설정합니다.    
{% highlight swift %}
class ImagesController {
  func getImagesSortedByTime(userID: Int) -> [Image] {
    let userService = UserService()  // 1
    let imageService = ImageService()  // 2
    let user = userService.getUser(userID: userID)  // 3
    let images = imageService.getImages(user: user)  // 4
    return images.sorted { $0.timestamp > $1.timestamp }  // 5
  }
}
{% endhighlight %}
ImagesController는 userID를 받아 해당 유저의 모든 이미지를 시간순으로 정렬하여 보여줍니다.  
1.  유저의 아이디를 이용해 유저를 얻어올 수 있는 유저 서비스를 생성합니다.  
2.  유저 오브젝트를 이용해 해당 유저의 이미지를 얻어오는 서비스를 생성합니다.  
3.  유저 아이디를 이용해 유저를 얻어옵니다.  
4.  해당 유저의 이미지들을 얻어옵니다.  
5.  얻어온 이미지를 정렬하여 반환합니다.  
서비스가 분리되어 있는 부분이나, 정렬이 필요한 부분에 불필요하다고 느낄수도 있지만, 의도를 전달하기 위하여 추가했습니다.  
위 코드는 ImageController가 두 서비스의 의존적이며 구현 수정이 힘듭니다.  
또 테스트 용이성이 매우 낮습니다.  
  
우선 테스트 가능성을 위해 Service의 의존성을 주입받을 수 있습니다.  
{% highlight swift %}
class ImageController {
  private let userService: UserSerivce
  private let imageService: ImageService
  
  init(userService: UserService, imageService: ImageService) {
    self.userService = UserService
    self.imageService = imageService
  }
  
  func getImagesSortedByTime(userID: Int) -> [Image] {
    let user = userService.getUser(userID: userID)
    let images = imageService.getImages(user: user)
    return images.sorted { $0.timestamp > $1.timestamp }
  }
}
{% endhighlight %}
이제는 구현클래스에 의존하고 있지만, 구현체를 subclass등의 방법으로 테스트가 가능해졌습니다.  
하지만 메서드가 final이라면?, 혹은 네트워크에서 받아오던 것들을 인터넷이 연결이 안된다면 로컬에서 받아 오고 싶다면?  
컨트로러 내에 모든 내용을 구현할 수 있겠지만, 컨트롤러가 비대해진 다는점, 테스트가 어려워진다는점등의 이슈가 남게 됩니다.  
상속을 통해 NetworkImageService, LocalImageService를 만드는것도 부자연 스러워 보입니다.  
이를 프로토콜을 통해 해결해본다면,  
{% highlight swift %}
protocol UserServiceType {
  func getUser(userID: Int)
}

protocol ImageServiceType {
  func getImages(user: User) -> [Image]
}

class ImageController {
  private let userService: UserSerivceType
  private let imageService: ImageServiceType

  init(userService: UserServiceType, imageService: ImageServiceType) {
    self.userService = UserService
    self.imageService = imageService
  }

  func getImagesSortedByTime(userID: Int) -> [Image] {
    let user = userService.getUser(userID: userID)
    let images = imageService.getImages(user: user) // point
    return images.sorted { $0.timestamp > $1.timestamp }
  }
}
{% endhighlight %}
컨트롤러는 프로토콜에 의존적이므로 구현에는 관여하지 않게 되었습니다.  
imageService의 프로토콜은 User를 받아 이미지들을 되돌려 주는 책임이 있습니다.  
만약 주입받은 서비스에 문제가 있다면, 이는 컨트롤러 혹은 서비스의 문제가 아닌 서비스의 문제로 축소시킬 수 있습니다.  
위 코드에서 ImageService는 유저오브젝트를 받기 때문에 해당 서비스를 이용하려는 모든곳에선 유저를 얻어오는 과정이 필요합니다.  
이를 ImageService 가 필요하다면, 직접 얻어오도록 수정할 수 있습니다.  

{% highlight swift %}
protocol UserServiceType {
  func getUser(userID: Int)
}

protocol ImageServiceType {
  func getImages(userID: Int) -> [Image]
}

class NetworkImageService: ImageServiceType {
  private let userService: UserServiceType

  init(userService: UserServiceType) {
    self.userService = userService
  }

  func getImages(userID: Int) -> [Image] {
    let user = userService.getUser(userID: userID)
    let images = // 이미지를 네트워크등에서 얻어오는 로직
    return images
  }  
}

class ImageController {
  private let imageService: ImageServiceType

  init(imageService: ImageServiceType) {
    self.imageService = imageService
  }

  func getImagesSortedByTime(userID: Int) -> [Image] {
    let images = imageService.getImages(userID: userID)
    return images.sorted { $0.timestamp > $1.timestamp }
  }
}
{% endhighlight %}
이젠 컨트롤러는 이미지서비스에만 의존적이게 되었으며, Mock ImageService를 이용하여 테스트 용이성도 올라갔습니다.  
로컬 혹은 네트워크와 로컬 두가지를 이용하는 이미지 서비스를 만들어 컨트롤러에 주입만 해준다면, 컨트롤러의 수정 없이 기능을 변경시킬 수 있습니다.  
{% highlight swift %}
class LocalDatabaseImageService: ImageServiceType {
  func getImages(userID: Int) -> [Image] {
    let images = // 로컬 파일, 데이터베이스등 에서 얻어오는 로직
    return images
  }
}

class FallbackImageService: ImageServiceType {
  private let firstImageService: ImageServiceType
  private let fallbackImageService: ImageServicetype

  init(firstImageService: ImageServiceType, fallbackImageService: ImageServicetype) {
    self.firstImageService = firstImageService
    self.fallbackImageService = fallbackImageService
  }

  func getImagesSortedByTime(userID: Int) -> [Image] {
    // ImageServiceType이 Optional이 아니기 때문에 아래 코드는 틀린 코드입니다.  
    // 프로토콜이 비동기 객체(Promise, Rx)등을 리턴하여 에러 핸들링이 가능하다고 가정하고  
    // 의미를 봐주시면 좋겠습니다.  
    if let images = self.firstImageService(userID: userID) {
        return images
    } else if let images = self.fallbackImageService(userID: userID) {
        return images
    } else {
        return []
    }
  }
}
{% endhighlight %}
이렇게 구현을 해볼 수 있습니다.  
  
컨트롤러를 테스트 해본다면,  
{% highlight swift %}
class MockImageService: ImageServiceType {
  func getImagesSortedByTime(userID: Int) -> [Image] {
    return [Image(timestamp: 2), Image(timestamp: 1), Image(timestamp: 3)]
  }
}

func testImageSortedByTime() {
    // Arrange
    let controller = ImageController(imageService: MockImageService())

    // Act
    let images = controller.getImagesSortedByTime(userID: 0)

    // Assert
    XCTAssertEqual(images[0].timestamp, 3)
    XCTAssertEqual(images[1].timestamp, 2)
    XCTAssertEqual(images[2].timestamp, 1)
}

// 만약 서비스가 에러 혹은 optional을 리턴한다면, 에러를 돌려주는 Mock을 만들어 테스트 해볼 수 있습니다.  
{% endhighlight %}
  
이러한 코드를 작성하면, 처음보다 코드의 양이 증가 했는데, 이게 좋은것인지에 대한 의문이 있습니다.  
이는 테스트 용이성등 얻을 수 있는 장점과, 단점을 비교하여 선택의 문제로 남겨놓을 수 있겠습니다.  
두번째로 의존성을 누가 해결하냐의 문제가 있습니다.  
iOS의 경우 Controller가 의존성을 해결(resolve)할 수 있고, 혹은 외부 라이브러리의 도움을 받을 수 있습니다.  
라이브러리로는 [Swinject](https://github.com/Swinject/Swinject), [Pure](https://github.com/devxoul/Pure)있는데 Pure의 경우 README에 좋은 내용이 많이 있습니다.   

### 마무리
책 한권이 나오는 방대한 내용이지만, 모든 내용을 다루기 보다는 최대한 가볍게 다루어 봤습니다.  
때문에 과도한 설계의 문제등의 내용은 생략되었습니다.  
프로토콜을 설계할 때 API를 설계한다는 생각을 갖고 하는것이 제 경우에는 도움이 되었습니다.  
틀린 부분이나 생각이 다른 부분을 공유해 주신다면 정말 감사합니다.  

### 참조
[A Conversation with Erich Gamma](https://www.artima.com/lejava/articles/designprinciples.html)  
C#으로 배우는 적응형 코드 디자인 패턴과 SOLID 원칙 기반의 애자일 코딩, ISBN 9791185890371
