---
layout: post
title: iOS12 간편해지는 인증 과정
tags: [swift, article, ios12, password-auto-fill]
---
이번 WWDC에서는 흥미로운 내용들이 발표되었습니다. 그중 인증플로우와 관련된 내용을 살펴보고자 합니다. 
앱 안에서 비밀번호를 자동으로 채워주는 기능은 iOS11에서 소개된 바 있습니다. [WWDC 2017 Introducing Password AutoFill for Apps](https://developer.apple.com/videos/play/wwdc2017/206/) 
이번에 소개된 부분은 위 기능에 이어서, 회원가입, 인증코드, 그리고 웹뷰 등 으로 확장되었습니다. [WWDC 2018 Automatic Strong Passwords and Security Code AutoFill](https://developer.apple.com/videos/play/wwdc2018/204) 

### 회원가입 
회원가입 TextField의 ContentType에 newPassword가 추가 되었습니다. 
iOS에서 제안하는 비밀번호를 사용할 경우 Associated Domains를 참조하여 키체인에 저장됩니다. 
Associated Domains가 없을 경우 해당 기능을 이용할 수 없습니다. 
자동으로 생성되는 비밀번호의 룰을 입력해줄 수 있는데, 
{% highlight swift %}
let rulesDescriptor = "allowed: upper, lower, digit; required: [$];"
newPasswordTextField.passwordRules = UITextInputPasswordRules(descriptor: rulesDescriptor)
{% endhighlight %}
[Password Rules Validation Tool](https://developer.apple.com/password-rules) 
길이, 허용하는 문자, 필수 문자등을 설정할 수 있습니다. 

### 인증코드 
인증코드가 메시지로 오게되면 새로운 휴리스틱이 적용되어 인증코드를 찾고, 키보드에 표시해줍니다. 
![인증코드]({{ site.baseurl }}/assets/img/posts/2018-06-09-iOS12-authentication/code.png) 
 
### 웹뷰에서 키체인
기존 iOS11에서는 SFSafariWebView에서만 키체인에 접근할 수 있었습니다. 
iOS12에서는 WKWebView, UIWebView에서도 키체인에 접근할 수 있습니다. 
![키체인]({{ site.baseurl }}/assets/img/posts/2018-06-09-iOS12-authentication/webview-keychain.png) 
많은 메타 서비스들이 해당 부분에서 효과를 볼것같습니다. 

