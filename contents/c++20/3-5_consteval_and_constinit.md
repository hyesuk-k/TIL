# 3.5 consteval과 constinit

※ gcc compiler 기준으로 정리

## Contents

- [constXXX지시자](#constxxxx)
- [consteval](#consteval)
- [constinit](#constinit)
- [Refs](#refs)

## constXXXX

* consteval
    + compile-time에 실행되는 함수를 제공

* constinit 
    + compile-time에 변수가 초기화되도록 보장

* const와 constexpr
    + 할당된 후 값을 변경할 수 없음
    + const
        - runtime constant
        - compile-time OR runtime에 값이 결정될 수 있음        - 
    + constexpr (since C++11)
        - compile-time constant expression
        - compile-time에만 값(변수/함수 반환)이 결정될 수 있음
        - 변수명으로 초기화 불가능 (array의 크기)
        - macro 대신 사용 가능 - 함수 앞에 constexpr이 붙는 경우 inline 함수로 암시됨
        - [literals](https://en.cppreference.com/w/cpp/named_req/LiteralType) : scalar (int, float, ...), reference, array 등

```cpp
int getRandomInt() {
    return rand() % 10;
}

int main() {
    const int varConst = getRandomInt();    // ok
    constexpr int varConstE = getRandomInt();  // error - call to non-'constexpr' function
    return 0;
}
```

## consteval

* 즉시 함수(immediate function)를 생성하는 지정자, 해당 함수는 compile-time constant
* 소멸자, 메머리 할당 또는 재할당 함수에는 consteval keyword 사용 불가
* 함수 선언 시, consteval/ constinit/ constexpr 중 하나의 keyword만 사용 가능
* consteval keyword를 사용한 함수는 암묵적으로 inline 함수


* constexpr 함수와 consteval 함수의 차이
```cpp
constexpr int mulTenC(int v) {
    return v*10;
}

consteval int mulTenE(int v) {
    return v*10;
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

* 변수가 compile-time에 초기화 되도록 함
    + 기존 static 변수는 초기화 순서가 정해져있지 않음
    + static 변수의 초기화 시점은 compile-time OR run-time에 해당 변수 선언 부 호출 시

* const와 같이 사용 가능 (constinit const Var OR const constinit Var)

* static 또는 thread storage를 가지는 변수를 선언
    + static initialization (전역변수, 정적 변수)
    + [thread storage duration](https://en.cppreference.com/w/c/language/thread_storage_duration) (since C++11)

* constinit 선언된 변수에 동적 초기화 사용 불가

```cpp

#include <iostream>

consteval int sqr(int n) {
    return n * n;
}

    constexpr auto res1 = sqr(5);                  
    constinit auto res2 = sqr(5);                 

int main() {

    std::cout << "sqr(5): " << res1 << std::endl;
    std::cout << "sqr(5): " << res2 << std::endl;
   
    constinit thread_local auto res3 = sqr(5);     
    std::cout << "sqr(5): " << res3 << std::endl;

}
```

## refs

* [2 new keyword](https://www.modernescpp.com/index.php/c-20-consteval-and-constinit)
* [const vs constexpr in C++](http://www.vishalchovatiya.com/when-to-use-const-vs-constexpr-in-cpp/)
* [cpprefs-consteval](https://en.cppreference.com/w/cpp/language/consteval)
* [cpprefs-constinit](https://en.cppreference.com/w/cpp/language/constinit)


* [storage duartion](https://en.cppreference.com/w/cpp/language/storage_duration)
    + automatic, static, dynamic, thread_local(since c++11)
        + automatic : 함수/블록 안에 들어갈 때 할당, 나올 때 해제. 함수 매개변수/로컬변수
        + static : 프로그램 실행 시간동안 유지되는 변수, static 변수들
        + dynamic : free OR heap 영역, new/delete로 관리되는 동적 할당 메모리
        + thread_local : storage-class specifier를 통해 구현 / thread가 살아있는 동안 유지