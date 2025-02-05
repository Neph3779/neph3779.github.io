---
layout: post
title: "Swift weak, strong, unowned 정리"
image:
categories: Swift
tags: 
  - ARC
  - weak
  - strong
  - unowned
sitemap:
  changefreq: daily
  priority: 1.0

---



제가 Swift에서 weak이란 단어를 처음 본 것은 스토리보드에서 UI를 코드로 끌어왔을때였습니다.

자동으로 weak var 선언이 되는 것을 보며 그 이유가 궁금했었는데 이번글에서는 weak을 사용하는 이유, ARC 등에 대해 다루고자 합니다.

## ARC란

인스턴스의 수명주기가 끝나는 시점(Reference Count가 0이 되는 시점)에 Swift는 해당 인스턴스를 nil로 바꿔주는 코드를 실행하며(deallocate 작업) 이것을 Automatic Reference Counting이라 칭합니다.

ARC를 공부하기 앞서 반드시 알아두어야 하는 것은 인스턴스의 "수명"입니다.

코딩을 처음 공부할때 

>  전역변수, 지역변수를 이야기하며 지역변수는 해당 블록 내에서만 살아있고, 전역변수는 프로그램이 동작하는 동안 살아있다.

와 같은 문장을 보신적이 있으실겁니다. Swift도 마찬가지로 인스턴스는 특정 블록 내에서 살아있습니다.

<br/> 

하지만 특정 블록이 끝났다고해서 무조건 해당 인스턴스(A)를 deallocate해버리면 문제가 발생합니다.

다른 곳에서 A를 사용하고 있었다면 A가 갑자기 nil로 변경되었을때 이에 대처할 수 없기 때문입니다.

<br/> 

Swift는 이런 경우를 방지하고자 "Reference Count"라는 장치를 두었습니다.

만약 A를 B가 사용하고 있다면 Reference Count를 사용하는 사람(인스턴스)의 숫자만큼 증가시켜주는 것입니다.

A가 존재하던 블록이 끝나서 Reference Count가 하나 줄어들더라도 B가 사용함으로 인해 증가된 Reference Count 덕분에

A의 Reference Count는 0이 되지 않고 따라서 nil로 바뀌지도 않습니다 (Reference Count가 0이되면 ARC에 의해 nil로 바뀝니다.)

과거 objective-c 시절에는 프로그래머가 직접 deallocate 작업을 해주어야하는 수고가 있었지만 Swift는 ARC 덕에 그런 수고를 덜었습니다.

<br/> 

여기서 하나 궁금해지는건 그럼 A는 언제 Reference Count가 0이 되어 메모리에서 없어질까? 인데

이는 B의 수명이 다하는 시점에 Reference Count가 0이 될 것이며 이때 A도 메모리에서 사라지게 될 것입니다.

<br/> 

사실 이 단락에서 적은 것이 ARC와 weak을 사용하는 이유의 전부입니다.

weak을 잘못 사용했을때 자꾸 인스턴스에 nil이 들어있다든가,

<img src="https://raw.githubusercontent.com/Neph3779/Blog-Image/forUpload/img/20220728173103.png" alt="image-20220728173103913" style="zoom:50%;" />

다음과 같은 오류를 마주하게 된다면 인스턴스의 수명에 대해 다시 생각해볼 필요가 있습니다.

아래서 다시 한번 설명하도록 하겠습니다.



## Reference Count

Reference, 즉 참조를 할 때 참조 당하는 쪽의 RC가 늘어납니다.

이때 강한참조(Strong)의 경우에만 RC가 늘어나게 됩니다. 

만약 a라는 변수에 A라는 클래스의 인스턴스를 담는 아래와 같은 코드가 있다고 하면

```swift
var a = A()
```

a의 RC는 1 증가한 상태입니다.

여기서 아래처럼 a1이라는 변수가 a의 주소값을 가져간다면

```swift
var a = A()
var a1 = a
```

a의 RC는 2가 된 상태인거죠.

이처럼 아무런 키워드를 적지 않고 참조를 하게 되면 Strong 참조를 사용한 것입니다.

a는 자신의 수명이 다하더라도 RC가 0이 아니니 계속 메모리에 살아있게 되는 것이죠

<br/> 

단순히 a1이 생존해있는동안 a가 조금 더 살아있는 경우에는 큰 문제가 되지 않습니다.

하지만 아래와 같은 경우라면 조금 이야기가 달라집니다. 



```swift
class Person {
    let name: String
    init(name: String) { self.name = name }
    var apartment: Apartment?
    deinit { print("\(name) is being deinitialized") }
}

class Apartment {
    let unit: String
    init(unit: String) { self.unit = unit }
    var tenant: Person?
    deinit { print("Apartment \(unit) is being deinitialized") }
}

var john: Person?
var unit4A: Apartment?

john = Person(name: "John Appleseed")
unit4A = Apartment(unit: "4A")
```

Person 객체는 Apartment의 인스턴스를 프로퍼티로 가지고

Apartment 객체는 Person의 인스턴스를 프로퍼티로 가집니다.



<img src="https://docs.swift.org/swift-book/_images/referenceCycle02_2x.png" alt="../_images/referenceCycle02_2x.png" style="zoom:50%;" />

이때 Person의 객체와 Apartment 객체가 각각 apartment, tennant라는 프로퍼티에 서로의 인스턴스를 저장해서 사용하다

수명이 다해서 메모리에서 사라져야 하는 시점이 왔어도 메모리에서 사라질 수 없습니다.

왜냐하면 Person의 인스턴스(john)는 Apartment 인스턴스(unit4A)가 가지고 있는 tennant라는 프로퍼티로 인해 RC가 1 올라가있는 상태이고

