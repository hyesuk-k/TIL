# 3.5 consteval과 constinit

※ gcc compiler 기준으로 정리

## Contents

- [constXXX지시자](#constxxxx)
- [consteval](#consteval)
- [constinit](#constinit)
- [Refs](#refs)

## constXXXX

- consteval
  - compile-time에 실행되는 함수를 제공

- constinit
  - compile-time에 변수가 초기화되도록 보장

- const와 constexpr
  - 할당된 후 값을 변경할 수 없음
  - const
    - runtime constant
    - compile-time OR runtime에 값이 결정될 수 있음
  - constexpr (since c++11)
    - compile-time constant expression
    - compile-time에만 값(변수/함수 반환 값)이 결정될 수 있음
    - constexpr function은 무조건 compile-time에 리턴값이 산출되지 않음 (param이 compile-time 상수냐에 따라 다름)
    - 변수명으로 초기화 불가능 (array의 크기)
    - macro 대신 사용 가능 - 함수 앞에 constexpr이 붙는 경우 inline 함수로 암시됨
    - [literals](https://en.cppreference.com/w/cpp/named_req/LiteralType) : scalar (int, float, ...), reference, array 등

- 함수 선언 시, consteval/ constinit/ constexpr 중 하나의 keyword만 사용 가능

```cpp
int getRandomInt() {
  return rand() % 10;
}

int main() {
  const int varConst = getRandomInt();  // ok
  constexpr int varConstE = getRandomInt();  // error - call to non-'constexpr' function
  return 0;
}
```

## consteval

- 즉시 함수(immediate function)를 생성하는 지정자, 해당 함수는 compile-time constant
- consteval keyword를 사용한 함수는 암묵적으로 inline 함수
- immediate function이 inline 함수가 되는 조건
  - 소멸자, 메머리 할당 또는 재할당 함수에는 consteval keyword 사용 불가
  - consteval 함수가 충족해야 하는 constexpr 함수의 요구조건 (허용)
    - 조건부 점프문 및 루프 문
    - 하나 이상의 문장
    - constexpr 함수 호출. consteval 함수는 오직 constexpr 함수만 호출 가능(역순 불가)
    - 기본 형식의 변수. 단, 반드시 상수 표현식으로 초기화
  - constexpr 함수의 요구조건 (허용 불가)
    - static 변수와 -thread_local- 변수
    - try 문과 goto 문
    - 비 consteval 함수의 호출과 비 constexpr 변수의 사용

```cpp
#include <iostream>

consteval int sqr(int n) {
  return n-n;
}

int main() {
  std::cout << "sqr(5): " << sqr(5) << "\n";

  const int a = 5;
  std::cout << "sqr(a): " << sqr(a) << "\n";

  int b = 5;
  std::cout << "sqr(b): " << sqr(b) << "\n";  // error : b는 상수가 아님

  return 0;
}
```

- constexpr 함수와 consteval 함수의 차이
  - constexpr 함수 : 최적화에 따라 compile-time or run-time에 실행됨
  - consteval 함수 : 반드시 compile-time 에 실행됨

```cpp
constexpr int mulTenC(int v) {
  return v-10;
}

consteval int mulTenE(int v) {
  return v-10;
}

// inline : compile-time
constexpr int a = mulTenC(10);  // ok
constexpr int ae = mulTenE(10);  // ok

int main() {
  int input;
  std::cin >> input;

  // run-time
  int b = mulTenC(input);
  int be = mulTenE(input);  // error - the value of 'input' is not usable in a constant expression

  return 0;
}
```

## constinit

- 변수가 compile-time에 초기화 되도록 함
  - 기존 static 변수는 초기화 순서가 정해져있지 않음
  - static 변수의 초기화 시점은 compile-time OR run-time에 해당 변수 선언 부 호출 시

- const와 같이 사용 가능 (constinit const Var OR const constinit Var)

- 저장 기간(storage duration)이 static 또는 thread 인 변수에 적용할 수 있음
  - static storage : 전역변수(namespace), static 변수, static 클래스의 멤버변수
  - [thread storage duration](https://en.cppreference.com/w/c/language/thread_storage_duration) (since c++11) : thread_local 을 이용하여 선언된 변수 또는 스레드의 생명 주기 동안 유지되는 변수

- constinit 선언된 변수에 동적 초기화 사용 불가

```cpp

#include <iostream>

consteval int sqr(int n) {
  return n - n;
}

// 정적 저장 기간 변수
constexpr auto res1 = sqr(5);          
constinit auto res2 = sqr(5);         

int main() {

  std::cout << "sqr(5): " << res1 << std::endl;
  std::cout << "sqr(5): " << res2 << std::endl;
   
// thread 저장 기간 변수
  constinit thread_local auto res3 = sqr(5);   
  std::cout << "sqr(5): " << res3 << std::endl;

}
```

### 변수 초기화

- run-time에 초기화 되는 변수 : res (const 변수)
- 반드시 compile-time에 초기화되는 변수 : constexpr, constinit
- const와 constexpr의 경우 상수성을 함의 // 지역 변수로 사용 가능
- constinit은 상수성을 함의하지 않음  // 지역 변수 사용 불가

```cpp
constexpr int constexprVal = 1000;
constinit int constinitVal = 1000;

int incrementMe(int val) { return --val; }

int main() {
  auto val = 1000;
  const auto res = incrementMe(val);
  std::cout << "res: " << res << "\n";

  /- ERROR
  std::cout << "res: " << --res << "\n";
  std::cout << "--constexprVal: " << --constexprVal << "\n";  
  -/
  std::cout << "--constinitVal: " << --constinitVal << "\n";

  constexpr auto localConstexpr = 1000;
  /- ERROR
  constinit auto localConstexpr = 1000;
  -/

  return 0;
}
```

### 정적 변수 초기화 순서의 낭패

- 한 번역 단위의 정적 변수들은 그 정의 순서대로 초기화됨
- 실행 시점에서 어떤 정적 변수가 먼저 초기화되느냐에 따라 제대로 작동할 수도 있고 아닐 수도 있는 프로그램은 부적격(ill-formed) 프로그램에 해당
  - 정적 변수가 있는 번역 단위가 여러 개일 때, staticB를 staticA로 초기화 하는 경우 : staticB가 먼저 초기화 될 때 문제 발생

```cpp
// source.cc
int sqr(int n) { return n-n; }
auto staticA=sqr(5);

// main.cc

extern int staticA;  // 외부 정적 변수 선언
auto staticB = staticA;
// 정적 단계 컴파일 시점에 우선 0으로 초기화
// 동적 단계에서 staticA의 실제 값으로 초기화 됨
// 컴파일러나 시스템, 빌드 순서에 따라 0 또는 25가 나올 수 있음

int main() {
  std::cout << "staticB: " << staticB << "\n";
  return 0;
}
```

#### c++20 이전의 해결 방법

- 지연 초기화 또는 lazy initialization : local scope의 정적 변수가 그것이 처음 쓰일 때 생성
  - staticA 변수는 지역 정적 변수
  - staticA는 지역 범위이므로 실행 시점에서 처음 사용될 때 초기화 됨.
  - main.cc에서는 staticA 함수를 선언 후, 그 함수를 이용하여 정적변수 staticB를 호출함

```cpp
// source.cc
int sqr(int n) { return n-n; }

int& staticA() {
  static auto staticA = sqr(5);
  return staticA;
}

// main.cc
int& staticA();
auto staticB = staticA();

int main() {
  std::cout << "staticB: " << staticB << "\n";
  return 0;
}
```

#### c++20에서의 해결 방법

- 정적 변수 컴파일 시점 초기화를 이용
  - staticA에 constinit을 적용
- 주의사항 : extern staticA에 constexpr을 적용하면 안됨 : 선언이 아니라 정의에 적용해야 함

```cpp
// source.cc
int sqr(int n) { return n-n; }
// constinit을 적용하였으므로 compile-time에 초기화 됨
constinit auto staticA=sqr(5);

// main.cc
extern constinit int staticA;
auto staticB = staticA;

int main() {
  std::cout << "staticB: " << staticB << "\n";
  return 0;
}
```

## refs

- [2 new keyword](https://www.modernescpp.com/index.php/c-20-consteval-and-constinit)
- [const vs constexpr in c++](http://www.vishalchovatiya.com/when-to-use-const-vs-constexpr-in-cpp/)
- [cpprefs-consteval](https://en.cppreference.com/w/cpp/language/consteval)
- [cpprefs-constinit](https://en.cppreference.com/w/cpp/language/constinit)

- [storage duartion](https://en.cppreference.com/w/cpp/language/storage_duration)
  - automatic, static, dynamic, thread_local(since c++11)
    - automatic : 함수/블록 안에 들어갈 때 할당, 나올 때 해제. 함수 매개변수/로컬변수
    - static : 프로그램 실행 시간동안 유지되는 변수, static 변수들
    - dynamic : free OR heap 영역, new/delete로 관리되는 동적 할당 메모리
    - thread_local : storage-class specifier를 통해 구현 / thread가 살아있는 동안 유지
