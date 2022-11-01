# 3.3 Spaceship operator

※ gcc compiler 기준으로 정리

## Contents

- [3중 비교 연산자?](#3중-비교-연산자)
- [C++20 이전의 순서 판정](#331-c20-이전의-순서-판정)
- [C++20 이후의 순서 판정](#332-c20-이후의-순서-판정)
- [비교 범주](#333-비교-범주)
- [컴파일러 생성 우주 연산자](#334-컴파일러-생성-우주-연산자)
- [표현식 재작성](#335-표현식-재작성)
- [사용자 정의 연산자와 컴파일러 생성 연산자](#336-사용자-정의-연산자와-컴파일러-생성-연산자)
- [Refs](#refs)

## 3중 비교 연산자

* <=>
* 주어진 두 값 A와 B에 대해 3가지 중 하나로 판단 (순서 무관)
    + A < B  : -1
    + A == B : 0
    + A > B  : 1
* spaceship operator || three-way comparison operator
* 3중 비교 연산자를 컴파일러가 자동으로 생성하게 하려면 <b>\<compare> 헤더를 포함</b>해야 함

## 3.3.1 C++20 이전의 순서 판정

* 6가지 비교 연산자 중 하나를 정의했다면, 나머지 관계들도 모두 정의 해야 함

```cpp
#include <iostream>

struct myInt {
    int value;
    explicit MyInt(int val): value{val} {}
    bool operator < (const MyInt& rhs) const {
        return value < rhs.value;
    }
    bool operator == (const MyInt& rhs) const {
        return value == rhs.value;
    }
    bool operator != (const MyInt& rhs) const {
        return !(*this == rhs.value);
    }
    bool operator <= (const MyInt& rhs) const {
        return !(rhs < *this);
    }
    bool operator > (const MyInt& rhs) const {
        return rhs < *this;
    }
    bool operator >= (const MyInt& rhs) const {
        return !(*this < rhs);
    }

};

template <typename T>
constexpr bool isLessThan(const T& lhs, const T& rhs) {
    return lhs < rhs;
}

int main() {
    std::cout << std::boolalpha << '\n';
    MyInt myInt2011(2011);
    MyInt myInt2014(2014);
    std::cout << "isLessThan(myInt2011, myInt2014): "
            << isLessThan(myInt2011, myInt2014) << '\n';    // true;
    return 0;
}
```

## 3.3.2 C++20 이후의 순서 판정

* 3중 비교연산자를 직접 정의
* = default를 이용하면 컴파일러가 6가지 비교 연산자를 자동으로 생성

* MyInt의 <=>에 대해서는 컴파일러가 연역한 반환 형식은 강순서 (strong order)를 지원
* MyDouble의 <=>에 대해서는 컴파일러가 연역한 반환 형식은 부분 순서(partial ordering)를 지원

```cpp
#include <compare>
#include <iostream>

struct MyInt {
    int value;
    explicit MyInt(int val): value{val} {}
    auto operator<=>(const MyInt& rhs) const {
        return value <=> rhs.value;
    }
};

struct MyDouble {
    double value;
    explicit constexpr MyDouble(double val): value{val} {}
    auto operator<=>(const MyDouble&) const = default;
};

template <typename T>
constexpr bool isLessThan(const T& lhs, const T& rhs) {
    return lhs < rhs;
}

int main() {
    std::cout << std::boolalpha << '\n';

    MyInt myInt1(2011);
    MyInt myInt2(2014);
    std::cout << "isLessThan(myInt1, myInt2): "
            << isLessThan(myInt1, myInt2) << '\n';    // true;

    MyDouble myDouble(2011);
    MyDouble myDouble2(2014);
    std::cout << "isLessThan(myDouble1, myDouble2): "
            << isLessThan(myDouble1, myDouble2) << '\n';    // true;

    return 0;
}
```

### 포인터 자동 비교

* 3중 비교 연산자는 포인터들에도 작동함
* 단, 포인터가 가리키는 객체가 아닌 포인터 자체(객체의 주소)를 비교하므로 사용 시 주의해야 함

```cpp
#include <iostream>
#include <compare>
#include <vector>

struct A {
    std::vector<int>* pointerToVector;
    auto operator <=> (const A&) const = default;
};

int main() {
    std::cout << std::boolalpha;
    A a1{new std::vector<int>()};
    A a2{new std::vector<int>()};

    std::cout << (a1==a2) << "\n";
    // std::vector<int>*의 주소를 비교하여 false
    return 0;
}
```

## 3.3.3 비교 범주

| 비교 범주 | 관계 연산자 | 동치 | 비교 가능 |
|:--------:|:----------:|:----:|:--------:|
|강 순서|성립|성립|성립|
|약 순서|성립| |성립|
|부분 순서|성립| | |

* comparison category는 3가지
* 강 순서(strong order), 약 순서(weak order), 부분 순서(partial order)
* 형식 T의 비교 범주를 결정하는 밥업
    + 1. T가 여섯 가지 비교 연산자 (==, !=, <, <=, >, >=)를 모두 지원한다 (관계 연산자 성질)
    + 2. 동치 (equivalent) 값들은 모두 구별할 수 없다 (동치 성질)
    + 3. T의 모든 값이 비교 가능이다. 즉, T의 임의의 값 a와 b에 대해 반드시 a < b, a==b, a > b 중 하나가 참 (비교 가능 성질)

* $강 순서 \in 약 순서 \in 부분 순서$
    + 강 순서를 지원하는 형식은 암묵적으로 약 순서와 부분 순서를 지원한다.
    + 약 순서를 지원하는 형식은 암묵적으로 부분 순서를 지원한다.

* 3중 비교 연산자의 반환 형식을 auto로 지정하면, 실제 반환 형식은 비교할 객체의 기반 객체와 멤버 하위 객체, 멤버 배열 요소들의 공통 비교 범주에 따라 결정됨


* equivalent
    + equiv(a, b), an expression equivalent to !comp(a, b) && !comp(b, a) [참고](https://en.cppreference.com/w/cpp/named_req/Compare)


## 3.3.4 컴파일러 생성 우주 연산자

* 컴파일러에 의해 자동으로 3중 비교 연산자를 생성하려면 \<compare> header 필요
* 컴파일러에 의해 자동으로 생성된 3중 비교 연산자는 암묵적으로 constexpr와 noexcept가 적용됨
    + constexpr은 기본적으로 const와 같은 상수를 의미
    + noexcept는 해당 함수는 예외를 발생시키지 않음을 의미
* 컴파일러에 의해 자동으로 생성된 3중 비교 연산자는 어휘순 비교를 수행
    + 어휘순 비교 (lexicographical comparison; 사전순 비교)
    + 모든 기반 클래스를 왼쪽에서 오른쪽으로 훑으면서 각 클래스의 비 정적(non-static) 멤버들을 순서대로 비교
    + c++20에서 컴파일러가 생성한 ==와 !==는 행동 방식이 다름 (성능상의 이유)


```cpp
#include <compare>
struct Basics {
    int a;
    float b;
    double c;
    char d;
    auto operator<=>(const Basics&) const = default;
};

struct Arrays{
    int ai[1];
    char ac[2];
    float af[3];
    double ad[2][2];
    auto operator<=>(const Array&) const = default;
};

struct Bases : Basics, Arrays {
    auto operator<=>(const Bases&) const = default;
};

int main() {   
    // 집합체 초기화 : class, struct, union 멤버를 직접 초기화 가능 (단, 모든 멤버가 public인 경우에만)
    constexpr Base a = {{0, 'c', 1.f, 1.},      //Basics
                        {{1}, {'a', 'b'}, {1.f, 2.f, 3.f},  // Arrays
                        {{1., 2.}, {3., 4.}}}
                        };
    constexpr Base b = {{0, 'c', 1.f, 1.},      //Basics
                        {{1}, {'a', 'b'}, {1.f, 2.f, 3.f},  // Arrays
                        {{1., 2.}, {3., 4.}}}
                        };
    
    static_assert(a == b);
    static_assert(!(a != b));
    static_assert(!(a < b));
    static_assert(a <= b);
    static_assert(!(a > b));
    static_assert(a >= b);
                        
    return 0;
}
```

* 최적화된 == 연산자와 != 연산자
    + 문자열이나 벡터와 같은 형식에 대해서는 추가적인 최적화를 적용할 여지가 있음
    + 길이를 이용하여 컴파일러가 생성한 3중 비교 연산자보다 더 빠르게 == 또는 !=를 구현 가능
    + 기존 컴파일러는 모든 값을 마지막까지 비교하므로 길이를 이용한 비교보다 느림
    + P1185R2를 적용한 버전의 경우, 길이를 이용하여 먼저 비교함
        - [gcc 10 이상](https://runebook.dev/ko/docs/cpp/compiler_support/20)

    

## 3.3.5 표현식 재작성

* 컴파일러는 a < b와 같은 표현식을 만나면 3중 비교 연산자를 사용하여 재작성 (rewriting)
    + a < b -> (a <=> b) < 0
* < 를 포함한 6가지 비교 연산자에 대해 모두 적용
* rewriting 규칙
    + a OP b는 (a <=> b) OP 0
    + a에서 형식 b로 변환이 안되는 경우 : 0 OP (b <=> a) 와 같이 표현

* rewritten expression의 강점?
    + <=> 의 결과는 binary result
    + 관계형 연산의 관점에서 순서를 표현가능

<details>
  <summary>발췌</summary>

You may find yourself asking why this rewritten expression is valid and correct.
The correctness of the expression actually stems from the semantics the spaceship operator provides.
The <=> is a three-way comparison which implies that you get not just a binary result, but an ordering (in most cases) and if you have an ordering you can express that ordering in terms of any relational operations. 
A quick example, the expression 4 <=> 5 in C++20 will give you back the result std::strong_ordering::less. 
The std::strong_ordering::less result implies that 4 is not only different from 5 but it is strictly less than that value, this makes applying the operation (4 <=> 5) < 0 correct and exactly accurate to describe our result.

</details>

## 3.3.6 사용자 정의 연산자와 컴파일러 생성 연산자

* 사용자 정의 연산자 : 여섯 가지 비교 연산자 중 일부를 개발자가 직접 정의
* 컴파일러 생성 연산자 : 우주선 연산자를 이용하여 컴파일러가 자동으로 생성
* 2가지 종류의 연산자가 공존하는 경우...
    + 사용자 정의 연산자를 우선 사용함

# refs

* [modern c++ : 3way op](https://www.modernescpp.com/index.php/c-20-the-three-way-comparison-operator)
* [modern c++ : more details](https://www.modernescpp.com/index.php/c-20-more-details-to-the-spaceship-operator)
* [modern c++ : 최적화된 비교](https://www.modernescpp.com/index.php/c-20-fast-comparison-with-the-spaceship-operator)
* [cppref default comparisons](https://en.cppreference.com/w/cpp/language/default_comparisons)
* [왜 rewriting expression을 사용하는가](https://devblogs.microsoft.com/cppblog/simplify-your-code-with-rocket-science-c20s-spaceship-operator/#rewriting-expressions)