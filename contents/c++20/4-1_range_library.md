<!-- markdownlint-disable-file MD042 MD037 MD033 -->
# 4.1 구간 라이브러리

※ gcc compiler 기준으로 정리

## Contents

- [구간 라이브러리](#구간-라이브러리)
- [range 콘셉트와 view 콘셉트](#range-콘셉트와-view-콘셉트)
- [컨테이너에 직접 작용하는 알고리즘들](#컨테이너에-직접-작용하는-알고리즘들)
- [함수 합성](#함수-합성)
- [지연 평가](#지연-평가)
- [뷰의 정의](#뷰의-정의)
- [파이썬 향 첨가](#파이썬-향-첨가)
- [요약](#요약)
- [Refs](#refs)

## 구간 라이브러리

- 구간 라이브러리 알고리즘들은 지연 평가(lazy evalution)을 지원
- 컨테이너에 직접 작용하고, 합성하기 쉬움
- 구간 라이브러리는 함수형 프로그래밍의 개념들을 반영하여 사용하기 편하고 강력함

```cpp
// rangesFilterTransform.cpp

#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> numbers = {1, 2, 3, 4, 5, 6};
  
    auto results = numbers | std::views::filter([](int n){ return n % 2 == 0; })  // 2 4 6
                           | std::views::transform([](int n){ return n * 2; });  // 2*2, 4*2, 6*2
                           
    for (auto v: results) std::cout << v << " ";     // 4 8 12
}
```

- auto results 우변의 표현식은 왼쪽에서 오른쪽으로 읽어야 함
- 파이프 기호 (|)는 함수의 합성(function composition)을 뜻함
- std::views::filter는 주어진 numbers의 값 중 짝수만 통과시킴
- std::views::transform 은 그 값들을 2배로 만듬
- numbers 벡터는 구간에 해당
- std::views::filter와 std::views::transform은 뷰에 해당

## range 콘셉트와 view 콘셉트

- range 콘셉트 (std::ranges::range)
  - 구간의 요구조건들을 정의
  - 구간(range)은 반복할 수 있는 요소들의 묶음
  - 구간은 구간의 시작을 가리키는 반복자와 끝을 넘어선 곳을 가리키는 반복자를 제공
  - STL의 컨테이너들은 구간

- view 콘셉트 (std::ranges::view)
  - 주어진 구간이 뷰일 수 있는 요구조건들을 정의
  - 뷰는 구간에 어떠한 연산을 수행하기 위한 수단
  - 뷰는 스스로 데이터를 소유하지 않음
  - 뷰는 복사/이동/배정 연산의 시간 복잡도가 상수

- C++20 에서의 view
  - std::views는 std::ranges::views의 alias
  - std::views는 주어진 구간에 대한 뷰를 대표하는 구간 적응자(range adaptor) 객체
    - 컨테이너에 특정 연산을 적용해서 만든 뷰 객체를 돌려주는 함수 객체를 의미
  - 일반적으로 std::views::*에는 그에 대응되는 std::ranges::*_view가 있음

|뷰|설명|
|:---:|:---:|
|std::views::all_t </br>std::views::all | 주어진 구간의 모든 요소를 포함한 뷰를 돌려준다|
|std::ranges::ref_view| 주어진 구간의 모든 요소에 대한 참조로 이루어진 뷰를 돌려준다|
|std::ranges::filter_view </br>std::views::filter| 주어진 술어를 충족하는 모든 요소로 이루어진 뷰를 돌려준다|
|std::ranges::transform_view</br>std::views::transform|모든 요소를 각각 변환해서 만든 뷰를 돌려준다|
|std::take_view</br>std::views::take|주어진 뷰에서 처음 n개의 요소를 취해서 만든 뷰를 돌려준다|
|std::take_while_view</br>std::views::take_while|주어진 뷰의 요소 중 주어진 술어가 true를 돌려주는 요소들로 이루어진 뷰를 돌려준다|
|std::drop_view</br>std::views::drop|주어진 뷰에서 처음 n개의 요소를 제외한 요소들로 이루어진 뷰를 돌려준다|
|std::drop_while_view</br>std::views::drop_while|주어진 뷰에서 주어진 술어가 처음으로 flase를 돌려주기까지의 요소들을 제외한 요소들로 이루어진 뷰를 돌려준다|
|std::join_view</br>std::views::join|여러 구간을 결합한 뷰를 돌려준다|
|std::split_view</br>std::views::split|주어진 뷰를 주어진 분리자를 이용하여 여러 뷰로 분할한다|

## 컨테이너에 직접 작용하는 알고리즘들

- 기존에는 STL 알고리즘 사용시, 예) 컨테이너 전체를 정렬하는 경우, 시작 반복자(begin iterator)와 끝 반복자 (end iterator)를 모두 요구함
- c++20 에서는 구간 라이브러리를 이용하여 컨테이너 자체를 인수로 넘김
- std::sort와 std::ranges::sort외에도 기존 알고리즘 라이브러리의 알고리즘(<algorithm> 헤더에 정의된)에는 그에 대응되는 구간 라이브러리 알고리즘이 존재함

```cpp
#include <numeric>
#include <iostream>
#include <vector>

int main() {
    
    std::vector<int> myVec{1, 2, 3, 4, 5, 6, 7, 8, 9};
    auto res = std::sort(myVec.begin(), myVec.end()); 
    // std::ranges::sort(myVec);  // 구간 라이브러리 사용 시, 
    for (auto v: myVec) std::cout << v << " ";  // -4, -3, 0, 5, 7
}
```

### 사영

- std::ranges::sort에는 두 개의 중복적재 버전이 있음
- Comp : 정렬 가능한 구간 R과 비교를 위한 술어
  - std::ranges::less (미만)
- Proj : 사영 객체
  - std::identity (항등)
- 사영은 하나의 집합을 그것의 한 부분집합에 대응시키는 mapping

```cpp
template<std::random_access_iterator I, std::sentinel_for<I> S,
        class Comp = ranges::less, class Proj = std::identity>
requires std::sortable<I, Comp, Proj>
constexpr I sort( I first, S last, Comp comp = {}, Proj proj = {} );

template<ranges::random_access_range R, class Comp = ranges::less,
        class Proj = std::identity>
requires std:;sortable<ranges::iterator_t<R>, Comp, Proj>
constexpr ranges::borrowed_iterator_t<R> sort( R&& r, Comp comp = {}, Proj proj = {} );
```

### 연관 컨테이너 (associative container)

- 키들과 값들에 직접 접근하는 뷰들을 제공
- std::unordered_map 예제
  - 키들에 대한 뷰와 값들에 대한 뷰를 생성 - 1)
  - 생성된 뷰를 각각 출력 - 2)
  - 뷰들을 따로 만들지 않고 즉석에서 생성하여 출력하는 것도 가능 - 3)

```cpp
#include <iostream>
#include <ranges>
#include <string>
#include <unordered_map>

int main() {
    
    std::unordered_map<std::string, int> freqWord{ {"witch", 25}, {"wizard", 33}, {"tale", 45},
                                                   {"dog", 4}, {"cat", 34}, {"fish", 23} };
    
    std::cout << "Keys" << std::endl;
    auto names = std::views::keys(freqWord);  // 1)
    for (const auto& name : names){ std::cout << name << " "; };  // 2)
    std::cout << std::endl;
    for (const auto& name : std::views::keys(freqWord)){ std::cout << name << " "; };  // 3)
    
    std::cout << "\n\n";
    
    std::cout << "Values: " << std::endl;
    auto values = std::views::values(freqWord);  // 1)
    for (const auto& value : values){ std::cout << value << " "; };  // 2)
    std::cout << std::endl;
    for (const auto& value : std::views::values(freqWord)){ std::cout << value << " "; }    // 3)
}
```

## 함수 합성

- 파이프 기호(|)를 이용하여 함수 합성
- 파이프 기호는 함수 합성을 위한 편의 구문에 해당
  - R|C는 C(R) 과 동일한 의미를 나타냄

```cpp
// rangesComposition.cpp

#include <iostream>
#include <ranges>
#include <string>
#include <map>

int main() {
    
    std::map<std::string, int> freqWord{ {"witch", 25}, {"wizard", 33}, {"tale", 45},
                                                   {"dog", 4}, {"cat", 34}, {"fish", 23} };
                                                
    std::cout << "All words: ";                     // 모든 키를 출력
    for (const auto& name : std::views::keys(freqWord)) { std::cout << name << " "; };                                               
                                                   
    std::cout << std::endl;
    
    std::cout << "All words reverse: ";             // 키들을 역순으로 출력
    for (const auto& name : std::views::keys(freqWord) | std::views::reverse) { std::cout << name << " "; };  
     
    std::cout << std::endl;
    
    std::cout << "The first 4 words: ";             // 처음 4개의 키만 출력
    for (const auto& name : std::views::keys(freqWord) | std::views::take(4)) { std::cout << name << " "; }; 
    
    std::cout << std::endl;
                                
    std::cout << "All words starting with w: ";     // 영문자 'w'로 시작하는 키들만 출력
    auto firstw = [](const std::string& name){ return name[0] == 'w'; };
    for (const auto& name : std::views::keys(freqWord) | std::views::filter(firstw)) { std::cout << name << " "; };
    
    std::cout << std::endl;
                                                     // (5)
    auto lengthOf = [](const std::string& name){ return name.size(); };
    auto res = ranges::accumulate(std::views::keys(freqWord) | std::views::transform(lengthOf), 0);
    std::cout << "Sum of words: " << res << std::endl;  
}
```

## 지연 평가

- Lazy Evaluation
- std::views::iota(0)은 값을 요청할 때만 실제로 평가됨

```cpp
// rangesIota.cpp

#include <iostream>
#include <numeric>
#include <ranges>
#include <vector>

int main() {
    
    std::cout << std::boolalpha;
    
    std::vector<int> vec;
    std::vector<int> vec2;
    
    for (int i: std::views::iota(0, 10)) vec.push_back(i);  // 0 ~ 9까지 총 10개의 정수를 생성
         
    for (int i: std::views::iota(0) | std::views::take(10)) vec2.push_back(i);  // 0에서 시작해서 1씩 증가하는 무한 데이터 스트림을 생성
    
    std::cout << "vec == vec2: " << (vec == vec2) << '\n'; // true
    
    for (int i: vec) std::cout << i << " ";                // 0 1 2 3 4 5 6 7 8 9                           
}
```

## 뷰의 정의

- std::ranges::view_interface
  - view가 되는 클래스는 공통 부분의 구현을 단순화 하기 위해 view_interface 클래스를 상속받음
  - view concepts를 충족하려면 기본 생성자가 적어도 1개 이상 필요하고, 멤버 함수 begin()과 end() 가 필요
  - view_interface는 CRTP(Curriously Recurring Template Pattern)에 의해 파생 클래스 타입을 전달 받을 수 있음
  - `https://en.cppreference.com/w/cpp/ranges/view_interface`

```cpp
class MyView : public std::ranges::view_interface<MyView> {
public:
  auto begin() const { /*...*/ }
  auto end() const { /*...*/ }
};
```

- ContainerView
  - 임의의 컨테이너에 대한 뷰를 생성
  - 클래스 템플릿 인수 연역 지침(class template argument deduction guide)
  - 더 내용 추가 필요..
    - 컨테이너에 대한 함수를 만들어서 begin / end 에 대해 생성자로 호출해주고, 반복문에서 이 ContainerView(컨테이너 객체)를 호출하면 begin/end에 대해 호출해서 출력해주는? 예제인 것 같은데...

## 파이썬 향 첨가

- 파이썬 표준 라이브러리에는 filter와 map 함수가 존재
  - filter : 반복 가능 객체의 모든 요소에 술어를 적용하여, 그 술어가 true인 경우의 요소만 돌려줌
  - map : 반복 가능 객체의 모든 요소에 함수를 적용하여, 그 함수의 반환값들로 이루어진 새 반복 객체를 돌려줌
- 목록 형성 : 반복 가능 객체에 필터링 단계와 매핑 단계를 적용해서 새 반복 객체를 생성함

## 요약

- 구간 라이브러리는 기존 STL 알고리즘들에 대응되는 알고리즘들을 제공한다
- 구간 라이브러리의 알고리즘들은,
  - 지연 평가되며, 따라서 무한 데이터 스트림에 사용할 수 있다.
  - 컨테이너에 적용할 수 있다. 시작 반복자와 끝 반복자로 구간을 정의할 필요가 없다.
  - 파이프 기호(|)를 사용하여 합성할 수 있다.

## Refs

- [C++20: The Ranges Library](https://www.modernescpp.com/index.php/c-20-the-ranges-library)
- [functional patterns with the range library](https://www.modernescpp.com/index.php/c-20-functional-patterns-with-the-ranges-library)
- [Pythonic with the ranges library](https://www.modernescpp.com/index.php/c-20-pythonic-with-the-ranges-library)
- 사영 또는 투영 : 어떤 집합을 부분집합으로 특정한 조건을 만족시키면서 옮기는 작용
