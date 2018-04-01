---
layout: post
title: UIView의 translatesAutoresizingMaskIntoConstraints 속성에 대한 이해
tags: [translatesAutoresizingMaskIntoConstraints, constraint, autoresizing, Swift, iOS, auto layout]
redirect_from:
  - /blog/uiview의-translatesautoresizingmaskintoconstraints-속성에-대한-이해
---
UIView의 [translatesAutoresizingMaskIntoConstraints](https://developer.apple.com/documentation/uikit/uiview/1622572-translatesautoresizingmaskintoco)속성에 대해 알아봅니다.

요약하면, 해당속성이 true이면, frame based인 [autoreszing mask](https://developer.apple.com/documentation/uikit/uiview/1622559-autoresizingmask)를 auto layout에서 사용할 수 있게 해줍니다.  
storyboard, xib와 같이 IB로 생성된 뷰는 자동으로 해당 속성이 false가 됩니다. 반대로 코드로 뷰를 작성할 경우 기본값은 true입니다.

해당 속성덕분에 아래의 코드가 오토 레이아웃으로 만든 뷰에서 사용이 가능합니다.  
{% highlight swift %}
let someView = UIView(frame: CGRect(x: 0, y: 0, width: 200, height: 200))
self.view.addSubview(someView)
{% endhighlight %}
자세히 알아보면, autoresizing mask를 통해 변화된 사이즈, 위치를 constraint를 사용하는 다른 뷰들과 함께 사용할 수 있도록 제약을 만들어줍니다.  
제약은 바로 만들어지는것은 아니고, 필요할 때 만들어집니다.
{% highlight swift %}
let autoResizingView = UIView(frame: CGRect(x: 0, y: 0, width: 400, height: 400))
autoResizingView.backgroundColor = .orange
autoResizingView.translatesAutoresizingMaskIntoConstraints = true // 원래 true이지만, 의미 전달을 위해
autoResizingView.autoresizingMask = [.flexibleWidth, .flexibleHeight] // 다음 테스트를 위해 미리 할당
self.view.addSubview(autoResizingView)
{% endhighlight %}

위 코드를 오토레이아웃으로 작성된 뷰에서 확인해보면, 아무 제약도 생기지 않습니다. `autoResizingView` 가 하위뷰로 들어갔지만, 해당 뷰와 연관된 제약이 없기때문입니다.  
{% highlight swift %}
let constraintView = UIView()
constraintView.backgroundColor = .blue
constraintView.translatesAutoresizingMaskIntoConstraints = false
self.view.addSubview(constraintView)
constraintView.addConstraint(NSLayoutConstraint(item: constraintView, attribute: .width, relatedBy: .equal, toItem: nil, attribute: .notAnAttribute, multiplier: 1, constant: 50))
constraintView.addConstraint(NSLayoutConstraint(item: constraintView, attribute: .height, relatedBy: .equal, toItem: nil, attribute: .notAnAttribute, multiplier: 1, constant: 50))
self.view.addConstraint(NSLayoutConstraint(item: autoResizingView, attribute: .bottom, relatedBy: .equal, toItem: constraintView, attribute: .top, multiplier: 1, constant: 0))
self.view.addConstraint(NSLayoutConstraint(item: autoResizingView, attribute: .left, relatedBy: .equal, toItem: self.view, attribute: .left, multiplier: 1, constant: 0))
{% endhighlight %}

위 코드를 이어서 붙이면, `constraintView`는 50, 50사이즈며 `constraintView`의 top과 `autoResizingView`의 bottom 을 일치 시킵니다. `autoResizingView`뷰가 다른 뷰와의 제약이 생겨 필요할 때가 되었습니다.

`func viewDidLayoutSubviews()`이 실행되어 새로운 제약이 만들어지는데, 수동으로 만든 제약외에 아래와 같은 제약이 자동으로 생성됩니다.

```
<NSAutoresizingMaskLayoutConstraint:0x h=-&- v=-&- UIView:autoResizingView.midX == UIView:view.midX + 12.5   (active)>
<NSAutoresizingMaskLayoutConstraint:0x h=-&- v=-&- UIView:autoResizingView.width == UIView:view.midX.width + 25   (active)>
<NSAutoresizingMaskLayoutConstraint:0x h=-&- v=-&- UIView:autoResizingView.midY == UIView:view.midX.midY - 133.5   (active)>
<NSAutoresizingMaskLayoutConstraint:0x h=-&- v=-&- UIView:autoResizingView.height == UIView:view.midX.height - 267   (active)>
```
(뷰의 주소를 이름으로 수정했습니다.)

view의 frame이 (0, 0, 375, 667)임을 생각하면, `autoResizingView`의 width는 375+25 = 400, height는 667-267 = 400으로 위에서 설정한 사이즈가 나옵니다.

![1]({{ site.baseurl }}/assets/img/posts/2018-03-02-UIView-translatesAutoresizingMaskIntoConstraints/1.png)
화면을 회전해도 정상적으로 작동합니다.  

`autoResizingView`의 mask는 [.flexibleWidth, .flexibleHeight]이었기 때문에, 화면 스크린이 늘어난/줄어난 사이즈와 동일하게 뷰의 width, height가 늘어나거나 줄어듭니다.  
따라서 width는 375 -> 667로 292만큼 늘어났고, height는 667 -> 375로 292만큼 줄어들어 해당 뷰의 사이즈는 692, 108이 되었습니다.  
위 내용을 바탕으로 제약조건도 업데이트 됩니다.

만약 `autoResizingView`의 해당 속성을 false로 바꾸게 되면 자동으로 만들어진 제약들이 모두 사라지게 됩니다.  
![2]({{ site.baseurl }}/assets/img/posts/2018-03-02-UIView-translatesAutoresizingMaskIntoConstraints/2.png)  
속성을 false로 바꾸어 자동으로 생성된 제약들이 모두 사라져 사이즈가 0,0이 되어 화면에 보이지 않습니다.

xib를 사용하여 UI를 만들어도 속성의 기본값이 true인 UIViewController의 view와 Cell의 contentView가 있습니다.
`UIViewController`의 `func loadView()`에서 뷰가 코드로 생성되고 바인딩 됩니다.  
이러한 뷰들은 UIView-Encapsulated-Layout-Width or Height라는 이름으로 제약이 생성됩니다.  
위 뷰들의 속성을 false로 바꾸면 전체적인 레이아웃이 깨지거나 무한 루프를 도는 모습을 볼 수 있습니다.

## 추가적으로...
`UIViewController`의 `view`의 해당 속성을 false로 바꾸어도 문제가 발생하지 않는 상황이 있습니다. 
 - `RootViewController`이어야 하며,
 - `ViewWillLayoutSubViews`이전에 false로 변경되어 있어야 한다.

위 두가지가 충족되는 상황인데요. 위 두가지가 충족되는 상황이면,   
`UIWindow`의 `open func makeKeyAndVisible()`이 실행되면 내부적으로
`private func addRootViewControllerViewIfPossible()`이 실행되는데, `subview(ViewController의 view)`의 속성이 변하여 `private func _rootViewConstraintsUpdateIfNecessary(forView arg1: Any!) -> Any!`이 실행됩니다.

해당 함수 혹은 어딘가에서 제약이 만들어져 `private func addConstraints(_ constraints: [NSLayoutConstraint])`로 window, view의 사이즈를 동일하게 하는 제약이 추가됩니다.

`private func addRootViewControllerViewIfPossible()`내에서 진행되는 과정이므로 최초에만 작동하게 됩니다. 

만약, false로 바꾸지 않고 정상적인 초기화 과정이었다면,  
`UIViewController`의 `open func updateViewConstraints()`이 실행되어 서브뷰들로 이벤트가 전파됩니다.
 
view의 `private func _updateSystemConstraints()`에서, `private func _updateAutoresizingConstraints()`이 실행되고, 해당 함수가 실행될때 이 안에서 `private func _setAutoresizingConstraints(_ arg1: Any!)`이 실행됩니다. 

해당 동작은 window와의 관계와는 무관하며 UIView-UIView의 `translatesAutoresizingMaskIntoConstraint`속성에 의한 동작입니다.
