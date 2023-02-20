<!-- markdownlint-disable-file MD042 MD037 MD033 -->
# 5.1 코루틴

※ gcc compiler 기준으로 정리

## Contents

- [개요](#개요)
- [코루틴의 특징](#코루틴의-특징)
- [대기 가능 객체와 대기자 객체](#대기-가능-객체와-대기자-객체)
- [제어 흐름](#제어-흐름)
- [co_return](#co_return)
- [co_yield](#co_yield)
- [co_await](#co_await)
- [요약](#요약)
- [Refs](#refs)

## 개요

- C++20 에서 코루틴이 도입됨
- 자신의 상태를 유지하면서 실행을 일시 정지(suspend) 하거나 재개(resume) 할 수 있는 함수
- 프로그램의 가장 작은 실행단위(routine)를 쪼개어 각 routine들이 번갈아 실행되도록 구성 (쪼개는 지점은 개발자가 키워드를 이용하여 지정 가능)
- 함수의 제어 흐름은 한 번의 호출과 한 번의 반환(return)으로만 구성됨
- 코루틴은 suspend와 resume을 여러 번 반복할 수 있으며, suspend 상태의 코루틴을 caller가 파괴(소멸)할 수 있음
- 코루틴을 위해 co_await과 co_yield 키워드가 추가됨
  - co_await
    - 표현식 앞에 붙이는 단항 연산자
    - 표현식의 실행을 suspend 또는 resume 가능
    - 해당 표현식을 포함하는 함수는 func 끝에 도달 전에 return될 수 있음
    - blocking 대신 waiting을 구현 가능
  - co_yield
    - generator 함수(호출할 때 새로운 값을 돌려주는 함수)의 구현에 사용됨
- C++ 함수의 실행에 2가지 개념이 추가됨

## 코루틴의 특징

- 전형적인 용례
  - event-driven programming (이벤트 주도 프로그램)
    - 시뮬레이션, 게임, 서버, 사용자 인터페이스, 알고리즘 등
  - cooperative multitasking (협조적 다중 태스킹)
    - 각 task가 시작을 필요한 만큼 사용하되, sleeping이나 waiting에는 빠지지 말고 다른 작업에 시간을 양보하는 것
    - 반대로 pre-emptive multitasking(선점형 다중 태스킹) 개념이 있음, 각 작업이 사용할 cpu 시간을 스케쥴러가 결정함

- 바탕 개념들 (C++20에서는 비대칭/일급/스택 없는 코루틴을 지원)
  - 비대칭 코루틴 (asymmetric coroutine)
    - 제어 흐름(control flow)이 caller로 돌아감
    - 대칭 코루틴(symmetric coroutine)은 제어 흐름이 caller로 돌아가지 않음 / 다른 coroutine에 control을 위임 가능
  - 일급 코루틴 (first-class coroutine)
    - 함수의 인수나 반환값으로 사용할 수 있고, 변수에 저장할 수 있음
  - 스택 없는 코루틴 (stackless coroutine)
    - 최상위 코루틴 (top-level coroutine)의 실행을 suspend 또는 resume 가능
    - 호출된 코루틴이 control flow를 양보(yield)하면 control flow는 caller로 돌아옴
    - resume을 위해 코루틴은 자신의 상태를 스택과 분리된 곳에 저장해둠
    - stackless coroutine을 resumable function이라고 부르기도 함

- 코루틴의 설계 목표
  - 고도로 규모가 가변적(scalable)이어야 함 (동시에 수십억 개의 코루틴들을 돌릴 수 있을 정도로)
  - suspend와 resume 연산이 고도로 효율적이어야 함 (보통 함수의 추가 부담과 비교 가능한 정도로)
  - C++의 기존 요소들과 추가 부담 없이 매끄럽게 상호 작용 가능해야 함
  - 확장성 있는 개방형(open-ended) 매커니즘을 갖추어야 함 (라이브러리 사용자가 생성기, goroutine, 작업 등의 코루틴 라이브러리를 개발 가능하도록)
  - 예외(exception)를 사용할 수 없거나 비용 때문에 사용이 사실상 불가능한 환경에서도 사용할 수 있어야 함
  - 규모가변성 목표와 기존 요소와의 상호작용 목표로 C++의 코루틴은 stackless

- 코루틴을 생성하는 4가지 방법
  - 아래 4가지 중 하나라도 사용하는 함수는 코루틴이 됨
  - co_return
  - co_await
  - co_yield
  - 구간 기반 for 루프 안의 co_await 표현식

- 코루틴은 서로 구분되는 2가지 개념을 지칭
  - 코루틴 객체
  - 코루틴 팩토리
    - co_return, co_wait, co_yield를 이용해서 코루틴 객체를 돌려줌

- 제약
  - 코루틴에는 return 문이나 자리표 반환 형식을 사용할 수 없음
  - 코루틴이 될 수 없는 함수 : 가변 인수 함수, constexpr 함수, consteval 함수, 클래스의 생성자와 소멸자, main 함수

## 코루틴 구현 프레임워크

- 코루틴 구현을 위한 프레임워크는 20개 이상의 함수로 구성됨
- 하나의 코루틴에는 약속 객체, 코루틴 핸들, 코루틴 프레임이라는 세 가지 부품이 관여함
- 클라이언트는 코루틴 핸들을 얻고, 그것으로 약속 객체와 상호 작용을 하며, 코루틴 프레임에는 코루틴의 상태가 저장됨

### 약속 객체

- 추가

### 코루틴 핸들

- 추가

### 코루틴 프레임

- 추가

## 대기 가능 객체와 대기자 객체

### 대기 가능 객체

- 대기 가능 객체 : 기다릴 수 있는 개체
- 코루틴이 일시 정지(suspend) 중인지 아닌지를 판정

## 제어 흐름

- 컴파일러는 개발자가 작성한 코루틴 함수를 적절하게 변환해서 두 개의 제어 흐름을 실행함
- 약속 제어 흐름 / 대기자 제어 흐름

### 약속 객체 제어 흐름

- 추가

### 대기자 제어 흐름

- 추가

## co_return

- 코루틴은 return 대신 co_return을 사용
- 코루틴을 완전히 종료하기 위해 사용
- co_return 이후로는 resume할 수 없음

```cpp
Type<int> func() {
  co_return 10;
}
```

### 미래 객체

## co_yield

- caller에게 값을 전달하기 위해 사용

```cpp
generator<int> createNum() {
  while(true) {
    srand(time(NULL));
    int n = rand();    
    co_yield n;
  }
}
```

## co_await

- 코루틴 중단을 위해 사용

```cpp
resumable func() {
  co_await std::experimental::suspend_always(); // OR std::suspend_never
}
```

## 요약

- 코루틴은 자신의 상태를 유지하면서 실행을 일시 정지하거나 제개할 수 있는, 일반화된 함수
- C++20은 구체적인 코루틴을 제공하는 것이 아니라 코루틴을 구현하기 위한 하나 이상의 프레임워크를 제공
- C++20은 새로운 두 가지 개념과 co_await과 co_yield 키워드를 도입해서 C++ 함수의 실행을 확장함
- co_await 표현식 형태의 구문을 이용하여 해당 표현식의 실행을 일시 정지하고 재개할 수 있음
  - 코루틴의 실행을 suspend 후 caller에게 제어권을 넘기는데 사용
- co_yield를 이용하면 무한 데이터 스트림을 생성하는 함수를 구현 가능
  - 코루틴을 suspend 후 caller에게 돌아갈 때, 값을 넘기는 등의 활용 가능

## Refs

- [wiki-coroutine](https://en.wikipedia.org/wiki/Coroutine)
- [cppref-co_await](https://en.cppreference.com/w/cpp/language/coroutines#co_await)
- [cppref-co_yield](https://en.cppreference.com/w/cpp/language/coroutines#co_yield)