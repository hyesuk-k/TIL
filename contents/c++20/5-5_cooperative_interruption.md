<!-- markdownlint-disable-file MD042 MD037 MD033 -->
# 5.5 협조적 가로채기

※ gcc compiler 기준으로 정리

## Contents

- [개요](#개요)
- [stop_source](#stdstop_source)
- [stop_token](#stdstop_token)
- [stop_callback](#stdstop_callback)
- [일반적인_신호전송_메커니즘](#일반적인-신호-전송-메커니즘)
- [jthread의 가로채기 지원](#stdjthread의-가로채기-지원)
- [condition_variable_any](#condition_variable_any)
- [요약](#요약)
- [Refs](#refs)

## 개요

- 협조적 가로채기 (cooperative interruption)
- std::stop_source, std::stop_token, std::stop_callback에 기반함
- std::jthread와 std::condition_variable_any가 협조적 가로채기를 지원

- 스레드 강제 종료 (kill)이 위함한 이유
  - 스레드의 상태를 알지 못하는 상태에서 스레드를 죽이는 것은 위험함
  - 스레드가 작업을 절반만 마친 상태에서 강제로 종료하면 스레드의 작업이 어디까지 진행되었는지 알 수 없음
  - 프로그램의 상태를 알 수 없고 정의되지 않은 행동이므로 어떤 일도 발생 가능함
  - 스레드가 임계 영역 (critical section) 안에 있고, 뮤텍스를 잠근 상태에서 강제 종료하면 교착이 발생될 수 있음

- std::stop_source, std::stop_token, std::stop_callback : 스레드 실행 종료를 비동기적으로 스레드에 요청하거나 스레드가 종료 신호를 받았는지 여부를 조회 가능

## std::stop_source

```cpp
std::stop_source();
explicit std::stop_source(std::nostopstate_t) noexcept;
```

- 기본 생성자는 새로운 중지 상태(stop state)를 가진 std::stop_source 객체를 생성
- std::nostopstate_t 객체를 받는 생성자는 연관된 중지 상태가 ㅇ벗는 std::stop_source 객체를 생성함

- std::stop_source 클래스의 인터페이스

|멤버 함수|설명|
|:---:|:---:|
|std::stop_source src| 새로운 중지 상태를 가진 std::stop_source 객체 src를 생성함 (기본 생성자)|
|std::stop_source src{nostopstate}|연관된 중지 상태가 없는 std::stop_source 객체 src를 생성|
|src.get_token() | src.stop_possible()이 true라면 연관된 중지 상태에 대한 stop_token 객체를 돌려주고, false라면 기본 생성된 (비어있는) stop_token 객체를 돌려줌
|src.stop_possible() |만일 src로 중지 요청이 가능하다면 true를 return|
|src.stop_requested()| stop_possible이 true이고, 소유자 중 하나가 request_stop()을 호출했으면 true를 return|
|src.request_stop()|만일 src.stop_possible()과 !src.stop_requested()가 둘 다 true면 스레드에 실행 중지를 요청, 그렇지 않으면 아무일도 하지 않음|

- [cppreference: stop_source](https://en.cppreference.com/w/cpp/thread/stop_source)
  - std::jthread의 cancellation과 같은 stop request를 실행할 수 있도록 지원함
  - 일단 stop request가 실행되면 되돌릴 수 없음

```cpp
#include <chrono>
#include <iostream>
#include <stop_token>
#include <thread>
 
using namespace std::chrono_literals;
 
void worker_fun(int id, std::stop_source stop_source) {
    std::stop_token stoken = stop_source.get_token();
    for (int i = 10; i; --i) {
        std::this_thread::sleep_for(300ms);
        if (stoken.stop_requested()) {
            std::printf("  worker%d is requested to stop\n", id);
            return;
        }
        std::printf("  worker%d goes back to sleep\n", id);
    }
}
 
int main() {
    std::jthread threads[4];
    std::cout << std::boolalpha;
    auto print = [](const std::stop_source &source) {
        std::printf("stop_source stop_possible = %s, stop_requested = %s\n",
                    source.stop_possible() ? "true" : "false",
                    source.stop_requested() ? "true" : "false");
    };
 
    // Common source
    std::stop_source stop_source;
    print(stop_source);
 
    // Create worker threads
    for (int i = 0; i < 4; ++i)
        threads[i] = std::jthread(worker_fun, i + 1, stop_source);
 
    std::this_thread::sleep_for(500ms);
 
    std::puts("Request stop");
    stop_source.request_stop();
 
    print(stop_source);
    return 0;
}
```

## std::stop_token

- std::stop_token은 연관된 중지 상태에 대한 "스레드에 안전한" 뷰
- stop_token은 std::jthread에서 가져오거나 std::stop_source 객체로부터 가져온다 (obj.get_token())
- stop_token은 std::jthread나 std::stop_source와 동일한 중지 상태를 공유함
- std::stop_token을 이용하면 연관된 std::stop_source가 스레드에 중지 요청을 한 적이 있는지 알 수 있음

- std::stop_token의 인터페이스

|멤버 함수|설명|
|:---:|:---:|
|std::stop_token stoken|연관된 중지 상태를 가진 중지 토큰을 생성|
|stoken.stop_possible() |만일 stop_token에 연관된 중지 상태가 있다면 true를 반환|
|stoken.stop_requested() | 만일 연관된 stop_source 객체의 멤버 함수 request_stop()이 호출된다면 true를, 아니라면 false를 return|

- 연관된 stop_token을 일시적으로 비활성화 하는 방법
  - 기본 생성된 토큰(연관된 중지 상태가 없는)으로 대체

```cpp
std::jthread jthr([](std::stop_token stoken) {
    ...
    std::stop_token interruptDisabled;  // associated stop state가 없는 객체
    std::swap(stoken, interruptDisabled);  // (1)  jthr의 stop_token이 비활성화 된 상태
    ...                                    // (2) jthr의 stop_token이 비활성화 된 상태
    std::swap(stoken, interruptDisabled);   // 
    ...
}
```

## std::stop_callback

- 이 클래스의 생성자는 stop_token을 위한 하나의 호출 가능 객체를 등록하고, 소멸자에서는 그 객체의 등록을 해제함
- stop_callback의 콜백 함수는
  - 연관된 stop_token의 stop_source에 대해 request_stop을 성공적으로 호출한 스레드에서 호출
  - 생성자가 등록되기 전에 이미 정지가 요청되는 경우, stop_callback을 생성하는 스레드에서 호출
- 동일한 stop_token 객체를 이용하여 하나 이상의 스레드에 대해 여러 개의 stop_callback을 생성 가능
- 실행 순서에 대해서는 보장되지 않음

## 일반적인 신호 전송 메커니즘

- std::stop_source와 std::stop_token의 조합을 신호를 보내기 위한 하나의 일반적인 메커니즘으로 간주할 수 있음
- std::stop_token 객체를 복사하여 뭔가를 실행중인 어떤 단위에 신호를 보낼 수 있음 (std::async, std::promise, std::thread, std::jthread 등을 조합 가능)
- 중지 요청이 왔을 때, std::thread, std::jthread, std::async, std::promise와 같은 실행 단위는 3가지 상태 중 하나일 수 있음
  - 시작되지 않음: 이 경우, stop_requested()는 true를 돌려줌, request_stop()으로 신호가 전송되면 callback이 실행됨
  - 실행 중: 실행 단위가 중지 요청 신호를 받음, 실행 단위가 stop_requested를 호출하기 전 request_stop이 호출되어야 함 (callback 등록 전 request_stop이 호출되야 함)
  - 종료 됨: request_stop 호출은 아무런 효과도 내지 않으며, callback이 호출되지 않음

## std::jthread의 가로채기 지원

- C++20에서 새로 도입됨
- 기존 std::thread (C++11)에 가로채기를 신호하는 기능과 자동으로 부모 스레드에 합류하는 기능을 추가
- stop_token 처리를 위한 jthread의 인터페이스

|멤버 함수|설명|
|:---:|:---:|
|jtr.get_stop_source()|공유된 중지 상태와 연관된 std::stop_source 객체를 돌려줌|
|jtr.get_stop_token()|공유된 중지 상태와 연관된 std::stop_token 객체를 돌려줌|
|jtr.request_stop()|공유된 중지 상태를 통해 실행 중지를 요청함|

## condition_variable_any

- std::condition_variable_any std::conditaion_variable을 일반화한 형식
  - condition_variable 클래스는 다른 스레드가 공유 변수를 수정하고 condition_variable로 타 스레드에 공유할 때까지 여러 스레드를 대기하도록 하는 동기화 기법을 위한 클래스
  - std::mutex를 이용하여 동작 (unlock 후 notify_one 또는 notify_all을 호출)
- std::condition_variable은 std::unique_lock<std::mutex> 형식의 자물쇠만 지원
- condition_variable_any는 멤버 함수 lo.lock()과 lo.unlock()이 있는 임의의 자물쇠 객체를 지원함
  - mutex만 이용 가능 vs user-supplied lock type을 제공

|멤버 함수|설명|
|:---:|:---:|
|notify_one|대기 중인 스레드 중 하나를 깨움|
|notify_all|대기 중인 모든 스레드 전체를 깨움|
|wait|스레드를 차단, 특정 상태 공유 전까지 현재 thread를 중지하고 대기 상태로 변경|
|wait_for|스레드를 차단하고 스레드가 차단 해제되는 시간 간격을 설정|
|wait_util|스레드를 차단하고 스레드가 차단 해제되는 최대 시간을 설정|

- condition_variable_any와 condition_variable
  - wait에 unique_lock<std:mutex>만 포함되느냐 다른 종류의 매개변수가 사용될 수 있느냐의 차이

```cpp
template< class Lock >
void wait( Lock& lock );  // (1)	(since C++11)
template< class Lock, class Predicate >
void wait( Lock& lock, Predicate stop_waiting );  // (2)	(since C++11)
template< class Lock, class Predicate >
bool wait( Lock& lock, std::stop_token stoken, Predicate stop_waiting );  // (3)	(since C++20)
```

```cpp
void wait( std::unique_lock<std::mutex>& lock );  // (1)	(since C++11)
template< class Predicate >
void wait( std::unique_lock<std::mutex>& lock, Predicate stop_waiting );  // (2)	(since C++11)
```

## 요약

- std::stop_source와 std::stop_token, std::stop_callback을 이용하면 스레드와 조건 변수를 협조적으로 가로챌 수 있음
  - 협조적 가로채기란 중지 요청을 받은 스레드가 그것을 무시하고 실행을 계속할 수 있다는 의미
- std::stop_token 객체를 스레드 연산에 전달하고 이후 그 토큰을 이용하여 중지 요청 여부를 조회하거나, std::stop_callback을 통해 callback을 등록 가능
- std::stop_source와 std::stop_token의 조합은 일반적인 신호 전송 메커니즘이라고 할 수 있음
- std::jthread뿐만 아니라 std::condition_variable_any도 중지 요청을 받을 수 있음

## Refs

- [cooperative-interruption](https://www.modernescpp.com/index.php/component/content/article/54-blog/c-20/542-cooperative-interruption-of-a-thread-in-c-20?Itemid=239)
- [condition_variable_any](https://en.cppreference.com/w/cpp/thread/condition_variable_any)
- [condition_variable_any와 condition_variable의 차이](https://stackoverflow.com/questions/8758353/what-is-the-difference-between-stdcondition-variable-and-stdcondition-variab)