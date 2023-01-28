<!-- markdownlint-disable-file MD042 MD037 MD033 -->
# 4.2 std::span

※ gcc compiler 기준으로 정리

## Contents

- [개요](#개요)
- [정적 길이 대 동적 길이](#정적-길이-대-동적-길이)
- [포인터와 크기로 std::span 생성](#포인터와-크기로-stdspan-생성)
- [참조하는 객체의 수정](#참조하는-객체의-수정)
- [std::span의 요소 접근](#stdspan-요소-접근)
- [수정 가능한 요소들의 상수 구간](#수정-가능-요소들의-상수-구간)
- [요약](#요약)
- [Refs](#refs)

## 개요

- std::span은 연속 객체 순차열(contiguous sequence of objects : 객체들이 메모리 안에 연달에 저장되는 순차열)을 참조하는 객체
  - 연속 객체 : C 배열, 포인터와 크기, std::array, std::vector, std::string 등이 해당됨
- 뷰처럼 std::span도 해당 객체들을 소유하지 않음

- array를 함수로 안전하게 전달할 수 있는 방법 중 하나
- 일반적인 implementation은 첫번째 요소에 대한 포인터와 크기로 구성됨
- 함수로 전달되면 배열이 포인터로 바뀌고, 이러한 변화는 c/c++에서 자주 발생되는 오류 중 하나

- std::span의 선언

```cpp
template <typename T, std::size_t Extent = std::dynamic_extent>
class span;
```

## 정적 길이 대 동적 길이

- 정적 길이 (static extent)
  - 크기가 컴파일 시점에 결정되고, 해당 형식 자체에 크기 정보가 포함됨
  - 크기가 알려져있고, 형식에 포함되어 있으므로, 연속된 객체들의 순차열의 첫 요소를 가리키는 포인터 만으로 std::span을 구현 가능
- 동적 길이 (dynamic extent)
  - 크기가 실행 시점에 결정됨
  - 동적 길이를 가진 std::span을 구현하려면 연속 객체 순차열의 첫 요소를 가리키는 포인터와 그 순차열의 크기(요수 개수)가 필요함
  - 동적 길이를 가진 std::span의 형식은 std::span<T>
  - 크기 정보가 형식에 포함되어 있지 않음

## 연속 객체 순차열 크기 자동 연역

- std::span<T>는 연속 객체 순차열의 크기를 자동으로 연역함
  - int 값들을 참조하여 주어진 연속 객체 순차열의 크기를 자동으로 연역함

```cpp
void printMe(std::span<int> container) {
  std::cout << "container.size() : " << container.size() << "\n";
  for (auto e : container)
    std::cout << e << ' ';
  std::cout << "\n";
}

int main() {
  int arr[]{1,2,3,4};
  printMe(arr);

  std::vector v{1,2,3,4,5};
  printMe(v);

  std::array arr2{1,2,3,4,5,6};
  printMe(arr2);
}
```

## 포인터와 크기로 std::span 생성

- 포인터와 포인터의 크기로 std::span 객체를 생성 가능

```cpp
int main() {
  std::vector v{1,2,3,4,5};

  std::span mySpan1{v};
  std::span mySpan2{v.data(), v.size()};

  bool spansEqual = std::equal(mySpan1.begin(), mySpan1.end(),
                                mySpan2.begin(), mySpan2.end());
  std::cout << spansEqual << "\n";  // true  
}
```

- std::span은 std::string_view도 아니고 뷰도 아님
  - std::span은 구간 라이브러리의 뷰나 std::strnig_view와 다름
  - 구간 라이브러리의 뷰
    - 구간에 적용해서 구간의 요소들에 어떤 연산을 수행하는 수단
    - 뷰는 데이터를 소유하지 않음
    - 복사/이동/배정 연산의 시간 복잡도가 상수
  - std::span과 std::string_view는 데이터를 소유하지 않음
    - std::string_view는 문자열에 특화된 뷰
    - 차이 : std::span은 참조하는 객체를 수정 가능

## 참조하는 객체의 수정

- std::span의 요소들을 수정하면, 그것이 참조하는 객체들이 실제로 수정됨
- std::span 전체를 수정할 수 있고, 일부분만 수정 가능
- subspan : std::span의 일부분

```cpp
int main() {
  std::vector v{1,2,3,4,5,6,7,8,9,10};
  
  std::span span1(v);
  std::span span2(span1.subspan(1, span1.size() - 2));

  std::transform(span2.begin(), span2.end(), span2.begin(),
                [](int i) { return i*i; });
}

// v : 1 ~ 10, size : 10
// v/span1 : span2에 의해 시작값 1과 마지막값 10을 제외한 나머지 값들이 제곱으로 변함
// 1, 4, 9, 16, 25, ~ 81, 10 / size : 10
```

## std::span 요소 접근

- std::span의 요소들에 접근하는 멤버 함수들과 요소들에 관한 정보를 돌려주는 멤버 함수
- std::span의 인터페이스 (sp는 임의의 std::span 객체를 의미)

|함수|설명|
|:---:|:---:|
|sp.front | 첫 요소에 접근한다|
|sp.back() | 마지막 요소에 접근한다|
|sp[i] | i번째 요소에 접근한다|
|sp.data() | 순차열의 첫 요소를 가리키는 포인터를 돌려준다|
|sp.size() | 순차열 요소들의 개수를 돌려준다|
|sp.size_bytes() | 순차열의 바이트 단위 크기를 돌려준다|
|sp.empty() | 순차열이 비었으면 true를 돌려준다|
|sp.first<count>()  sp.first(count) | 순차열의 첫 요소부터 count개의 요소들로 이루어진 subspan을 돌려준다|
|sp.last<count>() sp.last(count) | 순차열의 마지막 요소부터 count개의 요소들로 이루어진 subspan을 돌려준다|
|sp.subspan<first, count>() sp.subspan(first, count) | first번째 요소부터 count개의 요소들로 이루어진 subspan을 돌려준다|

## 수정 가능 요소들의 상수 구간

- std::vector<T> 형태로 선언된 std::vector객체는 std::string처럼 수정 가능(modifiable) 요소들의 수정 가능 구간이라는 모형을 따름
- std::vector에 const를 붙여서 const std::vector<T> 형태로 선언하면 상수(constant) 요소들의 상수 구간이 됨
- std::vector만으로는 수정 가능 요소들의 상수 구간을 만들 수 없으므로 std::span을 활용한다

| |수정 가능 요소|상수 요소|
|:---:|:---:|:---:|
|수정 가능 구간|std::vector<T>| |
|상수 구간|std::span<T>| const std::vector<T>  std::span<const T>|

## 요약

- std::span은 연속 객체 순차열을 참조하는 객체
- 종종 view라고 불리는 std::span은 객체를 소유하지 않고, 메모리를 할당하지 않는다
- 연속 객체 순차열에는 보통 C 배열, 포인터와 크기(객체 개수)의 조합, std::array, std::vector, std::string 이 포함
- C 배열과 달리 std::span<T>는 연속 객체 순차열의 크기를 자동으로 연역
- std::span의 요소를 수정하면 해당 객체(std::span이 참조하는)가 실제로 수정됨

## Refs

- [C++20 std::span](https://www.modernescpp.com/index.php/component/jaggyblog/c-20-std-span)
- [cpprefer-std::span](https://en.cppreference.com/w/cpp/container/span)
