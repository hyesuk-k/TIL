<!-- markdownlint-disable-file MD042 MD037 -->
# 3.8 새 특성들

※ gcc compiler 기준으로 정리

## Contents

- [특성](#특성)
- [nodiscard 특성](#nodiscard-특성)
- [likely와 unlikely 특성](#likely와-unlikely-특성)
- [no_unique_address 특성](#no_unique_address-특성)
- [Refs](#refs)

## 특성

- 소스 코드에 대한 추가적인 제약을 표현하거나 최적화 가능성을 컴파일러에 제시하는 수단
- 형식, 변수, 함수, 식별자, 코드 블록에 적용할 수 있음
- 하나의 대상에 여러 개의 특성을 적용하는 경우
  - 각각을 따로 지정
  - 하나의 이중 대괄호 쌍 안에 각 특성 이름을 쉼표로 분리하여 나열

```cpp
[[attribute1]] [[attribute2]] [[attribute3]]
int func1();

[[attribute1, attribute2, attribute3]]
int func2();
```

### 기존 특성들

- [[noreturn]] (C++11) : 해당 함수가 아무 겂도 반환하지 않음
- [[carries_dependency]](C++11) : 해제-소비 순서(release-consume ordering)의 의존관계사슬(dependency chain)을 나타냄
- [[deprecated]](C++14) : 해당 식별자를 사용하지 말아야 함을 의미
- [[fallthrough]](C++17) : 해당 case의 떨어짐(fallthrough)이 의도적임을 나타냄
  - switch문의 case 블록이 break로 끝나지 않고, 다음 case 블록으로 넘어가는 것을 의미
- [[maybe-unused]](C++17) : 사용되지 않은 식별자에 대한 컴파일러 경고 메시지 억제
- [[nodiscard]](C++17) : 반환값 폐기를 검출

## nodiscard 특성

- C++17에서 추가된 nodiscard 특성의 경우, 폐기 경고에 관한 추가적인 정보를 제공하는 수단이 없었음
- C++20 에서는 소괄호 안에 문자열 리터럴을 추가해서 폐기 경고의 이유를 명시 가능

```cpp
#incude <utility>

struct MyType {
    [[nodiscard("Implicit destroying of temporary MyInt.")]] MyType(int, bool) {}
};

template <typename T, typename ... Args>
[[nodiscard("You have a memory leak.")]]
T* create(Args&& ... args) {
    return new T(std::forward<Args>(args)...);
}

enum class [[nodiscard("Don't ignore the error code.")]] ErrorCode {
    Okay,
    Warning,
    Critical,
    Fatal
};

ErrorCode errorProneFunction() { return ErrorCode::Fatal; }

int main() {
    int *val = create<int>(5);
    delete val;

    // 컴파일 오류 시, memory leak 메시지가 출력됨
    create<int>(5);
    
    // 컴파일 오류 시, enum에 대한 메시지 출력됨
    errorProneFunction();

    // 컴파일 오류 시, Implicit 오류 발생
    MyType(5, true);

    return 0;
}
```

## likely와 unlikely 특성

- 선택될 가능성이 큰 또는 작은 실행 경로에 대한 힌트를 컴파일러가 최적화 할 때 제공

```cpp
for (size_t i = 0 ; i < v.size() ; ++i) {
    if (v[i] < 0) [[likely]] sum -= sqrt(-v[i]);
    else sum += sqrt(v[i]);
}
```

- C에서의 likely / unlikely 와 동일

```c
#define likely(x)       __builtin_expect((x),1)
#define unlikely(x)     __builtin_expect((x),0)
```

## no_unique_address 특성

- 클래스의 데이터 멤버에 적용하는 특성
- 해당 데이터 멤버(멤버 변수)의 주소가 고유할 필요가 없음을 의미
  - 클래스의 다른 모든 비정적 데이터 멤버와 구별되는 주소를 가질 필요 없음을 의미
- 이 특성이 적용된 데이터 멤버의 형식이 빈 형식(empty type)이면, 컴파일러가 그 데이터 멤버가 전혀 메모리를 차지하지 않도록 최적화 함

```cpp
#include <iostream>

struct Empty {};
struct NoUniqueAddress {
    int d{};
    [[no_unique_address]] Empty e{};
};

struct UniqueAddress {
    int d{};
    Empty e{};
};

int main() {
    std::cout << std::boolalpha;

    std::cout << "sizeof(int) == sizeof(NoUniqueAddress): " 
                << (sizeof(int) == sizeof(NoUniqueAddress)) << "\n";

    std::cout << "sizeof(int) == sizeof(UniqueAddress): " 
                << (sizeof(int) == sizeof(UniqueAddress)) << "\n";

    NoUniqueAddress NoUnique;

    std::cout << "&NoUnique.d: " << &NoUnique.d << "\n";
    std::cout << "&NoUnique.e: " << &NoUnique.e << "\n";

    UniqueAddress Unique;

    std::cout << "&Unique.d: " << &Unique.d << "\n";
    std::cout << "&Unique.e: " << &Unique.e << "\n";
    return 0;
}
```

## Refs

- [New Attributes with C++20](https://www.modernescpp.com/index.php/new-attributes-with-c-20)