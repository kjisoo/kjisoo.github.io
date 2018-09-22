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

### 예제


 
### 참조
[공변성과 반공변성은 무엇인가?](https://www.haruair.com/blog/4458)  
[공변성과 반공변성은 무엇인가?(원문)](https://www.stephanboyer.com/post/132/what-are-covariance-and-contravariance)  
[Covariance and Contravariance in Swift](https://medium.com/@aunnnn/covariance-and-contravariance-in-swift-32f3be8610b9)  

