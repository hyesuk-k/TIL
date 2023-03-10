<!-- markdownlint-disable-file MD042 MD037 MD033 -->
# 5.2 원자적 연산

※ gcc compiler 기준으로 정리

## Contents

- [개요](#개요)
- [요약](#요약)
- [Refs](#refs)

## 개요

- 원자적 참조와 원자적 스마트 포인터

## std::atomic_ref

- 클래스 템플릿 std::atomic_ref는 참조되는 객체에 원자적 연산을 적용
- 원자적 객체를 여러 쓰레드가 동시에 읽고 써도 데이터 경쟁(data race)가 발생하지 않음
- 참조되는 객체의 수명은 반드시 atomic_ref의 수명보다 길어야 함
- 어떤 객체를 한 atomic_ref가 참조한다면, 그 객체에 대한 다른 모든 접근도 atomic_ref를 통해 일어나야 함
- atomic_ref가 참조하는 객체의 부분 객체들은 다른 atomic_ref를 통해 접근할 수 없음

### std::atomic_ref의 특수화들

- 사용자 정의 형식에 대해 std::atomic_ref를 특수화 할 수 있음
- 포인터 형식에 대한 부분 특수화 지원
- 정수 형식이나 부동소수점 형식 같은 산술 형식에 대한 완전 특수화가 제공

#### 기본 템플릿

- 기본 템플릿 std::atomic_ref는 trivially copyable 형식 T로 인스턴스 화 할 수 있음

```cpp
struct Counters {
    int a;
    int b;
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

#### C++20의 std::atomic_flag 확장

### std::atomic의 확장

## 요약

## Refs
