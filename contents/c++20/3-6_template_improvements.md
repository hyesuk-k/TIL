# 3.6 Template 개선사항

※ gcc compiler 기준으로 정리

## Contents

- [Template](#template)
- [Refs](#refs)

## Template

- 코드를 찍어내는 틀
- 함수나 클래스 코드를 찍어내듯 생산할 수 있도록 일반화(generic) 시키는 도구
- function overloading

- 장점
  - 서로 다른 타입의 객체가 공통 동작을 공유할 수 있음
  - 소프트웨어의 생선성과 유연성을 높임 (함수 작성 용이, 코드의 재사용)

- 단점
  - 포팅에 취약 (컴파일러 버전 별로 달라짐)
  - 템플릿 관련된 컴파일 오류 메시지 확인이 어려움 (c++ 20 에서 concepts에 의해 향상됨)

```cpp
template <typename T>
template <class T>  // class 보다는 typename을 추천 - typename 과 동일한 의미
template <typename T1, typename T2>
template <typename T, int num>
template <typename T, int num=5>
```

- 아래에 정의되는 클래스에 대해 템플릿을 정의
- 템플릿 인자로 T를 받게되며, T는 반드시 어떠한 타입의 이름임을 명시

- #define
  - 매크로도 타입에 무관하게 동작함
  - 타입 안정성(type-safe)이 확보되지 않음
    - template은 generic type에 실제 적용되는 type을 검사하여 구체화 과정을 거침
    - 매크로는 개발자가 타입을 혼합해서 사용 가능

## 조건부 explicit

- 일부 묵시적 형변환만 허용 가능

- 예) 다양한 형식을 받는 std::variant 멤버

```cpp
class VariantWrapper {
  std::variant<bool, char, int, double, float, std::string> myVariant;
};
```

- 예) 묵시적 형변환을 모두 허용하는 경우

```cpp
struct Implicit {
  template <typename T>
  Implicit(T t) {
    std::cout << t << "\n";
  }
};

int main() {
  std::cout << "\n";
  Implicit imp1 = "implicit";
  Implicit imp2("explicit");
  Implicit imp3 = 1998;
  Implicit imp4(1998);
  std::cout << "\n";

  return 0;
}
```

- 예) 명시적 생성만 유효한 경우

```cpp
struct Explicit {
  template <typename T>
  explicit Explicit(T t) {
    std::cout << t << "\n";
  }
};

int main() {
  std::cout << "\n";
  // Explicit exp1 = "implicit"; > 묵시적 형변환
  Explicit exp2("explicit");
  // Explicit exp3 = 2011;
  Explicit exp4(2011);
  std::cout << "\n";

  return 0;
}
```

- 예) 조건부 explicit

```cpp
struct MyBool {
  template <typename T>
  explicit(!std::is_same<T, bool>::value) MyBoolExplicit(T t) {
    std::cout << typeid(t).name() << "\n";
  }
};

void needBool(MyBool b) { }

int main() {
  std::cout << "\n";
  MyBool myBool1(true);
  MyBool myBool2 = false;  // 묵시적 형변환 - !is_same이 false

  needBool(myBool1);
  needBool(true);   // 묵시적 형변환 - !is_same이 false
  // needBool(5);  // int - !is_same이 true
  // needBool("true");  // string - !is_same이 true
  std::cout << "\n";

  return 0;
}
```

## 비형식(non-type) 템플릿 매개변수

- non-type 매개변수
  - 정수와 열거자 (열거형의 멤버)
  - 객체, 함수, 클래스 멤버를 가리키는 포인터나 참조
  - std::nullptr_t
- C++20 에서는 부동소수점, 리터럴, 문자열 리터러를 비형식 템플릿 매개변수로 사용 가능

```cpp
std::array<int, 5> myVec;
```

### 부동소수점과 리터럴 형식 템플릿 매개변수

- 2가지 조건을 충족하는 리터럴 형식은 non-type 템플릿 매개변수로 사용 가능
  - 모든 기반 클래스와 비정적 데이터 멤버가 공개(public)이고 불변(mutable)
  - 모든 기반 클래스와 비정적 데이터 멤버가 구조적 형식(structural type) 또는 구조적 형식의 배열
- 리터럴 형식에는 반드시 constexpr 생성자가 있어야 함

```cpp
struct ClassType {
  constexpr ClassType(int) {}
};

template <ClassType cl>
auto getClassType() {
  return cl;
}

template <double d>
auto getDouble() {
  return d;
}

int main() {
  auto c1 = getClassType<ClassType(2020)>();
  auto d1 = getDouble<5.5>();
}
```

### 문자열 리터럴 형식 템플릿 매개변수

```cpp
template <int N>
class StringLiteral {
  public:
    constexpr StringLiteral(char const (&str)[N]) {
      std::copy(str, str + N, data);
    }
    char data[N];
};

template <StringLiteral str>
class ClassTemplate {};

template <StringLiteral str>
void FunctionTemplate() {
  std::cout << str.data << "\n";
}

int main() {
  ClassTemplate<"string literal - class"> cls;
  FunctionTemplate<"string literal-function"> cls;

  return 0;
}
```

## refs

- [cpprefs-template](https://en.cppreference.com/w/cpp/language/templates)
- [모두의코드-template](https://modoocode.com/219)
- [const vs constexpr in c++](http://www.vishalchovatiya.com/when-to-use-const-vs-constexpr-in-cpp/)
- [cpprefs-consteval](https://en.cppreference.com/w/cpp/language/consteval)
- [cpprefs-constinit](https://en.cppreference.com/w/cpp/language/constinit)
