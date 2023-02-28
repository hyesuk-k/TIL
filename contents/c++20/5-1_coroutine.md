<!-- markdownlint-disable-file MD042 MD037 MD033 -->
# 5.1 코루틴

※ gcc compiler 기준으로 정리

## Contents

- [개요](#개요)
- [코루틴의 특징](#코루틴의-특징)
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
- 코루틴의 구성요서 (component)
  - Operators & Awaitable type
  - Promise
  - Coroutine handle

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

- promise object는 코루틴 내부에서 조작됨
- 약속 객체를 이용하여 자신의 결과 또는 예외를 전달함
- 약속 객체는 반드시 아래 인터페이스를 지원해야 함

|멤버 함수|설명|
|:---:|:---:|
|기본 생성자|약속 객체는 반드시 기본 생성 가능이어야 함|
|initial_suspend()|코루틴이 일시 정지 상태로 시작하는지의 여부를 돌려줌|
|final_suspend() noexcept | 코루틴이 일시 정지 상태로 종료되는지 여부를 돌려줌|
|unhandled_exception() | 예외가 발생하면 호출됨|
|get_return_object|코루틴 객체 (재개 가능 객체)를 돌려줌|
|return_value(val)|co_return val에 의해 호출됨|
|return_void()|co_return에 의해 호출됨|
|yield_value(val)|co_yield val에 의해 호출됨|

- 멤버 함수들은 코루틴 실행 중 컴파일러가 자동으로 호출함
- 약속 객체에는 return_value, return_void, yield_value 중 적어도 하나가 있어야 함
- 대기 가능 객체는 코루틴이 일시 정지 되었는지 아닌지 판단하는 멤버 함수를 제공함

### 코루틴 핸들

- 코루틴 프레임의 실행 재개나 파괴를 외부에서 제어하는 데 쓰이는 비소유(non-owing) 핸들
- 코루틴 핸들은 재개 가능 함수의 한 부품
- 재개 가능 객체에는 내부 형식 promise_type이 있어야 함
  - 재개 가능 객체에는 반드시 promise_type이라는 내부 형식이 있어야 함

- Promise interface
  - promise_type을 통해 코루틴 실행 시, suspend 및 resume 지점에서 호출되는 interface를 정의하고 코루틴의 동작을 제어
  - 비동기 함수 호출 시, 어떤 상태를 넘기기로 사전에 약속한 객체

### 코루틴 프레임

- 코루틴의 상태를 담는 내부 객체, 흔히 내부 heap 영역에 할당됨
- 코루틴 프레임의 구성
  - 약속 객체와 코루틴 인수들의 복사본
  - 일시 정지 지점(suspension point)를 나타내는 객체
  - 일시 정지 지점에 도달 하기 전에 그 수명이 끝나는 지역 변수들
  - 수명이 현재 일시 정지 지점 이후까지 연장되는 지역 변수들
- 저장 공간 절약을 위해 컴파일러가 코루틴의 할당(allocation)을 최적화 하기 위한 2가지 조건
  - 코루틴 객체의 수명이 코루틴 호출자의 수명 안에 포함되어야 함
  - 코루틴 호출자가 코루틴 프레임의 크기를 알아야 함
- 코루틴 프레임워크의 2가지 핵심 추상
  - 대기 가능 객체
  - 대기자 객체

### 대기 가능 객체와 대기자 객체

- 약속 객체의 세 멤버 함수 yield_value, inital_suspend, final_suspend는 대기 가능 객체를 돌려줌

#### 대기 가능 객체

- 대기 가능 객체 : 기다릴 수 있는 개체
- 코루틴이 일시 정지(suspend) 중인지 아닌지를 판정
- yield_value와 initial_suspend, final_suspend 멤버 함수에 대해 co_await 연산자를 적용함
  - 멤버 함수 호출 전에 알아서 co_await 연산자를 적용해준다는 뜻인가..?

|함수|컴파일러가 생성한 호출|
|:---:|:---:|
|prom_obj.yield_value(value)|co_await prom_obj.yield_value(value)|
|prom_obj.initial_suspend()|co_await prom_obj.initial_suspend()|
|prom_obj.final_suspend()|co_await prom_obj.final_suspend()|

- 단항 연산자인 co_await은 대기 가능 객체를 인수로 받음
- 대기 가능 객체의 형식은 Awaitable 콘셉트를 충족해야 한다

#### Awaitable 객체

- Awaitable 콘셉트는 3가지 멤버 함수를 요구함

|멤버 함수|설명|
|:---:|:---:|
|await_ready|결과가 준비되었는지의 여부를 돌려줌, 이 멤버 함수가 false를 돌려주면 await_suspend가 호출됨|
|await_suspend|코루틴의 실행 재개(resume) 또는 파괴(소멸)를 결정|
|await_resume|co_await 표현식의 결과를 제공|

- std::suspend_always
  - 항상 코루틴을 일시 정지함
  - await_ready 멤버 함수의 결과가 항상 false

- std::suspend_never
  - 코루틴을 일시 정지 하지 않음
  - await_ready는 항상 true

- suspend_always와 suspend_never는 initial_suspend나 final_suspend 같은 약속 객체 멤버 함수의 기본 구축 요소
  - 두 함수는 코루틴이 실행되면 자동으로 호출됨
  - 코루틴의 시작에 initial_suspend, 코루틴의 끝에서 final_suspend가 호출됨

- initial_suspend
  - suspend_always 또는 suspend_never에 따라 다르게 코루틴을 시작함

```cpp
// 코루틴을 일시 정지 상태로 시작함
std::suspend_always initial_suspend() {
  return {};
}

// 코루틴을 즉시 실행
std::suspend_never initial_suspend() {
  return {};
}
```

- final_suspend
  - suspend_always : 코루틴은 자신의 끝에서 일시 정지됨
  - suspend_never : 코루틴은 끝에서 일시정지 되지 않음

```cpp
// 코루틴의 끝에서 일시 정지 됨
std::suspend_always final_suspend() noexcept {
  return {};
}

// 코루틴의 끝에서 일시정지 되지 않음
std::suspend_never final_suspend() noexcept {
  return {};
}
```

#### 대기자 객체

- 대기자 객체(Awaiter Ojbect) : 코루틴의 대기 가능 객체가 종료 또는 일시 정지되길 기다리는 객체
- 대기자 객체가 되는 2가지 방법
  - co_await 연산자를 정의
  - 대기 가능 객체를 대기자 객체로 변환
- 컴파일1러가 대기자 객체를 얻는 순서
  - 약속 객체의 co_await 연산자가 있으면 그것을 적용하여 대기자 객체를 획득함
    - awaiter = awaitable.operator co_await();
  - 독립적인 co_wait 연산자가 있다면 그것으로 대기자 객체를 획득함
    - awaiter = operator co_await();
  - 두 co_wait 연산자 모두 없으면 대기 가능 객체로부터 대기자 객체를 얻음
    - awaiter = awaitable;

## 제어 흐름

- 컴파일러는 개발자가 작성한 코루틴 함수를 적절하게 변환해서 두 개의 제어 흐름을 실행함
- 약속 제어 흐름 / 대기자 제어 흐름

### 약속 객체 제어 흐름

- 어떤 함수 내에서 co_yield나 co_await, co_return을 사용하면 그 함수는 코루틴이 됨
- 컴파일러는 변환한 코드를 약속 객체의 멤버를 이용하여 자동으로 실행
- 약속 객체를 이용한 제어 흐름 => 약속 제어 흐름
- 약속 제어 흐름의 주요 단계
  - 코루틴이 시작되면 컴파일러는 :
    - 필요하다면 코루틴 프레임을 할당
    - 모든 함수 매개변수를 코루틴 프레임에 복사
    - 약속 객체를 생성
    - 약속 객체.get_return_object()를 호출해서 코루틴 핸들을 생성하고 지역 변수에 저장, 호출 결과는 코루틴이 처음으로 일시 정지될 때 코루틴 호출자에게 반환됨
    - 약속 객체.initial_suspend()를 호출하고 그 결과에 대해 co_await을 적용
    - co_await 약속객체.initial_suspend()의 실행이 재개되면 코루틴의 본문을 실행함
  - 코루틴이 일시 정지 지점에 도달하면 컴파일러는 :
    - 이후 코루틴의 실행이 재개되면 앞에서 저장해 둔 반환 객체 (get_return_object())를 호출자에 돌려줌
  - 코루틴이 co_return에 도달하면 컴파일러는 :
    - co_return 또는 co_return <void 형식의 표현식> : 약속객체.return_void()를 호출
    - co_return <void가 아닌 형식> : 약속객체.return_value(형식)을 호출
    - 스택에 생성한 모든 변수를 파괴
    - final_suspend()를 호출하고 그 결과에 co_await을 적용함
  - 코루틴이 파괴되면(co_return이나, 기타 예외, 코루틴 핸들을 통한 코루틴 종료) 컴파일러는 :
    - 약속 객체의 소멸자를 호출
    - 함수 매개변수들의 소멸자를 호출
    - 코루틴 프레임이 사용하던 메모리를 해제
    - 제어 흐름을 다시 호출자에게 돌려줌
  - 코루틴 안에서 잡히지 않은 예외(uncaught exception)이 발생하면 컴파일러는 :
    - 그 예외를 해당 catch 문으로 잡아서 약속객체.unhandled_exception()을 호출
    - ifnal_suspend()를 호출하고 그 결과에 co_await을 적용함
  
- 제어 흐름이 코루틴 안의 co_await 표현식에 도달하거나 컴파일러가 암묵적으로 initial_suspend()/final_suspend()/yield_value를 호출하면 대기자 제어 흐름이 시작됨

### 대기자 제어 흐름

- 대기자 객체를 이용한 실행 흐름을 '대기자 제어 흐름'으로 부름
- co_await 표현식을 컴파일러가 대기자 객체 멤버 함수 await_ready, await_suspend, await_resume을 이용한 코드로 변환

- awaiteble.await_suspend()의 반환 형식에 따른 처리
|형식|처리|
|:---:|:---:|
|void|코루틴을 일시 정지 상태로 두고 호출자에게 제어 흐름을 돌려줌|
|bool|true : 계속 일시 정지 상태로 두고 호출자에게 제어 흐름을 돌려줌 / false : 코루틴의 실행을 재개, 제어 흐름은 호출자에게 돌아가지 않음|
|기타 코루틴 핸들|해당 코루틴의 실행을 재개하고 호출자에게 제어 흐름을 돌려줌|

- 코루틴 내에서의 예외
  - await_ready
    - 코루틴이 일시정지 되지 않음
    - await_suspend 호출이나 await_resume의 호출을 평가하지 않음
  - await_suspend
    - 예외를 잡고, 코루틴의 실행을 재개하고, 예외를 다시 던짐
    - await_resume은 호출하지 않음
  - await_resume
    - await_ready와 await_suspend를 평가하고 모든 값을 반환
    - await_resume은 결과를 돌려주지 않음

## co_return

- 코루틴은 return 대신 co_return을 사용
- 코루틴을 완전히 종료하기 위해 사용
- co_return 이후로는 resume할 수 없음
- co_return을 만나면 스택에 생성된 모든 변수들이 생성 반대 순서로 해제됨
- 약속객체.return_value(exprt) / 약속객체.return_void() 후에 co_await 약속객체.final_suspend()가 호출됨

```cpp
Type<int> func() {
  co_return 10;
}
```

## co_yield

- caller에게 값을 전달하기 위해 사용
- 값을 리턴하고 실행을 중지 및 제어 흐름을 넘기는 역할
- co_return과 다르게 코루틴 함수 실행을 종료하지는 않음

```cpp
generator<int> createNum() {
  while(true) {
    srand(time(NULL));
    int n = rand();    
    co_yield n;
  }
}

main() {
  for (auto i : N) {
    std::cout << createNum() << "\n";
  }
}
```

## co_await

- 코루틴의 실행을 일시 정지하거나 재개
- 실행 중인 함수를 멈추고 제어를 코루틴의 caller로 넘김
- 제어를 받아 다시 실행될 때 해당 문장 이후로 이어서 실행함
- co_await {awaitable} 로 대기자 객체를 사용하도록 되어 있음

```cpp
resumable func() {
  co_await std::experimental::suspend_always(); // OR std::suspend_never
}
```

## 코루틴을 이용한 스레드 동기화

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
- [cpp-coroutine알아보기](https://luncliff.github.io/coroutine/ppt/[Kor]ExploringTheCppCoroutine.pdf)
- [coroutine-예시](https://kukuta.tistory.com/222)
- [coroutine-예시-scs](https://www.scs.stanford.edu/~dm/blog/c++-coroutines.html)
