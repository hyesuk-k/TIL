<!-- markdownlint-disable-file MD042 MD037 MD033 -->
# 5.4 빗장과 장벽

※ gcc compiler 기준으로 정리

## Contents

- [개요](#개요)
- [std::latch](#stdlatch)
- [std::barrier](#stdbarrier)
- [요약](#요약)
- [Refs](#refs)

## 개요

- 빗장 (latch)과 장벽(barrier) : 카운터가 0이 될 때까지 스레드의 실행을 차단하는 용도로 쓰이는 조정(coordination) 메커니즘들
- std::latch 와 std::barrier 가 표준 라이브러리에 추가됨
  - 해당 멤버 함수를 여러 스레드가 동시에 호출해도 데이터 경쟁(data race)가 발생하지 않음

- std::latch와 std::barrier의 조정 메커니즘 차이는?
  - std::latch는 한 번만 사용 가능, std::barrier는 여러 번 사용 가능
  - std::latch는 여러 스레드가 하나의 작업을 수행하는 상황에 유용
  - std::barrier는 여러 스레드가 작업을 되풀이해서 수행하는 상황에 유용하며, 하나의 함수를 completion step 에서 실행할 수 있다는 장점이 있음
  - 완료 단계 (completion step)이란? 카운터가 0이 된 상태

- C++11과 c++14의 미래 객체, 스레드, 조건 변수와 자물쇠 조합으로는 할 수 없고 latch와 barrier로만 할 수 있는 일이 있는가?
  - 기존과 역할은 동일하지만 사용법이 더 쉽고, non-blocking 을 제공하는 경우로 인해 성능이 더 좋음

## std::latch

- std::latch의 멤버 함수들
|멤버 함수|설명|
|:---:|:---:|
|std::latch lat{cnt}|내부 카운터가 num인 std::latch 객체 lat을 생성|
|lat.count_down(upd = 1)|카운터를 원자적으로 upd만큼 감소, 호출자는 차단되지 않음|
|lat.try_wait()|만일 카운터가 0이면 true를 돌려줌|
|lat.wait()|카운터가 0이면 즉시 반환되고, 그렇지 않으면 카운터가 0이 될 때까지 차단됨|
|lat.arrive_and_wait(upd = 1)|lat.count_down(upd); lat.wait();와 같음|
|std::latch::max|구현이 지원하는 카운터 최댓값을 돌려줌|

- upd의 기본값은 1
- upd가 카운터보다 크거나 음수이면 미정의 행동임

- 책에는 arrive_and_wait이 있는데... [cppreference.com](https://en.cppreference.com/w/cpp/experimental/latch)에는 count_down_and_wait이 있음
  - count_down(size_t n) 은 카운트를 n 만큼 감소 시키지만 대기하지 않음
  - count_down_and_wait() 은 카운트를 1 감소시키고 카운트가 0이 될 때까지 대기
  - wait() 은 카운트가 0이 될 때까지 대기

```cpp
#include <iostream>
#include <latch>
#include <thread>
#include <vector>

constexpr int num_threads = 10;

std::latch start_latch{num_threads};
std::latch end_latch{num_threads};

void worker(int id) {
    start_latch.count_down_and_wait(); // 차단된 상태에서 대기하고 카운트를 1 감소시킵니다.
    std::cout << "Thread " << id << " is running.\n";
    end_latch.count_down(); // 작업이 완료되었으므로 카운트를 1 감소시킵니다.
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < num_threads; ++i) {
        threads.emplace_back(worker, i);
    }

    start_latch.wait(); // 모든 스레드가 시작할 준비가 될 때까지 대기합니다.
    std::cout << "All threads are now running.\n";
    
    end_latch.wait(); // 모든 스레드가 완료될 때까지 대기합니다.
    std::cout << "All threads have completed their work.\n";

    for (auto& t : threads) {
        t.join();
    }
}
```

- std::latch는 모든 작업 스레드가 시작되기 전에 대기, 작업 스레드가 완료될 때까지 대기하는 데 사용

## std::barrier

- std::latch와의 차이
  - std::barrier는 여러 번 사용 가능
  - std::barrier는 다음 단계(phase)를 위해 카운터의 값을 변경하는 수단을 제공
  - 카운터가 0이 되면 완료 단계가 시작됨
    - 완료 단계에서는 생성자로 지정해 둔 호출 가능 객체가 실행됨
      - 모든 스레드가 차단됨
      - 임의의 한 스레디의 차단이 풀리고, 그 스레드가 호출 가능 객체를 실행함
      - 그 호출 가능 객체는 예외를 던지지 말아야 하며, noexcept로 선언된 것이어야 함
      - 호출 가능 객체의 실행이 끝나면 모든 스레드의 차단이 풀림

- std::barrier의 멤버 함수
|멤버 함수|설명|
|:---:|:---:|
|std::barrier bar{cnt} |내부 카운터가 cnt인 std::barrier 객체 bar를 생성|
|std::barrier bar{cnt, call} |내부 카운터가 cnt이고 완료 단계에 호출되는 객체가 call인 std::barrier 객체 bar를 생성|
|bar.arrive(upd) |카운터를 upd만큼 원자적으로 감소|
|bar.wait() |완료 단계가 끝날 때까지 동기화 지점에서 실행을 차단|
|bar.arrive_and_wait() |wait(arrive())와 동등|
|bar.arrive_and_drop() |현재 단계와 다음 단계를 위해 카운터를 1 감소|
|std::barrier::max() |구현이 지원하는 카운터 최댓값(정적 멤버 함수)|

## 요약

- latch와 barrier는 카운터가 0이 될 때까지 스레드릐 실행을 차단하는 용도로 사용되는 조정 메커니즘
- std::latch는 한 번만 사용 가능하고, std::barrier는 여러 번 사용할 수 있음
- std::latch는 여러 스레드가 하나의 작업을 수행하는 상황에 유용함
- std::barrier는 여러 스레드가 작업을 되풀이해서 수행하는 상황에 유용함

## Refs

- [Latches and Barriers](https://www.modernescpp.com/index.php/latches-and-barriers)
