---
layout: post
title: Background blur에 UIBezierPath 적용하기
feature-img: "assets/img/posts/2017-09-24-UIBezierPath-with-Background-blur/3.png"
thumbnail: "assets/img/posts/2017-09-24-UIBezierPath-with-Background-blur/3.png"
tags: [iOS, Swift, blur, path, UIBezierPath]
redirect_from:
  - /blog/ios-path와-background-blur-함께-적용하기
  - /blog/ios-path와-background-blur-함께-적용하기/
---
카메라 프리뷰위에 Background blur를 올리기 위한 과정입니다.  
개발의 최종 화면은 다음과 같은 화면입니다.  
블러뷰 아래에 있는 이미지 대신 카메라가 들어갑니다.  

![1]({{ site.baseurl }}/assets/img/posts/2017-09-24-UIBezierPath-with-Background-blur/1.png)

처음 계획은 가음과 같았습니다.  

![2]({{ site.baseurl }}/assets/img/posts/2017-09-24-UIBezierPath-with-Background-blur/2.png)  
카메라 프리뷰 위에 블러 이펙트 뷰를 추가하고, 블러 이펙트 뷰에 `UIBezierPath`로 마스크를 추가하는 것입니다.  
코드로 보면,  
{% highlight swift %}
let blurEffect = UIBlurEffect(style: .light)
let effectView = UIVisualEffectView(effect: blurEffect)
effectView.frame = view.frame

let path = UIBezierPath(rect: view.bounds)
let visualPath = UIBezierPath(roundedRect: CGRect(x: 50,
                                                  y: 200,
                                                  width: self.view.frame.width - 100,
                                                  height: self.view.frame.height - 400),
                              cornerRadius: 8)
path.append(visualPath)

let maskLayer = CAShapeLayer()
maskLayer.path = path.cgPath;
maskLayer.fillRule = kCAFillRuleEvenOdd
effectView.layer.mask = maskLayer

self.view.addSubview(effectView)
{% endhighlight %}
![3]({{ site.baseurl }}/assets/img/posts/2017-09-24-UIBezierPath-with-Background-blur/3.png)  
(좌측부터 iOS 9, 10, 11)

iOS9, iOS11 의도한 결과대로 나옵니다.  iOS10 의도한 결과가 다릅니다.  
layer에 mask 는 되었지만, 블러가 사라졌습니다.  
해당 방법은 실패하여, 다음 시도는 다른 방법을 적용해봤습니다.

![4]({{ site.baseurl }}/assets/img/posts/2017-09-24-UIBezierPath-with-Background-blur/4.png)  

{% highlight swift %}
let maskView = UIView(frame: view.frame)
let maskLayer = CAShapeLayer()
maskLayer.path = path.cgPath;
maskLayer.fillRule = kCAFillRuleEvenOdd
maskView.layer.mask = maskLayer
maskView.backgroundColor = .white

effectView.mask = maskView
{% endhighlight %}
maskLayer 를 바로 effectView 에 적용시키지 않고, 하나의 뷰를 더 만들어 해당 뷰에 적용시킨 후, 해당 뷰를 maskView 로 사용했습니다.  
![5]({{ site.baseurl }}/assets/img/posts/2017-09-24-UIBezierPath-with-Background-blur/5.png)  
(좌측부터 iOS 9, 10, 11)

iOS 9에서 블러 뷰 자체가 보이지 않습니다. iOS 10에선 잘나옵니다. iOS 11에선 블러는 보이는데 마스크가 적용되지 않았습니다.  
의외에 여러가지 시도를 해봤지만 한가지 방법으로 원하는 결과를 얻기에는 실패했습니다.  

결국 위 두 방법을 섞기로 했습니다.  
{% highlight swift %}
if #available(iOS 11, *) {
  self.layer.mask = shapeLayer
} else if #available(iOS 10, *) {
  self.mask = createMaskView(with: shapeLayer)
} else {
  self.layer.mask = shapeLayer
}
{% endhighlight %}
위 방법처럼 9, 11에선 layer의 mask 를 입히고, 10 경우에선 mask view 를 사용합니다.

해당 부분을 구현한 코드를 github에 올려놓았습니다.  [MaskBlurView](https://github.com/kjisoo/MaskBlurView)
