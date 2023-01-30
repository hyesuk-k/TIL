<!-- markdownlint-disable-file MD042 MD037 MD033 -->
# 4.3 container 개선사항

※ gcc compiler 기준으로 정리

## Contents

- [개요](#개요)
- [constexpr 컨테이너와 알고리즘](#constexpr-컨테이너와-알고리즘)
- [std::array](#stdarray)
- [일관된 컨테이너 삭제](#일관된-컨테이너-삭제)
- [연관된 컨테이너를 위한 contains](#연관된-컨테이너를-위한-contains)
- [문자열 접두사/접미사 점검](#문자열-접두사접미사-점검)
- [요약](#요약)

## 개요

- std::vector와 std::string의 인터페이스에 constexpr이 적용
  - 변수, 함수, 생성자 함수에 대해 컴파일 타임에 평가되도록 처리 가능

## constexpr 컨테이너와 알고리즘

- constexpr 컨테이너는 생성자와 멤버 함수들에 constexpr이 적용되어 컴파일 시점에 사용 가능한 컨테이너

```cpp
constexpr int maxElement() {
  // 컴파일 시점에 생성됨
  std::vector v = {1,3,4,2};
  // 컴파일 시점에 실행됨
  std::sort(v.begin(), v.end());
  return v.back();
}

int main() {
  constexpr int maxValue = maxElement();
  std::cout << "max value : " << maxValue << "\n";  

  // 컴파일 시점에 람다 안에서 생성하여 정렬 수행
  constexpr int maxValue2 = [] {
    std::vector v = {1,2,4,3};
    std::sort(v.begin(), v.end());
    return v.back();
  }();
  std::cout << "max value2 : " << maxValue2 << "\n";  
}
```

## std::array

- 배열을 생성하는 두 가지 편의 수단이 추가됨

- std::to_array
  - 기존 1차원 배열로부터 std::array를 생성
  - 1차원 배열로부터 std::array 객체를 생성함
  - 생성된 std::array 요소들은 기존 1차원 배열로부터 복사 초기화 됨
  - 기존 1차원 배열 : std::initializer_list, std::pair

```cpp
int main() {
  auto arr1 = std::to_array("A simple test");
  for (auto a : arr1) std::cout << a;
  std::cout << "\n";

  auto arr2 = std::to_array({1,2,3,4,5});
  for (auto a : arr2) std::cout << a;
  std::cout << "\n";

  auto arr3 = std::to_array<double>({0,1,3});
  for (auto a : arr3) std::cout << a;
  std::cout << "\n";
}
```

- std::make_shared
  - 배열을 가리키는 std::shared_ptr을 생성하는 중복 적재 버전
  - std::shared_ptr<double[]> shar = std::make_shared<double[]>(1024) : 기본 값으로 초기화된 double 1,024개를 담은 배열을 가리키는 shared_ptr을 생성
  - std::shared_ptr<double[]> shar = std::make_shared<double[]>(1024, 1.0): 1.0으로 초기화된 double 1,024개를 담은 배열을 가리키는 shared_ptr을 생성

## 일관된 컨테이너 삭제

- erase-remove 관용구
  - 기존 C++에서 컨테이너에서 요소를 제거하는 방법은 std::remove_if 와 std::erase 함수를 둘 다 사용해야 함
  - std::remove_if 로 주어진 조건을 충족하는 요소를 컨테이너의 뒤로 옮기고
  - 논리적 끝에서부터 나머지 요소들을 std::erase로 삭제해야 함
  - erase와 remove를 하나의 표현식에서 사용하는 경우가 많아 erase-remove 관용구로 표현함

- C++20의 std::erase와 std::erase_if
  - 두 번의 함수 호출과 반복자를 지정해야 하는 불편함을 개선하고자 추가됨

```cpp
int main() {
  std::vector<int> v = {-2,-1,0,1,2};

  // 0 삭제
  std::erase(v,0);

  // 0미만 삭제
  std::erase_if (v, [](int Num) {return Num < 0; });

  for (auto n : v) std::cout << n << "\n";
}
```

## 연관된 컨테이너를 위한 contains

- 주어진 요소가 연관 컨테이너에 들어있는지의 여부를 돌려주는 멤버함수 contains가 추가됨
- 기존의 find와 count와의 차이
  - find : 호출이 장황함, 요소의 위치를 검색
  - count : 컨테이너 포함 유무만 체크하는 용도로 사용하기엔 처음부터 마지막 요소까지 전부 훑으며 갯수를 셈

## 문자열 접두사/접미사 점검

- std::string에 starts_with와 end_with 멤버함수가 추가됨
- 특정 부분 문자열로 시작하는지 (prefix) 또는 끝나는지 (suffix) 를 점검

```cpp
int main() {
  std::string str = "test strings !"

  str.starts_with("test"); // true
  str.end_with("!"); // true
}
```

## 요약

- std::vector와 std::string은 컴파일 시점에서 사용할 수 있는 constexpr 컨테이너에 해당됨
  - 컴파일 시점에서 std::vector와 std::string을 생성하고 STL의 constexpr 알고리즘들을 적용 가능
- C++20에서는 배열을 새엇ㅇ하는 두 가지 편의 수단이 추가됨
  - std::to_array : 기존 배열로부터 std::array를 생성
  - std::make_shared : 배열을 가리키는 std::shared_ptr을 생성
- 임의의 STL 컨테이너에서 생성하는 특정 요소들 또는 주어진 술어를 충족하는 요소들을 좀 더 간편하게 제거할 수 있는 std::erase와 std::erase_if 가 추가됨
- 연관 컨테이너들에 추가된 멤버 함수 contains로 연관 컨테이너에 특정 키가 포함되어 있는지 확인 가능해짐
- std::string 함수 추가 (start_with와 end_with) : 문자열이 특정한 접두사로 시작하거나 접미사로 끝나는지 점검하는 함수

### 기타

- const
  - read only memory 에 할당되며, 주소를 직접 접근하여 수정하면 변형 가능 (위험..)
  - type을 명시적으로 기록
  - runtime / compile time 무관
  - #define과 동일한 의미로 사용된다면 컴파일러가 최적화 시켜 메모리에 적재되지 않음 (debuging compile 제외)

- #define
  - 메모리에 올라가지 않음 (임베디드 시스템이나 메모리가 부족한 환경)
  - pre-compile에 치환됨
  - type을 명시적으로 기록하지 않음
  - C에서 const가 없을 때 사용 가능
  - C++에서는 enum을 활용,

- constexpr
  - 변수 함수, 클래스를 compile time에 정수로 사용 가능
