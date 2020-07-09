---
layout: post
title: Swift Package Manager(SPM)에 새롭게 추가된 바이너리 배포
tags: [ios, carthage, cocoapods, spm, swiftpackagemanager]
---
### SPM 바이너리 배포 소개 
swift5.3, xcode12
remote checksum, local path
빌드속도 감소
first party
prebuilt binaries SPM으로 배포 가능
prebuilt 권장하는건 아님

### xcframework 빌드
framework zip

### carthage 차이
carthage의 binary 만 커버하고 있음
carthage의 use-binary는 별개의 케이스
carthage binary -> build
spm 택1
직접 checksum 생성해야함, 동적으로 선택 불가,
local 추적 힘듬 google 같은거에나 써야할듯 
remote 버전 변경 귀찮음 
carthage + rome 이 좋아보임

플랫폼 종속성(checksum관련)

결론 : 절반짜리

https://developer.apple.com/documentation/swift_packages/distributing_binary_frameworks_as_swift_packages
https://github.com/apple/swift-evolution/blob/master/proposals/0272-swiftpm-binary-dependencies.md
https://developer.apple.com/wwdc20/10147 Distribute binary frameworks as Swift packages
https://developer.apple.com/wwdc19/416 Binary Frameworks in Swift

