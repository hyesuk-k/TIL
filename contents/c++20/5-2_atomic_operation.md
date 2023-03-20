<!-- markdownlint-disable-file MD042 MD037 MD033 -->
# 5.2 원자적 연산

※ gcc compiler 기준으로 정리

## Contents

- [개요](#개요)
- [std::atomic_ref](#stdatomic_ref)
- [원자적 스마트 포인터](#원자적-스마트-포인터)
- [요약](#요약)
- [Refs](#refs)

## 개요

- 원자적 참조와 원자적 스마트 포인터

- pipelining(여러 개의 작업을 동시에 실행)을 위해 cpu나 컴파일러가 코드를 재배치하여 modification order가 변경됨
- 모든 연산이 원자적
  - 모든 스레드들의 수정 순서가 동일할 때
  - 원자적 연산이 아닌 경우, 모든 스레드에서 같은 수정 순서를 보장하지 않으므로 개발자가 직접 동기화 작업이 필요

- 원자적이란?
  - CPU가 명령어 1개로 처리하는 명령으로 중간에 다른 스레드가 끼어들 여지가 전혀 없는 연산
  - 원자처럼 쪼갤 수 없다는 의미로 atomic이라고 함
  - mutex 없이도 원자적이므로 멀티 스레드 환경에서 연산 속도가 더 빠름
  - C의 원자적 연산 : __sync_fetch_and_add / __sync_fetch_and_sub 등

## std::atomic_ref

- 클래스 템플릿 std::atomic_ref는 참조 객체에 원자적 연산을 적용
- 원자적 객체를 여러 쓰레드가 동시에 읽고 써도 데이터 경쟁(data race)가 발생하지 않음
- 참조되는 객체의 수명은 반드시 atomic_ref의 수명보다 길어야 함
- 어떤 객체를 한 atomic_ref가 참조한다면, 그 객체에 대한 다른 모든 접근도 atomic_ref를 통해 일어나야 함
- atomic_ref가 참조하는 객체의 부분 객체들은 다른 atomic_ref를 통해 접근할 수 없음

```cpp
template< class T >
struct atomic_ref;

template< class T >
struct atomic_ref<T*>;
```

### std::atomic_ref의 특수화들

- 사용자 정의 형식에 대해 std::atomic_ref를 특수화 할 수 있음
- 포인터 형식에 대한 부분 특수화 지원
- 정수 형식이나 부동소수점 형식 같은 산술 형식에 대한 완전 특수화가 제공

#### 기본 템플릿

- 기본 템플릿 std::atomic_ref는 trivially copyable 형식 T로 인스턴스 화 할 수 있음
  - trivially copyable
    - 얕은 복사(shallow copy)가 가능한 타입
    - 단순히 메모리 블록을 복사하여 객체를 복사할 수 있음
    - int, char, char*, 단순 복사 가능한 구조체 
    - 아래에서 Counters는 trivially copyable 타입이므로 c2 = c1과 같이 단순 복사가 가능
      - c1의 메모리 블록을 그대로 c2에 복사 가능하여 빠르게 처리됨

```cpp
struct Counters {
    int a; int b;
};

Counter counter;
std::atomic_ref<Counters> cnt(counter);
```

#### 포인터 형식에 대한 부분 특수화

- 포인터 형식에 대한 부분 특수화 std::atomic_ref<T*> 를 제공

#### 산술 형식에 대한 완전 특수화

- std::atomic_ref<산술_형식>을 제공
- <산술 형식>
  - 문자 형식 : char, char8_t(C++20), char16_t, char32_t, wchar_t
  - 부호 있는 정수 형식 : signed char, short, int, long, long long
  - 부호 없는 정수 형식 : unsigned char/short/int/long/long long
  - <cstdint>20 에 정의된 추가적인 정수 형식:
    - int8_t, int16_t, int32_t, int64_t
    - uint8_t, uint16_t, uint32_t, uint64_t
    - int_fast8_t, int_fast16_t, int_fast32_t, int_fast64_t (가장 빠른 부호 있는 정수)
    - uint_fast8_t, uint_fast16_t, uint_fast32_t, uint_fast64_t
    - int_least8_t, int_least16_t, int_least32_t, int_least64_t (가장 작은 부호 있는 정수)
    - uint_least8_t, uint_least16_t, uint_least32_t, uint_least64_t
    - intmax_t, uintmax_t (가장 큰 부호 있는 정수와 부호 없는 정수)
    - intptr_t, uintptr_t (포인터 값을 담는 부호 있는 정수와 부호 없는 정수)
  - 부동 소수점 형식 : float, double, long double

#### atomic_ref가 지원하는 원자적 연산

- is_lock_free : atomic_ref 객체가 lock-free인지 점검
- is_always_lock_free : 주어진 원자적 형식이 항상 lock-free인지 점검
- load : 참조된 객체의 값을 atomic으로 반환한다
- operator T : 참조된 객체의 값을 atomic으로 반환한다. load()와 동등
- store : 참조된 객체의 값을 주어진 비원자적 객체로 원자적으로 대체
- fetch_add, += / fetch_sub, -= : 원자적으로 주어진 값을 참조된 객체의 값에 더하거나 뺌
- fetch_or, |= / fetch_and, &= / fetch_xor, ^= : 원자적으로 주어진 값과 참조된 객체의 비트 연산 수행
- ++, -- : 참조된 객체를 원자적으로 증가 또는 감소 (전위 / 후위 모두 가능)
- notify_one : 원자적 대기 연산 하나의 차단을 품
- notify_all : 모든 원자적 대기 연산의 차단을 품
- wait : 통지될 때까지 실행을 차단

## 원자적 스마트 포인터

- std::shared_ptr은 control block과 resource로 구성됨
- control block은 thread-safe 함
  - 참조 카운터 수정이 원자적 연산이며, 자원이 정확히 한 번만 삭제됨을 보장
- resource는 thread-safe하지 않음

- thread-safe한 스마트 포인터의 중요성
  - multi-thread에 std::shared_ptr을 사용하는 것은 바람직 하지 않음
  - std::shared_ptr은 가변적(mutable) 포인터라 동기화 되지 않는 읽기 및 쓰기 연산에 이상적임
  - multi-thread 환경에서는 해당 연산에 미정의 행동이 발생할 여지가 많음

- 기존 스마트 포인터의 단점
  - 일관성 문제
    - std::shared_ptr에 대한 atomic operation은 오직 비원자적 데이터 형식에 대한 atomic operation
  - 정확성 문제
    - 전역 atomic operation은 정확한 사용법을 따라야만 제대로 작동함 (개발자의 실수가 발생 가능)
    - 원자적 스마트 포인터를 사용 시, 잘못된 사용법에 대해 type system이 지적
  - 성능 문제
    - 원자적 스마트 포인터는 비원자적 스마트 포인터에 비해 성능이 좋음
    - 원자적 버전은 내부적으로 std::atomic_flog를 저렴한 spinlock 으로 사용 가능
    - 포인터 함수들의 비원자적 버전을 thread에 안전하게 설계하기 위해서는 단일 thread에 불필요한 코드가 추가될 수 있음(단일 thread에서는 성능이 떨어짐)

### std::atomic_flag의 확장

#### C++11의 std::atomic_flag

- std::atomic_flag는 원자적 부울 (atomic boolean) 형식
- boolean 객체의 상태를 true로 설정 혹은 false 로 해제(clear)
- std::atomic_flag는 무잠금이 보장되는 유일한 atomic type
- std::atomic_flag는 더 높ㅇ느 수준의 스레드 추상들을 구현하는 구축 요소로 사용됨
- 한계 : 현재 상태를 수정하지 않고 조회만 하는 멤버 함수가 없음 (C++20에서 추가됨)

#### C++20의 std::atomic_flag 확장

- C++20에서 추가된 interface

|멤버 함수|설명|
|:---:|:---:|
|atomicFlag.clear() | 원자적 플래그를 해제|
|atomicFlag.test_and_set() | 원자적 플래그를 설정하고 기존 값을 돌려줌|
|atomicFlag.test() (C++20) | 원자적 플래그의 값을 돌려줌|
|atomicFlag.notify_one() (C++20) | 원자적 플래그를 기다리는 스레드 하나에 통지|
|atomicFlag.notify_all (C++20) | 원자적 플래그를 기다리는 모든 스레드에 통지|
|atomicFlag.wait(bo) (C++20) | 원자적 플래그가 바뀌어서 통지될 때까지 스레드의 실행을 차단|

- C++20에서 추가된 test() 멤버 함수는 atomic_flag의 값을 변경하지 않고 그대로 돌려줌
- wait, notify_one, notify_all은 스레드 동기화에 유용함
- C++11과 달리 std::atomic_flag의 기본 생성자는 false로 초기화 됨

- is_lock_free
  - 모든 객체의 atomic operation이 lock-free 상태인지 체크하는 함수

- std::atomic<T>::is_always_lock_free
  - true if this atomic type is always lock-free (항상 lock-free인 경우)
  - false if it is never or sometimes lock-free (lock-free가 아닌 경우가 존재할 때)

- sample: promise와 future를 이용한 동기화

```cpp
#include <iostream>
#include <future>
#include <thread>
#include <vector>

std::vector<int> v{};

void prepareWork(std::promise<void> p) {
  v.insert(v.end(), {0, 1, 0, 3});
  std::cout << "Sender: Data prepared." << "\n";
  p.set_value();
}

void completeWork(std::future<void> f) {
  std::cout << "Waiter: Waiting for data." << "\n";
  f.wait();
  v[2] = 2;
  std::cout << "Waiter: Complete the work." << "\n";
  for (auto i: v) std::cout << i << " ";
  std::cout << "\n";
}

int main() {
  std::promise<void> sendNotification;
  auto waitForNotification = sendNotification.get_future();

  std::thread t1(prepareWork, std::move(sendNotification));
  std::thread t2(completeWork, std::move(waitForNotification));

  t1.join();
  t2.join();

  std::cout << "\n";
  return 0;
}
```

- sample: std::atomic_flag를 이용한 동기화

```cpp
#include <iostream>
#include <atomic>
#include <thread>
#include <vector>

std::vector<int> v{};
std::atomic_flag atomicFlag{};

void prepareWork() {
  v.insert(v.end(), {0, 1, 0, 3});
  std::cout << "Sender: Data prepared." << "\n";
  atomicFlag.test_and_set();  // data 준비 후 true로 설정
  atomicFlag.notify_one();
}

void completeWork() {
  std::cout << "Waiter: Waiting for data." << "\n";
  atomicFlag..wait(false);
  v[2] = 2;
  std::cout << "Waiter: Complete the work." << "\n";
  for (auto i: v) std::cout << i << " ";
  std::cout << "\n";
}

int main() {
  std::thread t1(prepareWork);
  std::thread t2(completeWork);

  t1.join();
  t2.join();

  std::cout << "\n";
  return 0;
}
```

### std::atomic의 확장

- C++20 에서 확장됨
- float, double, long 과 같은 부동소수점 형식에 대한 std::atomic 특수화 추가
- 멤버 함수 notify_one, notify_all, wait이 추가되어 std::atomic_flag 뿐만 아니라 std::atomic도 스레드 동기화에 사용 가능
- std::atomic과 atd:atomic_ref의 모든 오나전 특수화와 부분 특수화(boolean, 정수, 부동소수점, 포인터)가 통지와 대기를 지원

## 요약

- std::atomic_ref
  - 참조되는 객체에 원자적 연산들을 적용함
  - 참조되는 객체에 대한 읽기 연산과 쓰기 연산이 원자적이므로 동시에 여러 스레드가 접근해도 데이터 경쟁이 발생하지 않음
  - 참조되는 객체의 수명은 반드시 atomic_ref의 수명보다 길어야 함

- std::shared_ptr
  - 제어 블록과 자원으로 구성
  - 제어 블록은 thread-safe하지만 자원은 thread-safe하지 않음
  - C++에는 원자적 스마트 포인터 형식인 std::atomic<`std::shared_ptr<T>`>와 std::atomic<`std::weak_ptr<T>`>가 추가됨

- std::atomic_flag
  - atomic boolean 형식으로 C++에서 무잠금이 보장되는 유일한 자료구조
  - boolean 상태를 조회하는 멤버 함수와 스레드 동기화를 위한 멤버 함수가 도입됨

## Refs

- [Atomic References with C++20](https://www.modernescpp.com/index.php/atomic-ref)
- [모두의 코드: C++ atomic](https://modoocode.com/271)
- [cppreference : trivially copyable C++11](https://en.cppreference.com/w/cpp/named_req/TriviallyCopyable)
- [모두의 코드: shared_ptr](https://modoocode.com/252)
- [cppreference : is_lock_free](https://en.cppreference.com/w/cpp/atomic/atomic/is_lock_free)
- [cppreference : is_always_lock_free](https://en.cppreference.com/w/cpp/atomic/atomic/is_always_lock_free)
- [c : __sync](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fsync-Builtins.html)

- std::shared_ptr
  - shared_ptr를 통해 접근되는 객체의 수명은 공유 포인터가 공유된 소유권(shared ownership)을 통해 관리
  - 모든 shared_ptr은 객체가 더 이상 필요하지 않게 된 시점에서 객체가 파괴됨을 보장하기 위해 협동함
  - 객체가 가리키던 마지막 std::shared_ptr가 객체를 더 이상 가리키지 않게 되면, 그 std::shared_ptr는 자신이 가리키는 객체를 파괴함
  - reference count (참조 횟수)
    - shared_ptr이 자신이 객체를 가리키는 최후의 포인터임을 아는 방법
    - shared_ptr의 생성자는 count를 증가시키고, 소멸자는 감소, 복사 배정 연산자는 증가와 감소를 모두 수행
      - `sp1 = sp2;`에 의해 sp1이 가리키던 자원의 참조 횟수는 -1, sp2가 가리키던 자원의 참조 횟수는 +1
  - 일반 포인터 크기의 2배
  - 참조 횟수를 담을 메모리를 반드시 동적으로 할당해야 함
  - 참조 횟수의 증가와 감소는 반드시 원자적 연산이여야 함
