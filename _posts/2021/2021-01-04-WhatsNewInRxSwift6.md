---
layout: post
title: "RxSwift 6의 변경사항 (What’s new in RxSwift 6)"
category:
  - RxSwift
tags:
  - RxSwift 6
comments: true
published: true
---

RxSwift 6이 릴리스 됐습니다. 이번 포스트에서는 RxSwift 6의 변경사항에 대해 살펴보겠습니다. 내용은 원문인 [이 포스트](https://dev.to/freak4pc/what-s-new-in-rxswift-6-2nog#new-logo)를 번역했습니다.

내용이 좀 많은데요. 개인적으로 가장 유용하다고 생각하는 기능 두 개를 목차에서 별⭐️로 표시해 두었습니다. 바쁘신 분들은 이 두 기능만은 꼭 확인하시길 추천드립니다.

## 목차
1. 새 로고
2. `Binder`가 RxCocoa에서 RxSwift로 이동
3. `withUnretained` 추가 (⭐️)
4. `@dynamicMemberLookup`을 사용한 `Binder`의 자동 합성 (⭐️)
5. `Infallible`
6. `Observable<Data>`를 위한 `decode(type:decoder:)`연산자 제공
7. `driver()`, `emit()`에 다중 바인딩 제공
8. `Single`의 구현을 `Swift`의 `Result`를 사용하도록 변경
9. `distinctUntilChange(at:)`연산자에서 Key Path 지원
10. `ReplayRelay` 연산자 추가
11. `DisposeBag` 함수 빌더 추가
12. 여러 연산자의 이름이 변경 됨
13. XCFrameworks 지원 개선

## 1. 새 로고
로고가 좀 더 귀엽게 변경됐습니다. ☺️

![RxswiftNewLogo]({{ site.baseurl }}/images/2021/RxswiftNewLogo.png)

## 2. `Binder`가 RxCocoa에서 RxSwift로 이동

```swift
viewModel.isButtonEnable.bind(to: myButton.rx.isEnabled)
```

`Binder`는 이름 그대로 input이나 output을 `Binding`하기 위해 사용하는데요. 그동안에는 `Binder`를 사용하기 위해 RxCocoa 모듈을 import 해야만 했습니다. 

커뮤니티의 많은 요청에 따라 RxCocoa 모듈에 있던 `Binder`를  RxSwift로 이동했습니다. RxSwift 6부터는 더 이상 `Binder`를 사용하기 위해 RxCocoa 모듈을 import할 필요가 없습니다.

## 3. `withUnretained` 추가 (⭐️)
클로저에서 `self`에 대한 약한 참조를 하고 클로저 안의 코드가 끝까지 실행되는 것을 보장하기 위해 보통 다음과 같은 코드를 사용합니다.

```swift
viewModel.importantInfo
    .subscribe(onNext: { [weak self] info in 
        guard let self = self else { return }
        self.doImportantTask(with: info)
    })
    .disposed(on: disposeBag)
```

이런 코드를 한번만 작성하는 것이라면 괜찮겠지만, 실제로는 여러번 작성해야하기 때문에 코딩시 가독성과 효율성이 떨어집니다.  

[RxSwiftExt](https://github.com/RxSwiftComunity/RxSwiftExt)(RxSwift에 없는 연산자를 추가로 구현한 오픈소스) 라는 익스텐션에 `withUnretained` 라는 연산자가 있는데, 이 연산자의 인기가 계속 올라감에 따라 이 연산자를 RxSwift에 공식적으로 포함시켰습니다. 

RxSwift 6부터는 위 코드를 `withUnretained`를 사용해 다음과 같이 작성할 수 있습니다.

```swift
viewModel.importantInfo
  .withUnretained(self) // (Object, Element) 듀플 반환
  .subscribe(onNext: { owner, info in 
    owner.doImportantTask(with: info)
  })
  .disposed(by: disposeBag)
```

코드가 훨씬 깔끔해 졌습니다. 

## 4. `@dynamicMemberLookup`을 사용한 `Binder`의 자동 합성 (⭐️)
RxSwift 에서는 특정 객체의 프로퍼티에 `Binding` 하기 위해  `.rx`라는 네임스페이스를 사용합니다.

예를들어, 만약 다음과 같은 `MyView` 경우

```swift
class MyView: UIView { 
    var title: String
    var subtitle: String?
    var icon: UIImage?
}
```

프로퍼티에 `Binding`할 수 있도록 위해 일반적으로 이렇게 Reactive extension을 구현합니다.

```swift
extension Reactive where Base: MyView {
    var title: Binder<String> {
       Binder(base) { base, title in 
           base.title = title
       }
    }

    var subtitle: Binder<String?> {
       Binder(base) { base, subtitle in 
           base.subtitle = subtitle
       }
    }

    var icon: Binder<UIImage?> {
       Binder(base) { base, icon in 
           base.icon = icon
       }
    }
```

다음은 프로퍼티를 `binding`하는 코드입니다.

```swift
viewModel.title.bind(to: myView.rx.title)
viewModel.subtitle.bind(to: myView.rx.subtitle)
viewModel.icon.drive(myView.rx.icon)
```

프로퍼티에 `Binding`을 지원하기 위해 Base의 모든 프로퍼티에 대해 Reactive extension을 만드는 것은 매우 반복적인 비효율적인 작업이 아닐 수 없습니다. 

행복하게도 Swift 5.1 부터 이 문제 해결을 위한 `@dynamicMemberLookup`을 제공해 줍니다. 

RxSwift 6에서는 어떤 클래스든 모든 프로퍼티를 자동으로 합성해 `Binding`이 가능한 Reactive extension을 제공해줍니다. 👍 

이제 프로퍼티를 `Binding`하기 위해 생성한 Reactive extension을 코드에서 완전히 제거 할 수 있습니다. 

앞으로 Xcode에서`AnyObject`를 상속하는 어떤 클래스에서든 `.rx`를 입력만 하면 즉각 자동으로 합성된 모든 프로퍼티의 `Binder`를 보실 수 있을 것입니다. 😆

![ReactiveExtension]({{ site.baseurl }}/images/2021/ReactiveExtension.jpg)

참고로 이미 선언돼 있는 Reactive extension은 자동으로 합성되어 생성되는 Reactive extension보다 우선해서 실행됩니다. 그렇기 때문에 커스텀하게 구현한 Reactive extension이 자동으로 생성된 Reactive extension에 의해 오버라이딩 되지 않을까 걱정하지 않으셔도 됩니다.

## 5. `Infallible`
`Infallible`는 `Observable`과 한가지가 다른 새로운 타입입니다.
`Infallible`는 실패하지 않습니다. 다시 말하면 `.error` 이벤트를 방출(emit)하지 않습니다.

예를들어, `Observable.create`와 유사하게 `Infallible.create`를 다음과 같이 생성할 수 있습니다.

```swift
Infallible<String>.create { observer in
    observer(.next("Hello"))
    observer(.next("World"))
    observer(.completed)
    // 여기에서 .error를 방출할 수 없습니다.

    return Disposables.create {
        // Clean-up
    }
}
```
`Infallible`에서는 `.next(Element)` 혹은 `.completed` 이벤트만 전달할 수 있습니다.

RxCocoa를 사용해 보셨다면 “그럼 `Driver`나 `Signal`과 차이점이 뭐지?” 라고 의아해 하실 수 있는데요.

우선 `Driver`나 `Signal`은 RxCocoa에 포함돼 있는 반면`Infallible`는 RxSwift에 포함돼 있습니다. 더 중요한 차이는`Driver`나 `Signal`는 항상 `MainScheduler`를 사용하고 `share()`를 사용해 자원을 공유합니다. 반면 `Infallible`는 이 경우에 해당하지 않는 완전히 새로운 타입의 기본 `Observable`입니다.

## 6. `Observable<Data>`를 위한 `decode(type:decoder:)`연산자 제공
RxSwift 6에서는 `Data`를 방출하는 `Observable`에서 동작하는 `Combine`과 유사한 `decode` 연산자를 제공합니다.

```swift
service.rx
       .fetchJSONUsers() // Observable<Data> 반환
       .decode(type: [User].self, decoder: JSONDecoder()) // Observable<[User]> 반환
```

## 7. `driver()`, `emit()`에 다중 바인딩 제공
RxSwift 5에서 처음으로 `Binder`에 다중 바인딩이 제공 됐습니다.

```swift
viewModel.string.bind(to: input1, input2, input3)
```

RxSwift 6에서는 같은 다중 바인딩을 `Driver`와 `Signal`에도 제공합니다.
```swift
viewModel.string.drive(input1, input2, input3)
viewModel.number.emit(input4, input5)
```

## 8. `Single`의 구현을 `Swift`의 `Result`를 사용하도록 변경
RxSwift 5까지 `Single`은 커스텀 이벤트를 사용했습니다.

```swift
public enum SingleEvent<Element> {
    case success(Element)
    case error(Swift.Error)
}
```

RxSwift 6 부터 커스텀 이벤트 대신 Swift의 `Result<Element, SwiftError>`를 사용합니다. SingleEvent는 `Result<Element, SwiftError>` 의 별칭(alias)일 뿐입니다.

이런 변화는 `subscribe`같은 다른 API의 사용에도 반영이 되었습니다.

```swift
// RxSwift 5
single.subscribe(
    onSuccess: { value in
        print("Got a value: \(value)")
    },
    onError: { error in   // onError사용
        print("Something went wrong: \(error)")
    }
)

// RxSwift 6
single.subscribe(
    onSuccess: { value in
        print("Got a value: \(value)")
    },
    onFailure: { error in // onFailure 사용
        print("Something went wrong: \(error)")
    }
)
```

## 9. `distinctUntilChange(at:)`연산자에서 Key Path 지원
`distinctUntilChange`는 비효율적인 중복작업을 피할 수 있게 해주는 매우 유용한 연산자 입니다. 이 연산자를 RxSwift 5까지는 이렇게 사용했습니다.

```swift
myStream.distinctUntilChanged { $0.searchTerm == $1.searchTerm }
```

RxSwift 6부터는 추가적으로 Key Path를 이용해 보다 간략하게 코드를 작성할 수 있습니다.
```swift
myStream.distinctUntilChanged(at: \.searchTerm)
```

## 10. `ReplayRelay` 연산자 추가
 `Relay`는 `Subject`의 wrapper로 스트림이 절대 실패(fail)하거나 종료(complete)되지 않는 것을 보장합니다. RxSwift 6에서는 `ReplaySubject`를 wrapping한 `ReplayRelay`를 기존의 `BehaviorRelay`와 `PublishRelay`에 추가했습니다.

`ReplayRelay`를 생성하는 방법은 `ReplaySubject`를 생성하는 방법과 완전히 동일합니다.
```swift
// Subject
ReplaySubject<Int>.create(bufferSize: 3)

// Relay
ReplayRelay<Int>.create(bufferSize: 3)
```

## 11. `DisposeBag` 함수 빌더 추가
RxSwift 6 에서는 `DisposeBag`에 SwiftUI 같이 콤마(,)가 없는 함수 빌더가 추가 되었습니다.

```swift
var disposeBag = DisposeBag { 
    observable1.bind(to: input1)

    observable2.drive(input2)

    observable3.subscribe(onNext: { val in 
        print("Got \(val)")
    })
}

// disposeBag의 insert에도 사용 가능합니다.
disposeBag.insert {
    observable4.subscribe()

    observable5.bind(to: input5)
}
```

## 12. 여러 연산자의 이름이 변경 됨
RxSwift 6에서는 많은 연산자의 이름을 Swift의 코드 가이드라인에 따라 변경했습니다.

변경된 이름은 다음과 같습니다. (여기에 명시되지 않은 연산자가 있을 수 있습니다.)

| RxSwift 5      | RxSwift 6 |
| ----------- | ----------- |
| catchError(_:)      | catch(_:)|
| catchErrorJustReturn(_:)   | catchAndReturn(_:)|
| elementAt(_:)  | element(at:)|
| retryWhen(_:) | retry(when:) |
| takeUntil(_:)  | take(until:) |
| takeUntil(behavior:_:) | take(until:behavior:) |
| takeWhile(_:) | take(while:) |
| takeWhile(behavior:_:) | take(while:behavior:) |
| take(.seconds(3)) | take(for: .seconds(3)) |
| skipWhile(_:) | skip(while:) |
| takeUntil(_:) | take(until:) |
| observeOn(_:) | observe(on:) |
| subscribeOn(_:) | subscribe(on:) |

### 13. XCFrameworks 지원 개선
RxSwift 6의 모든 릴리스는 앞으로 XCFramework 번들로도 제공됩니다.

## 정리
RxSwift 6에서는 이와 같이 많은 기능이 추가되거나 변경 됐는데요. 이 포스트에서 소개하지 않은 소소한 개선사항이나 버그 수정도 있습니다. 더 자세한 내용은 [여기](https://github.com/ReactiveX/RxSwift/releases/tag/6.0.0)에서 확인 가능합니다.

RxSwift 6를 확인하고 느낀점은 Combine이 있지만, RxSwift를 이렇게 계속 보완/발전시키는걸 봐서는 RxSwift가 쉽게 Combine으로 대체되진 않을꺼란 생각이 드네요. 

또 RxSwift의 main 메인테이너인 [Shai Mishali](https://twitter.com/freak4pc) 이분은 RxSwift뿐만아니라 Swift, Combine도 잘 하시는데, '역시 다른 언어를 잘하니 거기에서 이렇게 차용해 올 수 있구나?' 라는 생각도 했습니다. 너무 당연한 얘기지만요. 공부를 더 해야합니다. 😅

이상 포스트를 마치겠습니다.

👨🏻‍💻지식이 +5 늘었다. 다음 포스트에서 또 만나요 🚀😄
