---
layout: post
title: WKWebView의 UIDelegate completion crash 해결하기
tags: [ios, swfit, wkwebview, troubleshooting]
---
WKWebView에선 UIDelegate를 통해서 웹에서 일어난 alert, confirm, prompt에 대해서 custom ui를 제공할 수 있게 도와줍니다.   
UIDelegate에서는 CompletionHandler를 통해서 javascript와 값을 주고받는데, 핸들러를 호출하지 않거나, 두번 호출하게 되면 크래시가 나면서 앱이 종료됩니다.    
```
Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: 'Completion handler passed to -[ViewController webView:runJavaScriptAlertPanelWithMessage:initiatedByFrame:completionHandler:] was not called'
Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: 'Completion handler passed to -[ViewController webView:runJavaScriptAlertPanelWithMessage:initiatedByFrame:completionHandler:] was called more than once'

Completion handler passed to -[ViewController webView:runJavaScriptConfirmPanelWithMessage:initiatedByFrame:completionHandler:] was not called
Completion handler passed to -[ViewController webView:runJavaScriptConfirmPanelWithMessage:initiatedByFrame:completionHandler:] was called more than once

Completion handler passed to -[ViewController webView:runJavaScriptTextInputPanelWithPrompt:defaultText:initiatedByFrame:completionHandler:] was not called
Completion handler passed to -[ViewController webView:runJavaScriptTextInputPanelWithPrompt:defaultText:initiatedByFrame:completionHandler:] was called more than once
```
alert, confirm, prompt에서 핸들러를 호출하지 않았을때, 두번호출할때 나는 에러인데 이를 간단하게 CompletionHanderWrapper를 구현하여 해결할 수 있습니다.  

### 문제가 발생하는 원인  
문제가 발생하는건 핸들러를 호출하지 않은 상태로 핸들러가 메모리에서 사라지거나, 2번 호출하는 경우인데 2번 호출하는 경우보다는 호출하지 않는 케이스가 많을거라 생각합니다.  
호출하지 않고 크래시가 나는 경우 delegate내 코드 문제가 아니라, WebView를 포함하고 있는 Controller이 사라지는 케이스를 의심해봐야 합니다.  
프로젝트가 충분히 크다면, Deeplink, universal link를 이용하여 화면 네비게이션을 움직이고 있을 수 있는데, 이때 문제가 발생할 수 있습니다.

### 해결
CompletionHandler에 대해  `1회는 무조건 호출한다`, `2회는 호출할 수 없게한다`를 만족하면 됩니다.   
{% highlight swift %}
class CompletionHandlerWrapper<Element> {
  private var completionHandler: ((Element) -> Void)?
  private let defaultValue: Element

  init(completionHandler: @escaping ((Element) -> Void), defaultValue: Element) {
    self.completionHandler = completionHandler
    self.defaultValue = defaultValue
  }

  func respondHandler(_ value: Element) {
    completionHandler?(value)
    completionHandler = nil
  }

  deinit {
    respondHandler(defaultValue)
  }
}
{% endhighlight %}
위 핸들러를 통해 `() -> Void`, `(Bool) -> Void`, `(String?) -> Void`의 핸들러를 하나의 클래스로 해결할 수 있습니다.  
핸들러를 생성하고 respondHanlder를 통해 핸들러를 조작한다면, 1회만 호출됨을 보장합니다.  
핸들러가 메모리에서 사라질 경우 1회 호출을 보장합니다.   
  
{% highlight swift %}
  extension ViewController: WKUIDelegate {
    func webView(_ webView: WKWebView, runJavaScriptAlertPanelWithMessage message: String, initiatedByFrame frame: WKFrameInfo, completionHandler: @escaping () -> Void) {
      let completionHandlerWrapper = CompletionHandlerWrapper(completionHandler: completionHandler, defaultValue: Void())
      /* custom UI */
  }
  
  func webView(_ webView: WKWebView, runJavaScriptConfirmPanelWithMessage message: String, initiatedByFrame frame: WKFrameInfo, completionHandler: @escaping (Bool) -> Void) {
    let completionHandlerWrapper = CompletionHandlerWrapper(completionHandler: completionHandler, defaultValue: false)
    let alertController = UIAlertController(title: message, message: nil, preferredStyle: .alert)
    alertController.addAction(UIAlertAction(title: "확인", style: .default) { _ in completionHandlerWrapper.respondHandler(true) })
    alertController.addAction(UIAlertAction(title: "취소", style: .cancel) { _ in completionHandlerWrapper.respondHandler(false) })
    present(alertController, animated: true, completion: nil)
  }
  
  func webView(_ webView: WKWebView, runJavaScriptTextInputPanelWithPrompt prompt: String, defaultText: String?, initiatedByFrame frame: WKFrameInfo, completionHandler: @escaping (String?) -> Void) {
    let completionHandlerWrapper = CompletionHandlerWrapper(completionHandler: completionHandler, defaultValue: "")
    /* custom UI */
  }
}
{% endhighlight %}
실제 사용시 `completionHandlerWrapper`를 클로저등을 통해 alertController과 life-cycle를 같이할 수 있습니다.  
