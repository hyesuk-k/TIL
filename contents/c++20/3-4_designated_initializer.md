# 3.4 지명 초기화

※ gcc compiler 기준으로 정리

## Contents

- [지명초기화](#designated-initializers)
- [Refs](#refs)

## Designated initializers

* 구조체나 클래스의 멤버 변수를 초기화할 때, 지정자 (Designator)를 사용하여 어떤 변수를 초기화 할 지 명시
* 특정 변수를 지정하여 초기화 가능

```cpp
#include <iostream>
struct A { 
    int x, y, z;
};

int main() {
    A v{1,2,3}; // c++11
    A v2{.x = 10, .z = 20};  // y = 0, -Wmissing-field-initializers
    A v3{.y = 100, .z = 200};  // x = 0, -Wmissing-field-initializers
    // A v4{1000, 2000, .z= 3000};  // ERROR, either all initializer clauses should be designated or none of them should be
}
```


### 사용 방법 및 주의사항

* 특정 멤버 변수를 생략 가능 (생략된 변수는 0으로 초기화 됨)
* 지정자의 명시 순서는 구조체 또는 클래스 내에 선언된 변수의 순서와 동일해야 함
* 지정자 사용 시, 모든 인자를 지정자로 작성 (should be, missing-field-initializers warning 발생하므로 컴파일 옵션에 따라 다를듯)
* 지정자를 중첩하여 변수 초기화 불가능


## Refs

* [aggregate_initialization](https://en.cppreference.com/w/cpp/language/aggregate_initialization)
* [Designated_initializers](https://en.cppreference.com/w/cpp/language/aggregate_initialization#Designated_initializers)