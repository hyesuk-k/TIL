<!-- markdownlint-disable-file MD042 MD037 MD033 -->
# 5.6 std::jthread

※ gcc compiler 기준으로 정리

## Contents

- [개요](#개요)

- [요약](#요약)
- [Refs](#refs)

## 개요

- std::jthread는 자동으로 합류하는(join) 기능을 갖춘 스레드
- c++ 11의 std::thread에 소멸자를 통한 자동 합류 기능과 협조적 가로채기 기능을 추가한 스레드
- 예외 발생 시 자동으로 부모 스레드에 합류함

- std::jthread의 인터페이스

|멤버 함수|설명|
|:---:|:---:|
|t.join |스레드 t의 실행이 끝날 때까지 대기|
|t.detach() |생성된 스레드 t를 부모 스레드 (그 스레드를 생성한 스레드)와 독립적으로 실행|
|t.joinable() |만일 t가 합류 가능한 상태라면 true를 return|
|t.get_id() |스레드 t의 id를 return|
|std::this_thread::get_id() |현재 스레드의 id를 return|
|std::jthread::hardware_concurrency() |동시에 실행 가능한 스레드의 개수를 return|
|std::this_thread::sleep_until(absTime) |현재 스레드를 absTime까지 재움|
|std::this_thread::sleep_for(relTime) |현재 스레드를 지속시간 relTime만큼 재움|
|std::this_thread::yield() |시스템이 다른 스레드를 실행할 수 있도록 실행 흐름을 양보함 (yield) |
|t.swap(t2), std::swap(t1,t2) |두 스레드를 맞바꿈 (교환) |
|t.get_stop_source() |공유된 중지 상태와 연관된 중지 출처(std::stop_source 객체)를 return|
|t.get_stop_token() |공유된 중지 상태와 연관된 중지 토큰(std::stop_token 객체)를 return|
|t.request_stop() |공유된 중지 상태를 통해 실행 중지를 요청함|

## 자동 합류

- c++11의 std::thread 예제

 ```cpp
 #include <iostream>
 #include <thread>

 int main() {
    std::cout << std::boolalpah;
    std::thread thr{[] { std::cout << "Joinable std::thread" << "\n"; }};
    std::cout << "thr.joinable() : " << thr.joinable() << "\n";
    std::cout << "\n";
 }
 ```

- c++20의 std::jthread 예제

```cpp
#include <iostream>
#include <thread>

jthread::~jthread() {
    if (joinable()) {  // 스레드 자신이 여전히 합류 가능한지 점검 - join 또는 detach 가 호출된 적이 없다면 합류 가능
        request_stop();  // 실행 중지를 요청
        join();  // 스레드의 실행이 종료될 때까지 차단
    }
}

int main() {
    std::cout << std::boolalpah;
    std::jthread thr{[] { std::cout << "Joinable std::thread" << "\n"; }};
    std::cout << "thr.joinable() : " << thr.joinable() << "\n";
    std::cout << "\n";
}
```

- std::thread의 경우, std::thread의 실행이 종료된 후 주 스레드가 core dump를 발생시킴
- std::jthread의 경우, 소멸자에 의해 thr가 자동으로 주 스레드에 합류, 프로그램이 정상 종료됨

## 협조적 가로채기

```cpp
#include <chrono>
#include <iostream>
#include <thread>

using namespace::std::literals;

int main() {
    std::jthread nonInteruptible([] {
        int counter{0};
        while (counter < 10) {
            std::this_thread::sleep_for(0.2s);
            std::cerr << "nonInterruptible: " << counter++ << "\n";
        }
    });

    std::jthread interruptible([](std::stop_token st) {
        int counter{0};
        while (counter < 10) P
        std::this_thread::sleep_for(0.2s);
        if (st.stop_requested()) return;
        std::cerr << "interruptible: " << counter++ << "\n";
    });

    std::this_thread::sleep_for(1s);

    std::cerr << "Main thread interrupts both jthreads\n";
    nonInterruptible.request_stop();
    interruptible.request_stop();

    std::cout << "\n";
}
```

## 요약

- std::jthread는 c++11의 std::thread에 소멸자를 통한 자동 합류 기능과 협조적 가로채기 기능을 추가한 thread이다.
- 합류와 관련한 std::thread의 행동은 직관적이지 않음
  - std::thread가 합류 가능하면 해당 소멸자에서 std::terminate가 호출됨
  - std::jthread가 합류 가능하면 소멸자는 자동으로 스레드를 합류시킴
- std::jthread는 std::stop_token을 이용하여 협조적으로 가로채기가 가능하다. (협조적 : std::jthread가 중지 요청을 무시 가능함)

## Refs

- [jthread](https://www.modernescpp.com/index.php/a-new-thread-with-c-20-std-jthread)
