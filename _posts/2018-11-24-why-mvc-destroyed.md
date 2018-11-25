---
layout: post
title: iOS에서 MVC는 왜 망가질까
thumbnail: "assets/img/posts/2018-11-24-why-mvc-destroyed/model_view_controller.png"
tags: [ios, mvc, pattern, architecture]
---
한때 대표적인 아키텍쳐로 칭송받던 MVC패턴이 지금은 Massive ViewController로 조롱받고, 금기시 되는 패턴으로 여겨지고 있습니다.  
왜 Massive 해지는지 이해가 없는 상태에서 MVP, MVVM, RIBs등 유행하고 있는 패턴을 적용하면 Massive Presnter, Massive ViewModel로 변현만 됩니다.  
어떤 오해로 인해 MVC가 Massive해지는지, 망가지는지에 대한 제 생각입니다.  
  
### Apple의 MVC
![MVC]({{ site.baseurl }}/assets/img/posts/2018-11-24-why-mvc-destroyed/model_view_controller.png)  
우선 애플의 MVC 문서를 먼저 살펴보면,  
[Model-View-Controller by Apple](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/MVC.html)  
문서에서는 Model, View, Controller의 역할과 커뮤니케이션에 대해 다루고 있습니다.  
[layered architecture](https://ko.wikipedia.org/wiki/다층_구조)를 기반으로 Model, Controller, View 로 레이어가 나누어져 있고,  
Model <---> Controller <---> View 로 각 레이어는 커뮤니케이션 하는 레이어가 정해져있습니다.  
따라서 어떤 로직이 어떤 레이어에 있을지 예측 가능하고, 역할과 책임이 분리되어 있습니다.  
아래에서 세분화 하여 살펴보지만, 사실은 모두 각 레이어의 관심사 분리(SoC)와 단일책임원칙(SRP)를 구분 하지 않아서 생기는 문제입니다.  
  
### MVC가 망가지는 이유
#### a. Model layer의 영역 축소
Model layer의 영역을 단순히 Entity, [Anemic domain model](https://en.wikipedia.org/wiki/Anemic_domain_model) 로 너무 작게 생각하여 생겨나는 문제들이 있습니다.  
Model이 Non-anemic 해야 한다는 의미가 아닌, business logic을 담당하고 있는 오브젝트가 Model layer에 포함되지 않는 상태입니다.  
이러한 코드가 Controller에 들어가게 되는데, 예를 들면,  
{% highlight swift %}
// ItemTableViewController.swift
func loadItems() {
  guard let url = URL(string: "https://example.com/items") else {
    return
  }
  
  let urlSession = URLSession(configuration: .default)
  var urlRequest = URLRequest(url: url)
  urlRequest.httpMethod = "GET"
  urlRequest.addValue("application/json", forHTTPHeaderField: "Accept")
  
  let task = urlSession.dataTask(with: urlRequest) { (data, response, error) -> Void in
    guard let data = data, let items = try? JSONDecoder().decode([Item].self, from: data) else {
      return
    }
    self.items = items
    self.tableView.reloadData()
  }
  
  task.resume()
}
{% endhighlight %}
이제 Controller는 데이터를 가저오는 역할까지 담당하게 되었습니다.  
따라서 한 페이지에서 데이터를 요청하는 부분이 많아질수록 Controller의 크기는 커지게 됩니다, 즉 Massive Controller가 됩니다.  
  
데이터를 가저오는 부분을 Model로 구분하고 Controller가 이를 Notify 받는 설계로 변경하여 해결할 수 있습니다.  
{% highlight swift %}
protocol ItemReceiveDelegate {
  func recived(items: [Item])
}
 
protocol ItemServiceType {
  var delegate: ItemReceiveDelegate { get set }
  func loadItems()
}
{% endhighlight %}
데이터 영속화, 여러 로직들이 위 예시에 해당하며 이것들이 Controller 에 들어가면서 Controller의 크기가 커지게 됩니다.  

#### b. Comunication Flow 파괴(Massive View)
a케이스에서 Model 역할이 축소된 부분을 이야기 했습니다. 이와 비슷한 부분인데,  
Flow가 Model -> Controoler -> View 가 아닌, View가 Model, Controller 도움 없이 모든걸 다 하게 되는 케이스입니다.  
예를들어 A Controller에서 Item를 노출하는 테이블뷰가 있고, B Controller에서는 Item을 노출하는 테이블뷰와 몇개의 컴포넌트가 추가로 있습니다.  
따라서 2개의 Controller가 있는데, TableView는 공통적으로 사용하므로 재사용을 하고 싶습니다. 이는 좋은 생각입니다.  
이 과정에서 TableView내에 Model, Controller가 해야 할 일을 추가하게 되면서 안좋은 상황이 됩니다.  
{% highlight swift %}
func loadItem() {
  self.itemService.loadItems()
}

extension ItemTableView: ItemReceiveDelegate {
  func recived(items: [Item]) {
    self.items = items
    self.reloadData()
  }
}
{% endhighlight %}
데이터를 가저오는 로직, 혹은 TableView의 Delegate, DataSource까지 TableView가 구현하여 처리합니다.  
이 과정속에서 자연스럽게 View는 Controller, Model의 역할을 수행하게 됩니다.  
  
#### c. Massvie Model
b케이스와 동일한 예시입니다. 이번엔 반대로 Model 역할이 과대해지는 부분이 있는데,  
View를 Model에서 직접 조작하는 일이 많아지는 케이스입니다.  
예를들어, 네트워크, 작업완료등으로 Alert, Confirm, Animation등 UI요소에 접근하는 일을 Model 에서 하게 되는것입니다.  
이는 complete, error handler를 통해 VC에서 처리하거나,  
구체화된 UI에 의존하는것이 아니라, 추상화에 의존하면 해결할 수 있습니다.  
  
#### d. Singleton으로 이루어진 Service, Helper 들
Singleton pattern 동기와는 상관없이 모든 Service, Helper을 싱글톤으로 구성하는 케이스가 있습니다.  
이 자체는 패턴의 남용이지만, 대게 이렇게 되는 경우는 의존성에 대해 고려하지 않고 서로 얽혀있을 때 였습니다.  
각 Service Model들은 서로를 싱글톤으로 참조하고 동작합니다.  
이로 인해 테스가 어려워집니다. 그리고 b 케이스 혹은 다른 안좋은 케이스로 빠지기 쉬워집니다.  
혹은 하나의 싱글톤 객체가 너무 많은 역할을 하고 있는 경우입니다.  
AppDelegate, GodSingleton에서 모든걸 구현(20000~30000라인)하고 공유하는 식으로 사용하는 프로젝트를 봤습니다.  
  
### MVC의 한계
#### a. 테스트 커버리지의 한계
앱 어플리케이션 프로젝트에서는 View와 연관된 부분들은 테스트가 까다롭고  
테스트 대비 얻는 이익이 크지 않기때문에 진행을 하지 않는 경우가 많습니다.  
MVC에서 테스트는 M 까지가 용이합니다. 여러 Utils, Services, Helpers 등이 테스트 대상입니다.  
만약 모델 레이어의 테스트가 용이하지 않다면 설계의 문제가 있을 가능성이 있습니다.  
MVP는 M, P가 테스트 대상인데 Presenter에서는 추상화된 View에 접근하기 때문에 테스트가 가능합니다.  
MVVM은 M, VM이 테스트 대상인데 ViewModel은 View와 독립적이며 바인딩으로 연결되기 때문에 테스트가 쉽습니다.  
  
#### b. 여전히 Massive 해지는 Controller
이전 앱 어플리케이션에서는 트랜지션, 애니메이션 없이 개발해도 됐지만,  
요즘은 자연스러운 애니메이션 또한 스팩중 하나가 되었습니다.  
각 역할을 특정 오브젝트가 하겠지만, 이를 이용하는건 여전히 Controller 입니다.  
복잡한 뷰에서는 이용하는 Model, View 들이 많은데 서로의 통로 역할로도 Controller가 비대해집니다.  
이를 Controller 분리로 해결할 수 있겠지만, 빠른 개발속도와 트레이드 오프를 해야합니다.  
  
잘못된 포인트, 혹은 그 외 모든 피드백 환영합니다.  

