<!-- markdownlint-disable-file MD042 MD037 MD033 -->
# 5.7 동기화된 출력 스트림 객체

※ gcc compiler 기준으로 정리 (2020년 기준으로 gcc 11만 지원 / 2021년 9월 기준 msvc도 지원)

## Contents

- [내용](#내용)
- [요약](#요약)
- [Refs](#refs)

## 내용

- C++11 표준은 std::cout 의 스레드 안전성을 보장
  - 단, 하나의 스트림에 여러 개의 문자열을 출력하는 경우 다른 스레드가 출력한 문자열들과 섞일 수 있음
- C++11은 lock_guard와 같은 잠금 수단을 이용하여 std::cout 에 대한 접근을 동기화
- C++20에서는 std::basic_syncbuf, std::basic_osyncstream 등을 활용하여 여러 스레드가 출력한 문자들이 뒤섞이는 현상이 발생하지 않도록 지원 가능

## 요약

- std::cout은 스레드에 안전하지만, 여러 스레드가 동시에 std::cout에 출력하면 문자열이 뒤섞여 나올 수 있음
  - 이는 시각적인 문제일 뿐, 데이터 경쟁은 아님
- c++20에서는 동기화된 출력 스트림 객체를 지원
  - 출력할 내용을 내부 버퍼에 누적하고, 버퍼 내용 전체를 원자적 단계로 출력함
  - 스트림 출력 연산이 동시에 진행 되더라도 문자열이 섞이는 현상은 발생하지 않음ㄴㄴㄴㄴ

## Refs

- [Synchronized-outputstreams](https://www.modernescpp.com/index.php/component/content/article/54-blog/c-20/546-synchronized-outputstreams?Itemid=239)
