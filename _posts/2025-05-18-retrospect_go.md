---
title: Go에 관하여
description: >-
  go의 코드 구조에 대한 회고록
author: jay
date: 2025-05-18 20:55:00 +0800
categories: [Retrospect, Geeson]
tags: [golang]
pin: false
media_subpath: ''
---

## golang의 리시버에 관하여

geeson 프로젝의 인증 서버를 go의 gin 라이브러리를 사용해서 구현하다 보니 인터페이스를 구현하고 그를 구현하는 과정에 리시버의 사용에 대해서 혼동이 왔다.

그에 관한 회고 셉션이다.

### 1. 리시버의 구현과 동작 원리

어떤 코드 한줄에 꽂혀서 생각을 하게 되었다. 바로 아래의 코드이다.

``` go
var _ repository.UserRepository = (*UserRepository)(nil)
```

자바에서는 위의 코드는 이상하게 느껴진다. repository.UserRepository는 값타입이고 *UserRepositorty는 포인터 타입이다. 그럼 값타입에 포인터를 넣는다고? 이러한 의문이 드는 것이다.
이 의문에는 두가지 오해가 담겨져 있었다.
1. 인터페이스 자체(값)를 가리키는 포인터(인터페이스는 별도의 참조 타입(reference type))
2. Go의 특징 중 하나는 untyped nil 상수를 모든 포인터 타입(뿐만 아니라 map, slice, chan, func, interface 타입 등)으로 변환할 수 있다는 점
   - var a2 MyInterface = &b1 이런식의 구현도 가능한 것이다.(물론 아래와 같은 제약사항이 있다.)
3. 리시버에서 해당 변수를 사용할때는 구조체 멤버를 대상의 얕은 복사가 이루어지지만 초기화시에는 주소값 대상의 얕은 복사가 이루어진다.(즉 같은 메모리를 가리킨다.)

```go
type MyInterface interface {
    Foo()
}

// 값 리시버로 구현한 경우
type Impl1 struct{}
func (Impl1) Foo() { fmt.Println("Impl1") }

// 포인터 리시버로 구현한 경우
type Impl2 struct{}
func (*Impl2) Foo() { fmt.Println("Impl2") }

func main() {
    // 1) 값 리시버만 있을 때: 값·포인터 모두 인터페이스 구현
    var b1 Impl1
    var a1 MyInterface = b1    // OK: Impl1 implements MyInterface
    var a2 MyInterface = &b1   // OK: *Impl1 also has Foo()

    // 2) 포인터 리시버만 있을 때: 포인터만 인터페이스 구현
    var b2 Impl2
    // var a3 MyInterface = b2 // ✗ 컴파일 오류: Impl2에 Foo()가 없음
    var a4 MyInterface = &b2   // OK: *Impl2 implements MyInterface
}
```

### 2. 값타입 리시버와 포인터 타입 리시버

값 리시버에 포인터를 넘기는 코드는 Go가 편리하게 자동으로 역참조해 주긴 하지만, 읽는 사람 입장에선 “이게 원본을 바꾸는 건가, 복사본을 바꾸는 건가?” 헷갈릴 수 있을 것 같음

따라서 다음의 원칙을 준수해야함

1. 변경이 필요한 메서드는 포인터 리시버로 통일
2. 불변·작은 데이터엔 값 리시버만 쓰기
   - 단순 조회, 계산용 메서드만 있고 구조체 크기가 작다면
3. 한 타입에 값 리시버와 포인터 리시버를 섞지 않기
4. 인터페이스 구현용 어설션에도 일관성 유지
   - 포인터 리시버 메서드만 있다면 반드시 *T를,
   - 값 리시버만 있으면 T 또는 *T 어느 쪽이든 허용되지만 한 쪽으로 통일
5. 코드 리뷰나 문서에 리시버 전략 명시하기

만약 리시버를 섞어서 사용하면 다음과 같은 현상이 생긴다.

```go
type T struct{ X int }

// 값 리시버 메서드
func    (t T) Val()    { /*…*/ }
// 포인터 리시버 메서드
func    (t *T) Ptr()   { /*…*/ }
```