Apartment의 인스턴스(unit4A)는 Person의 인스턴스(john)이 가지고 있는 apartment라는 프로퍼티로 인해 RC가 1 올라가있는 상태이기 때문입니다.

따라서 ARC는 이들을 자동으로 nil로 전환해주지 않습니다. 

<br/> 

여기서 그럼 john과 unit4A를 직접 nil로 할당해주면 안되느냐와 같은 질문을 할 수 있겠지만

현재 strong reference cycle이 생긴 상태이기 때문에 이 방법으로도 메모리에서 내릴수는 없습니다.

john.apartment와 unit4A.tennant에 각각 nil을 할당해준다면 메모리에서 내릴 수 있겠지만요.



## Weak

우리는 이러한 상황을 방지하기 위해 weak이란 키워드를 사용합니다

weak은  RC를 증가시키지 않습니다. 따라서 현재 해당 인스턴스가 메모리에 올라와있는지 아닌지 알 수 없으니

optional값을 풀어봤을때 nil값이 들어있다면 현재 메모리에서 내려갔거나 아직 해당 weak 변수에 아무것도 들어오지 않은 경우,

값이 있다면 현재 메모리에 올라와있는 경우로 판단할 수 있습니다.

<br/> 

사실 이게 strong, weak의 전부입니다.

하지만 제대로된 이해없이 "결국 결론은 weak을 쓰면 메모리 누수를 막을 수 있다는거군" 라고 생각한다면

필연적으로 아래의 오류를 만나게 될 것입니다. 

<img src="https://raw.githubusercontent.com/Neph3779/Blog-Image/forUpload/img/20220728173103.png" alt="image-20220728173103913" style="zoom:50%;" />

```swift
class A {
    weak var b: B?
    init() {
        b = B()
    }
}

class B {

}
```



위의 예시에서 b라는 weak var에 B의 인스턴스를 넣어주었습니다.

겉으로 보면 이게 왜 문제지 싶겠지만 weak var는 RC가 증가하지 않는다는 점을 눈여겨봐야 합니다.

이해를 돕기 위해 아래의 사진을 첨부합니다.



<img src="https://raw.githubusercontent.com/Neph3779/Blog-Image/forUpload/img/20220728175542.png" alt="../_images/weakReference02_2x.png" style="zoom:50%;" />

위의 그림을 보면 john은 weak 변수, unit4A는 strong 참조를 사용하고 있는것을 알 수 있습니다.

Person의 인스턴스는 RC가 증가하지 않았고 Apartment의 인스턴스는 RC가 증가했겠네요

<br/> 

다시 A, B 클래스의 예시로 돌아와서 이야기하자면

weak var b는 RC를 증가시켜주지 않는 weak reference이므로

B()라는 B의 인스턴스를 할당해주더라도 RC가 0이므로 ARC에 의해 자동으로 nil로 바뀌는 것입니다.

<br/> 

그렇다면 weak var에는 어떤 인스턴스를 넣어야하는 걸까요?

정답은 "이미 RC가 증가되어있는 인스턴스"입니다.



```swift
class A {
    weak var b: B?
    init() {
        let temp = B()
        self.b = temp
    }
}

class B {

}
```

위의 예시처럼 코드를 작성하면 아까 만났던 warning은 피할 수 있습니다.

temp라는 상수에 B()라는 인스턴스를 집어넣어서 temp의 RC가 이미 1이기 때문에

b에다 집어넣어도 문제가 없는 것이죠

<br/> 

물론! 여기서 temp는 init이 끝나는 시점에 수명이 다해서 RC가 내려갈 것이며

b는 다시 nil값이 저장될 것임을 알 수 있습니다.



## Unowned

Unowned를 자주 사용할 일은 없겠지만 개념에 대해 정리해보겠습니다.

Unowned는 RC를 증가시키지 않지만, 값이 반드시 존재하는 것을 보장합니다. (즉, optional이 아닙니다.)

Unowned는 어떤 인스턴스의 수명이 자신보다 길거나 같은 경우에 사용합니다.



```swift
class Customer {
    let name: String
    var card: CreditCard?
    init(name: String) {
        self.name = name
    }
    deinit { print("\(name) is being deinitialized") }
}

class CreditCard {
    let number: UInt64
    unowned let customer: Customer
    init(number: UInt64, customer: Customer) {
        self.number = number
        self.customer = customer
    }
    deinit { print("Card #\(number) is being deinitialized") }
}

var john: Customer
john = Customer(name: "John Appleseed")
john.card = CreditCard(number: 1234_5678_9012_3456, customer: john)
```



CreditCard는 Customer가 살아있는 동안에만 존재가치가 있습니다.

따라서 이를 unowned로 선언하여 strong reference cycle을 방지합니다.



<img src="https://docs.swift.org/swift-book/_images/unownedReference01_2x.png" alt="../_images/unownedReference01_2x.png" style="zoom:50%;" />

위의 그림과 같은 상태에서  john이 nil이 되었을때

weak의 예시에서와 같이 strong reference이 생기지 않고 메모리에서 내려갑니다.

<br/> 

이번 글에서는 조금 헷갈렸던 strong, weak, unowned에 대해 정리해봤습니다.

스토리보드에서 끌어온 UI가 weak var로 선언되던 것은 스토리보드가 만든 인스턴스를 뷰컨트롤러의 변수가 가리킴으로써 뷰컨트롤러가 사라져도

해당 인스턴스가 죽지 않는 메모리 누수가 발생할 수 있기 때문에 자동으로 weak var로 선언되던 것이었습니다.

이와 더불어 escaping closure도 수명주기와 연관지어 내부 구조를 생각해보면 escaping 명시의 여부가 왜 필요한지에 대해서도 알 수 있었습니다.
