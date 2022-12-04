<!-- markdownlint-disable-file MD042 MD037 -->
# 3.7 람다 개선사항

※ gcc compiler 기준으로 정리

## Contents

- [lambda](#lambda)
- [람다의 템플릿 매개변수 지원](#람다의-템플릿-매개변수-지원)
- [this 포인터의 암묵적 복사 방지](#this-포인터의-암묵적-복사-방지)
- [미평가-문맥의-람다와-상태-없는-람다의-기본-생성-및-복사-배정-지원](#미평가-문맥의-람다와-상태-없는-람다의-기본-생성-및-복사-배정-지원)
- [Refs](#refs)

## 요약 (c++20의 람다)

- 람다는 템플릿 매개변수를 가질 수 있음
- 컴파일러는 this 포인터의 암묵적 복사를 검출할 수 있음
- 미평가 문맥에서 람다를 사용할 수 있음
- 상태 없는 람다는 기본 생성과 복사 배정을 지원함

## lambda

- 이름 없는 함수
- format : [ captures ] ( params ) specs requires(optional) { body }
  - captures
    - lambda 안에서 lambda 밖에 있는 변수를 접근하기 위해 사용
    - 4가지 형태를 지원
    - 1. [&]() {/* */} : 외부의 모든 변수들을 레퍼런스로 가져옴 (call-by-reference)
    - 2. [=]() {/* */} : 외부의 모든 변수들을 값으로 가져옴 (call-by-value , const 형태로 가져옴)
    - 3. [=, &x, &y] {/* */} OR [&, x, y] {/* */} : 외부의 모든 변수들을 값/참조로 가져오되, x와 y는 참조/값 으로 가져옴
    - 4. [x, &y, &z]{/* */} : 지정한 변수들을 지정한 형태에 따라 가져옴
  - param
    - 일반 함수의 매개변수와 동일
  - specs requires(optional)
    - specifiers, exception, return type 등을 선택적으로 기입 가능
      - specifiers : constexpr(c++17), consteval(c++20), static(c++23)
      - exception : dynamic exception 또는 noexcept
  - body
    - 함수 내용
- 함수를 객체처럼 활용 가능 (함수 객체와 다르게 class 선언이 불필요)
- runtime에 생성 가능

## 람다의 템플릿 매개변수 지원

- c++11의 형식 있는 람다 (typed lambda)
- c++14의 일반적 람다 (generic lambda)
- c++20의 템플릿 람다 (template 매개변수를 가진 lambda)

```cpp
#include <iostream>
#include <string>
#include <vector>

auto sumInt = [](int fir, int sec) { return fir+sec; };
auto sumGen = [](auto fir, auto sec) { return fir+sec; };
auto sumDel = [](auto fir, decltype(fir) sec) { return fir+sec; };
auto sumTem = []<typename T>(T fir, T sec) { return fir+sec; };

int main() {
    std::cout << "sumInt(2000, 11) : " << sumInt(2000,11) << "\n";
    std::cout << "sumGen(2000, 11) : " << sumGen(2000,11) << "\n";
    std::cout << "sumDel(2000, 11) : " << sumDel(2000,11) << "\n";
    std::cout << "sumTem(2000, 11) : " << sumTem(2000,11) << "\n\n";

    std::string hello="Hello ";
    std::string world="World!";
    // std::cout << "sumInt(hello, world) : " << sumInt(hello, world) << "\n";
    std::cout << "sumGen(hello, world) : " << sumGen(hello, world) << "\n";
    std::cout << "sumDel(hello, world) : " << sumDel(hello, world) << "\n";
    std::cout << "sumTem(hello, world) : " << sumTem(hello, world) << "\n\n";

    std::cout << "sumInt(true, 2010) : " << sumInt(true, 2010) << "\n";
    std::cout << "sumGen(true, 2010) : " << sumGen(true, 2010) << "\n";
    // 2010이 bool 형태로 인식되므로 위 2가지 함수와 다르게 결과가 2가 나옴 : true + true
    std::cout << "sumDel(true, 2010) : " << sumDel(true, 2010) << "\n";
    // std::cout << "sumTem(true, 2010) : " << sumTem(true, 2010) << "\n\n";

    return 0;
}
```

- sumInt
  - C++11
  - 형식이 있는 람다
  - 함수 설명 : int로 변환 가능한 형식만 받음
- sumGen
  - C++14
  - 일반적 람다
  - 함수 설명 : 모든 형식을 받음
- sumDel
  - C++14
  - 일반적 람다
  - 함수 설명 : 두번째 매개변수의 형식은 반드시 첫 형식으로 변환 가능
- sumTem
  - C++20
  - 템플릿 람다
  - 함수 설명 : 첫번째 매개변수와 두번째 매개변수가 반드시 같아야 함

- ※인수 기반 클래스 템플릿 형식 연역 (C++17)
  - 컴파일러가 클래스 템플릿의 형식을 인수들로부터 연역 가능

```cpp
std::vector<int> myVec {1,2,3};
->
std::vector myVec {1,2,3};
```

## this 포인터의 암묵적 복사 방지

- this 포인터의 암묵적 복사를 검출하여 오류를 보고
- this 포인터가 복사를 통해 암묵적으로 람다 안에 갈무리(capture)되면 미정 행동이 발생 가능
- 미정 행동 : 해당 프로그램의 행동에 대해 그 어떤 보장도 없는 상태

```cpp
#include <iostream>
#include <string>

struct LambdaFactory {
    auto foo() const {
        return [=] {std::cout << s << "\n"; };
    }
    std::string s = "LambdaFactory";
    ~LambdaFactory() {
        std::cout << "Good bye\n";
    }
};

auto makeLambda() {
    LambdaFactory lambdaFactory;
    return lambdaFactory.foo();
}

int main() {
    auto lam = makeLambda();
    lam();

    return 0;
}
```

- 멤버함수 foo가 return하는 lambda [=]{ std::cout << s <<\n"; }는 [=]로 인해 암묵적으로 this 포인터의 복사본이 람다 안에 생성됨
- 포인터의 복사본은 가리키는 객체가 유효할 때만 유효함
- 이 포인터 복사본은 makeLambda() 내에서 생성한 lambdaFactory 지역 객체를 가리킴
- lambdaFactory 지역 객체는 makeLambda() 함수 종료 시, 생명주기가 끝남
- 이로 인해, makeLambda() 함수에 return으로 돌려준 foo() 내에 this 포인터는 존재하지 않는 객체를 가리킴
- 기존 c++에서는 컴파일러가 오류를 발생시키지 않았으나, c++20 부터는 해당 경우에 컴파일러가 경고 메시지를 출력함

## 미평가 문맥의 람다와 상태 없는 람다의 기본 생성 및 복사 배정 지원

### 미평가 문맥

- 어떤 대상을 평가(실행 또는 호출) 없이 사용할 수 있는 문맥
- ex) 정의되지 않고 선언만 된 함수를 typeid나 decltype 과 같은 평가되지 않은 피연산자를 받는 연산자에서 사용 / 함수 호출은 불가

### 상태 없는 람다

- 주어진 환경에서 아무것도 갈무리 하지 않는 람다
- 람다 정의의 초기화 대괄호 쌍 [] 안에 아무것도 없는 람다
- auto add = [](int a, int b) { return a+b; };

### 개선된 람다를 이용한 STL 연관 컨테이너 적용

- 내용 추가 에정

## Refs

- [cpprefs-lambda](https://en.cppreference.com/w/cpp/language/lambda)
- [모두의코드-람다함수](https://modoocode.com/196)
