# 3.1 Concepts

## Contents

- [Concepts 란?](#concepts-란)
- [Motivation](#motivation)
- [Advantages](#advantages)
- [Concept의 사용](#concept의-사용)
- [Concept 적용 방법](#concepts를-적용하는-방법)
- [미리 정의된 concepts](#미리-정의된-concepts)


## Concepts 란?

* cppreference : [concepts](https://en.cppreference.com/w/cpp/language/constraints)
* [concept wiki](https://ko.wikipedia.org/wiki/%EC%BD%98%EC%85%89%ED%8A%B8_(C%2B%2B))

* C++20부터 지원
* 템플릿의 확장 기능
* 템플릿에서 사용하는 매개변수에 제약(조건)을 정의
* concepts로 정의된 제약을 만족하는 경우에만 템플릿을 이용하여 함수를 생성 가능


* concept에 사용되는 키워드
    + _concept_
    + _requires_


## Motivation

* 잘못된 접근 방식으로 템플릿을 사용하는 경우 방지하기 위함

1. c++의 암묵적 형변환 (implicit conversion) 으로 인한 문제
    1-1. narrowing cnoversion (좁히기 변환;축소 변환)
        데이터의 정밀도가 손실되는 변환 (ex: double -> int)
    1-2. 정수 승격
        2개의 bool type을 이용하여 연산 시, int 형으로 변환됨

```cpp
template <typeame T>
auto add(T first, T second) {
    return first + second;
}
 int main() {
    add(true, false);
    return 0;
}
```

2. 일반적인 접근 방식으로 인한 문제
    2-1. sort는 (RandomIt first, RandomIt last) 로 first부터 last까지의 원소를 오름차순 정렬
    2-2. first와 last는 임의 접근 반복자 (RandomIterator)여야 함
    2-3. 따라서 순차 접근 반복자를 사용하는 list의 경우, sort로 정렬 수행 불가
```cpp
#include <algorithm>
#include <list>

int main() {
    std::list<int> mlist{1,10,2,3,5};
    std::sort(mlist.begin(), mlist.end());
    return 0;
}
```


## Advantages

* 템플릿 매개변수에 대한 요구조건이 인터페이스의 일부가 됨
* concepts에 기반한 함수 중복 적재 또는 클래스 템플릿의 특수화
* 함수/클래스 템플릿 외에 클래스의 일반적인 함수에도 적용 가능
* 템플릿 매개변수에 대한 요구조건들을 이용하여 컴파일러가 개선된 오류 메시지 생성 가능 (분석 용이)
* 다른 사람들이 미리 정의한 concepts 활용 가능
* 함수 선언에 concepts를 사용 시, 그 함수는 자동으로 함수 템플릿이 됨
* auto와 concepts의 사용법이 통합됨
    + c++20 에서는 함수 매개변수에 auto 사용 가능 ()

```cpp
template <auto T>
auto sum(auto first, auto second) {
    return first + second;
}
```

## Concept의 사용

### concepts 정의 방법

```cpp
#include <concepts>

template <template-param-list; typedef T or Class T or Arg list...>
concepts concept-name = constraint-expression;
```

### Concepts clause

* concepts 선언 예제
* concept를 이용한 conecepts 가능 

```cpp
#include <concepts>

template<typename T>
concept underHundred = std::integral<T> && sizeof(T) <= 100;
```


## concepts를 적용하는 방법

1. requires 절
2. 후행 (trailing) requires 절
3. 제약이 있는 템플릿 매개변수
4. 단축 함수 템플릿

### requires 절 

* _requires_ 키워드로 시작
* 템플릿 매개변수나 함수 선언에 대한 요구조건(requirement) 또는 제약(constraint)을 서술
* _requires_ 키워드 다음 올 수 있는 형태
    + 하나의 명명된 concepts
    + 논리곱(&&) 또는 논리합(||)
    + compile-time predicate (컴파일 시점 술어)

```cpp
#include <iostream>

template<unsigned int i>
// compile-time predicate 사용하는 경우
requires (i <= 20)
int sum(int j) {
    return i+j;
}

int main() {
    // i 가 20 이하인 경우 정상 동작, sum<23>(2000)의 경우, i가 20을 초과하므로 컴파일 에러 발생됨
    std::cout << "sum<20>(2000): " << sum<20>(2000) << "\n";
    return 0;
}
```

### Trailing requires 절

* 함수 뒤에 _requires_ 를 이용하여 concepts를 붙임

```cpp
#include <iostream>

template<typename T>
auto sum(T first, T second) requires std::integral<T> {
    return first + second;
}
```

### Constrained template parameter

* template 안에 concept를 넣음

```cpp
#include <iostream>

template<std::integral T>
auto sum(T first, T second) {
    return first + second;
}

```

### Abbreviated function template

* 단축 함수 템플릿
* 함수 안에 concepts를 추가하며, template 대신 _auto_ 키워드를 이용

```cpp
#include <iostream>

auto sum(std::integral auto first, std::integral auto second) {
    return first + second;
}
```

### 여러 concept를 동시에 적용하여 사용

* find의 iterator Iter와 value V는 아래 요구조건을 만족해야 함
    + Iter는 반드시 input_iterator여야 함
    + Val은 반복자의 값 형식과 equality_comparable  비교가 가능한 형태여야함
```cpp
template <typename Iter, typename Val>
requires std::input_iterator<Iter>
    && std::equality_comparable<Value_type<Iter>, Val>
Iter find(Iter b, Iter e, Val v)
```

## 미리 정의된 concepts

* input_iterator 나 equality_comparable와 같은 미리 정의된 concepts

* #include \<concepts> 
    + [Concepts library](https://en.cppreference.com/w/cpp/concepts)
* #include \<iterator>
    + [Iterator library](https://en.cppreference.com/w/cpp/iterator#C.2B.2B20_iterator_concepts)
    + [Algorithm library](https://en.cppreference.com/w/cpp/iterator#Algorithm_concepts_and_utilities)
* #include \<ranges>
    + [Range library](https://en.cppreference.com/w/cpp/ranges#Range_concepts)