**호출 가능 대상**
```go
var v T
var p *T = &v

v.Val()    // OK: T에 Val이 있음
v.Ptr()    // ✗ 컴파일 오류: T에는 Ptr이 없음

p.Val()    // OK: *T에도 Val이 있음 (값 리시버는 포인터에도 자동으로 역참조 해서 적용)
p.Ptr()    // OK: *T에 Ptr이 있음
```


**인터페이스 구현 여부**
```go
type I interface {
  Val()
  Ptr()
}

// ↓ 컴파일 에러! *T가 I를 구현하지 못함을 검증
var _ I = &T{}  
// 이유: &T{}(type *T)의 메서드 집합에는 Ptr만 있고 Val이 빠졌다고 생각하기 쉽지만,
// 실제로는 *T에 Val도 포함되므로 다음은 OK입니다:

var _ I = &T{}  // OK: *T 메서드 집합은 {Val, Ptr}
// 다만…
var _ I = T{}   // ✗ 컴파일 오류: T에는 Ptr()가 없어서 I를 구현하지 못함

```

## goland의 인터페이스

### 1. 인터페이스란

인터페이스는 별도의 참조 타입(reference type)

Go에서 인터페이스는 interface 키워드로 정의된 고유의 타입 계열로, 포인터(*T)나 맵·슬라이스와 마찬가지로 “값이 아닌 값을 가리키는” 성격을 가집니다.

컴파일러 관점에선 인터페이스 자체가 포인터 타입은 아니지만, 내부 구현(두 워드 구조)을 보면 포인터처럼 동작하는 부분이 있습니다.

> interface{}는 “어떤 타입이든 담을 수 있는 범용 참조”로,
실제 값이 들어오면 (itab, data) 쌍으로 저장되는 별도의 내부 표현을 갖고 있습니다.

### 2. 인터페이스와 메모리

인터페이스 변수에 값을 할당한다는 건, 메모리 관점에서 크게 두 단계로 일어납니다.

값 복사(또는 포인터 저장) 및 타입 정보 결정

할당하려는 값이 포인터 타입이라면, 그 포인터 값(메모리 주소) 자체를 그대로 사용합니다.

할당하려는 값이 값 타입(구조체, 기본형 등)이라면, 런타임이 그 값을 담을 메모리(보통 힙에 할당된 복사본)를 만들고, 그 주소를 저장합니다.

인터페이스 내부에 2워드(word) 저장
Go 인터페이스 값은 내부적으로 두 개의 machine word로 구성됩니다.

| itab/type | data ptr |
| --- | --- |

**첫 번째 워드(itab 또는 type):**

비어 있는 인터페이스(interface{})인 경우엔 단순히 타입 디스크립터를 가리키는 포인터 메서드가 선언된(non-empty) 인터페이스인 경우엔 “타입+메서드 테이블” 정보(itab)를 가리킵니다.

**두 번째 워드(data ptr):**

포인터 타입 값을 저장했다면 그 포인터 값 타입 값을 저장했다면, 복사된 값이 담긴 메모리 블록의 주소


**따라서** 

“인터페이스 할당”은 단순히 변수 하나에 값을 쓰는 게 아니라, 필요 시 값을 복제(얕은 복사)하고 메모리를 확보하고, 인터페이스 구조체 두 워드에 타입·값 정보를 세팅하는 작업입니다.

```go
type MyInterface interface {
    Foo()
}
type Impl struct { X int }

func (i *Impl) Foo() {}

// 1) 포인터 값 할당
var p = &Impl{X: 10}
var a MyInterface = p
// → a.itab = ptr to (Impl's itab)
// → a.data = p         (same &Impl{10})

// 2) 값 타입 할당 (값 리시버라도 동작 동일)
type Impl2 struct { Y int }
func (i Impl2) Foo() {}

// 값 타입 복사본 생성
var v = Impl2{Y: 20}
var b MyInterface = v
// → 런타임이 new(Impl2) 하여 복사본 할당
// → b.itab  = ptr to (Impl2's itab)
// → b.data  = ptr to that new Impl2 copy
```
