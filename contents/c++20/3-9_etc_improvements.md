<!-- markdownlint-disable-file MD042 MD037 -->
# 3.9 기타 개선사항

※ gcc compiler 기준으로 정리

## Contents

- [volatile](#volatile)
- [구간 기반 for 루프 안의 초기값](#구간-기반-for-루프-안의-초기값)
- [가상 constexpr 함수](#가상-constexpr-함수)
- [char8_t](#char8_t)
- [지역범위 using enum](#비트-필드-멤버의-기본값-초기화)
- [비트 필드 멤버의 기본값 초기화](#비트-필드-멤버의-기본값-초기화)
- [Refs](#refs)

## volatile

- 주어진 객체의 값이 구현이 검출할 수 없는 수단에 의해 변경될 수 있으므로 그러한 객체와 관련된 코드에 공격적인 최적화를 하지 말라는 힌트를 제공
- 컴파일러는 volatile 연산을 제거하거나 순서를 바꿀 수 없음

- 비권장 (폐기 예정) 기능
  - 복합 배정과 전/후 증가/감소에 대한 volatile
  - 함수 매개변수나 반환 형식에 대한 volatile
  - 구조적 바인딩 선언에 대한 volatile

```cpp
// 1)
int neck, tail;
volatile int brachiosaur;
brachiosaur = neck;  // OK : volatile 저장 연산
tail = brachiosaur;  // OK : volatile 적재 연산
// 비권장 : brachiosaur가 한 번 접근될 지 두 번 접근 될 지?
tail = brachiosaur = neck;
// 비권장 : brachiosaur가 한 번 접근될 지 두 번 접근 될 지?
brachiosaur += neck;
// Ok : volatile 적재, 덧셈, volatile 저장
brachiosaur = brachiosaur + neck;

// 2)
// 비권장 : 반환 형식에 대한 volatile은 무의미 함
volatile struct amber jurassic();
// 비권장 : volatile 매개변수는 호출자에 무의미 함 - 함수 안에서만 적용
void trex(volatile short left_arm, volatile short right_arm);
// OK : 포인터는 volatile이 아니고, 포인터가 가리키는 데이터가 volatile
void fly(volatile struct pterosaur* pterandon);

// 3)
struct linhenykus { volatile short forelimb; };
void park(linhenykus alvarezsauroid) {
    // 비권장 : 바인딩이 forelimbs를 복사?
    auto [what_is_this] = alvarezsauroid;  // 구조적 바인딩
}
```

- volatile
  - C/C++ 컴파일러는 자동으로 코드를 최적화
  - 의미 없는 코드의 경우 자동으로 지워버림
  - volatile은 컴파일러가 해당 코드를 최적화하지 않도록 지정하는 옵션

- volatile의 기능
  - 필요없는 코드의 제거
  - 변수의 순서를 고정
  - 캐싱된 데이터를 읽어오지 않음 >> 임베디드 소프트웨어에서 중요

- 일반적으로는 많이 사용하진 않음

## 구간 기반 for 루프 안의 초기값

- initializer in ranged-based for loop

```cpp
int main() {
    for (auto vec = std::vector{1,2,3}; auto v:vec) {
        std::cout << v << " ";
    }
    std::cout << "\n\n";
    for (auto initList = {1,2,3}; auto e:initList) {
        e *= e;
        std::cout << e << " ";
    }
    std::cout << "\n\n";

    using namespace std::string_literals;
    for (auto str = "Hello World"s; auto c: str) {
        std::cout << c << " ";
    }
    std::cout << "\n\n";
}
```

## 가상 constexpr 함수

- constexpr 함수는 가능하면 컴파일 시점에 실행되고, 그렇지 않으면 실행 시점에 실행됨
- constexpr 함수가 실행 시점에서 실행될 수 있다는 것은 가상 함수가 될 수 있다는 의미

```cpp
// virtualConstexpr.cpp

#include <iostream>

struct X1 {
    virtual int f() const = 0;
};

struct X2: public X1 {
    constexpr int f() const override { return 2; }
};

struct X3: public X2 {
    int f() const override { return 3; }
};

struct X4: public X3 {
    constexpr int f() const override { return 4; }
};

int main() { 
    X1* x1 = new X4;
    // 포인터를 통한 virtual dispatch - latebinding 을 사용
    std::cout << "x1->f(): " << x1->f() << std::endl;
    
    X4 x4;
    X1& x2 = x4;
    // 참조를 통한 virtual dispatch를 사용
    std::cout << "x2.f(): " << x2.f() << std::endl;
    
}
```

## char8_t

- C++ 11 : char16_t, char32_t
- C++ 20 : char8_t 가 추가됨 (8 bit) + std::u8string
  - 임의의 UTF-8 코드 단위를 담기에 충분함
  - unsigned char와 크기, 부호여부, 정렬 특성이 동일하지만 다른 type

- char와 char8_t
  - 하나의 char 값은 1바이트 (c++ 표준이 명시적으로 정의하지 않으므로 비트수가 정의되지 않음)
  - char8_t 는 명확한 8비트
  - 단 모든 구현에서 1바이트는 8비트를 의미함

- std::u8string
  - UTF-8 문자열은 char8_t를 사용

```cpp
std::u8string: std::basic_string<char8_t>
u8"Hello World"
```

```cpp
// char8Str.cpp

#include <iostream>
#include <string>

int main() {

    const char8_t* char8Str = u8"Hello world";
    std::basic_string<char8_t> char8String = u8"helloWorld";
    std::u8string char8String2 = u8"helloWorld";
    
    char8String2 += u8".";
    
    // 10
    std::cout << "char8String.size(): " << char8String.size() << std::endl;
    // 11
    std::cout << "char8String2.size(): " << char8String2.size() << std::endl;
    
    char8String2.replace(0, 5, u8"Hello ");
    // 12
    std::cout << "char8String2.size(): " << char8String2.size() << std::endl;
}
```

## 지역범위 using enum

- using enum 선언은 주어진 열거형의 enumerator들을 현재 지역 범위에 도입함
- using enum 선언을 통해 지역 범위에 도입된 enumerator는 범위 없는 enumerator처럼 사용 가능
  - 아래 예제에서는 Color:: 를 붙이지 않고 사용 가능

```cpp
// enumUsing.cpp

#include <iostream>
#include <string_view>

enum class Color {
    red,
    green, 
    blue
};

std::string_view toString(Color col) {
  switch (col) {
    using enum Color;                   // (1) 
    case red:   return "red";           // (2) 
    case green: return "green";         // (2) 
    case blue:  return "blue";          // (2) 
  }
  return "unknown";
}

int main() {

    std::cout << std::endl;
   
    std::cout << "toString(Color::red): " << toString(Color::red) << std::endl;
    
    using enum Color;                                                    // (1)
    
    std::cout << "toString(green): " << toString(green) << std::endl;    // (2) 
    
    std::cout << std::endl;
}  
```

## 비트 필드 멤버의 기본값 초기화

- 비트 필드
  - 컴퓨터 프로그래밍에서 사용되는 자료구조
  - 수많은 인접 컴퓨터 메모리 위치로 구성되어 있음
  - 일련의 비트를 보유하기 위해 할당됨
  - 하나의 비트 또는 여러 비트의 그룹 주소를 참조할 수 있또록 저장됨
  - 비트 필드는 알려진 고정 비트 너비의 정수형을 표현하기 위해 흔히 사용됨

- C++20에서는 비트 필드의 각 멤버를 특정한 기본 값으로 초기화할 수 있음

```cpp
// bitField.cpp

#include <iostream>

struct Class11 {             // (1)
    int i = 1;
    int j = 2;
    int k = 3;
    int l = 4;
    int m = 5;
    int n = 6;
};

struct BitField20 {          // (2)
    int i : 3 = 1;
    int j : 4 = 2;
    int k : 5 = 3;
    int l : 6 = 4;
    int m : 7 = 5;
    int n : 7 = 6;
};

int main () {
    
    std::cout << std::endl;

    // 24
    std::cout << "sizeof(Class11): " << sizeof(Class11) << std::endl;
    // 4
    std::cout << "sizeof(BitField20): " << sizeof(BitField20) << std::endl;
    
    std::cout << std::endl;    
}
```

## Refs

- [volatile and Other Small Improvements in C++20](https://www.modernescpp.com/index.php/volatile-and-other-small-improvements-in-c-20)