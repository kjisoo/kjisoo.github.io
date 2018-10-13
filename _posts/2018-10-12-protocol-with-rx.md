---
layout: post
title: RxSwift 친화적 프로젝트 만들기
tags: [swift, protocol, rx, rxswift, rxcocoa]
---
RxSwift(Reactive)는 비동기 데이터 흐름, 바인딩등을 위해 많은 프로젝트에서 사용하고 있습니다.  
Rx를 프로젝트에 어떻게 적용하거나 기존의 부분을 대체할지에 대해 프로토콜 중심으로 다뤄봅니다.
  
글 이해를 위해서 RxSwift 4.3기준, RxSwift에 대해 지식이 필요합니다.
  
### Observable로 Protocol 만들기
순수한 swift를 이용해 비동기 콜백을 만들때는 클로저를 이용합니다.
Rx를 이용하면 클로저를 Observable로 대체할 수 있습니다.  
클로저의 타입별로 어떻게 대체할지 살펴봅니다.  
{% highlight swift %}
protocol GithubType {
  func saveRepo(repo: Repo, completion: (Bool) -> Void)
  func findRepo(id: String, completion: (Repo?, Bool) -> Void)
  func myInfo(completion: (Info?, Bool) -> Void)
}
{% endhighlight %}
bool 혹은 error? 를 통해서 성공적으로 완료되었는지 결과를 체크할 수 있습니다.  
위와 같이 3개의 func이 있을때 위 completion을 Rx를 적용해 스트림으로 변경할 수 있습니다.  
단순하게 1:1 매핑으로 Observable로 바꾸면 다음 형태가 됩니다.  

{% highlight swift %}
protocol GithubType {
  func saveRepo(repo: Repo) -> Observable<Void>
  func findRepo(id: String) -> Observable<Repo?>
  func myInfo() -> Observable<Info>
}
{% endhighlight %}
Observable의 onError를 이용하면 에러를 검출할 수 있기 때문에 Bool이 사라졌습니다.  
`myInfo`에서 Error와 결과를 분리할 수 있으므로 옵셔널에서 논-옵셔널로 변경되었습니다.  
위 Observable을 Traits를 이용하면 더 명확하게 변경할 수 있습니다.  

{% highlight swift %}
protocol GithubType {
  func saveRepo(repo: Repo) -> Completable
  func findRepo(id: String) -> Maybe<Repo>
  func myInfo() -> Single<Info>
}
{% endhighlight %}
RxSwift의 Traits정보는 [Github RxSwift Traits](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Traits.md#rxswift-traits)<sub>1</sub>에서 확인할 수 있습니다.  
간단하게 요약하면  
`Completable` -> 반환 데이터 없는 성공(.completed) or 실패(.error)  
`Maybe` -> 반환 데이터 없는 성공(.completed) or 반환 데이터 있는 성공(.success) or 실패(.error)  
`Single` -> 반환 데이터 있는 성공(.success) or 실패(.error)  
각각 Void, Optional(Data), Data 와 유사합니다.  
따라서 저장 성공의 여부만 필요한 `saveRepo`는 `Completable`로,  
Repo 를 찾는 `findRepo`는 찾는 레포가 없을 수 있기 때문에 `Maybe<Repo>`  
`myInfo`가 항상 내 정보를 돌려준다고 가정하면, `Single<Info>` 가 됩니다.  
  
### 기존 프로토콜에 Rx 입히기
이미 만들어 사용중인 프로토콜, 구현체를 Rx로 변환하는 일은 많은 사이드 이팩트를 유발할 수 있습니다.  
직접 수정보다 extension을 활용해 Rx로 확장할 수 있습니다.  
기존 프로토콜 
{% highlight swift %}
protocol GithubType: AnyObject, ReactiveCompatible {
  func saveRepo(repo: Repo, completion: (Bool) -> Void)
  func findRepo(id: String, completion: (Repo?, Bool) -> Void)
  func myInfo(completion: (Info?, Bool) -> Void)
}
{% endhighlight %}
기존 프로토콜에 `AnyObject`, `ReactiveCompatible` 를 추가했습니다.  
`AnyObject`는 이후 나올 클로저에서 weak caputre를 위해 추가했고,  
`ReactiveCompatible`는 `extension Reactive where Base: GithubType` 표현식을 사용하기 위해 추가했습니다.   
기존 프로토콜을 변경할 수 없는 경우에는,  
`class Github: GithubType{}` 구현 클래스가 있다면, `extension Github: ReactiveCompatible` 로 할 수 있습니다.  
이제 위 코드에 Rx를 추가해보면,  
{% highlight swift %}
extension Reactive where Base: GithubType {
   func findRepo(id: String) -> Maybe<Repo> {
    return Maybe.create(subscribe: { [weak base] maybe -> Disposable in
      let error = NSError(domain: "Error object", code: 0, userInfo: nil)

      if let base = base {
        base.findRepo(id: id, completion: { (repo, result) in
          guard result == false else {
            maybe(.error(error))
            return
          }
          // 결과는 성공했지만, Repo 유무에 따라서 success, completed로 구분합니다.
          if let repo = repo {
            maybe(.success(repo))
          } else {
            maybe(.completed)
          }
        })
      } else {
        maybe(.error(error))
      }

      // 결과가 나오기 전에 Dispose되면 에러로 간주합니다.
      return Disposables.create { maybe(.error(error)) }
    })
  }
}
{% endhighlight %} 
전체 코드는 [gist](https://gist.github.com/kjisoo/1afa8da5bb447dd13ba9afab212b3fdf)<sub>2</sub>에 있습니다.  
  
기존 구현의 프록시 형태를 만들 수 있습니다.  
사용시에는  
{% highlight swift %}
class Github: GithubType {}
let github = Github()
github.rx.findRepo(id: "kjisoo").subscribe
{% endhighlight swift %}
의 형태가 됩니다.  
[Moya RxProvider](https://github.com/Moya/Moya/blob/master/Sources/RxMoya/MoyaProvider%2BRx.swift)<sub>3</sub> 를 보면 도움이 됩니다.  
  
### 링크
1. RxSwift의 Traits 설명 문서 - https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Traits.md#rxswift-traits
2. Reactive의 Extension 전체 코드 - https://gist.github.com/kjisoo/1afa8da5bb447dd13ba9afab212b3fdf
3. RxMoyaProvider의 코드 - https://github.com/Moya/Moya/blob/master/Sources/RxMoya/MoyaProvider%2BRx.swift
