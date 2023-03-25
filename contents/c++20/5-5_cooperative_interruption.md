<!-- markdownlint-disable-file MD042 MD037 MD033 -->
# 5.5 협조적 가로채기

※ gcc compiler 기준으로 정리

## Contents

- [개요](#개요)

- [요약](#요약)
- [Refs](#refs)

## 개요

- std::stop_source, std::stop_token, std::callback에 기반함
- std::jthread와 std::condition_variable_any가 협조적 가로채기를 지원

- 스레드 강제 종료 (kill)이 위함한 이유
  - 스레드의 상태를 알지 못하는 상태에서 스레드를 죽이는 것은 위험함
  - 스레드가 작업을 절반만 마친 상태에서 강제로 종료하면 스레드의 작업이 어디까지 진행되었는지 알 수 없음
  - 프로그램의 상태를 알 수 없고 정의되지 않은 행동이므로 어떤 일도 발생 가능함
  - 스레드가 임계 영역 (critical section) 안에 있고, 뮤텍스를 잠근 상태에서 강제 종료하면 교착이 발생될 수 있음

- std::stop_source, std::stop_token, std::stop_callback : 스레드 실행 종료를 비동기적으로 스레드에 요청하거나 스레드가 종료 신호를 받았는지 여부를 조회 가능

## std::stop_source

## std::stop_token

## std::stop_callback

## 일반적인 신호 전송 메커니즘

## std::jthread의 가로채기 지원

## contition_variable_any

## 요약

## Refs

- [cooperative-interruption](https://www.modernescpp.com/index.php/component/content/article/54-blog/c-20/542-cooperative-interruption-of-a-thread-in-c-20?Itemid=239)
