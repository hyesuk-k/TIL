<!-- markdownlint-disable-file MD042 MD037 MD033 -->
# 5.3 세마포어

※ gcc compiler 기준으로 정리

## Contents

- [개요](#개요)
- [세마포어](#세마포어)
- [요약](#요약)
- [Refs](#refs)

## 개요

- 공유 자원에 대한 동시적인 접근들을 제어하는 데 쓰이는 동기화 매커니즘
- C++20 에서는 0보다 큰 카운터를 가진 특별한 세마포어인 counting semaphore를 구현한 클래스 템플릿 std::counting_semaphore를 제공
  - 세마포어 객체의 카운터는 생성자에서 초기화 됨
  - 세마포어를 획득(또는 확보; acquire) : 카운터가 감소
  - 세마포어를 해제(release) : 카운터가 증가
- 카운터가 0인 세마포어를 어떤 스레드가 획득하려 하면, 그 스레드는 다른 스레드가 그 세마포어를 해제해서 카운터가 0보다 커질 때까지 차단됨

## 세마포어

- std::counting_semaphore<1>의 별칭인 std::binary_semaphore를 제공
`using binary_semaphore = std::counting_semaphore<1>;`

|멤버 함수|설명|
|:---:|:---:|
|std::semaphore sem{num}|세마포어 sem을 생성한다. 내부 카운터는 num으로 초기화|
|sem.max() (정적 멤버 함수)|카운터의 최댓값을 돌려줌|
|sem.release(upd = 1)|카운터를 upd만큼 증가, 세마포어 sem을 획득하려는 스레드들의 차단을 푼다|
|sem.acquire()|카운터를 1 증가하거나, 카운터가 0보다 커질 때까지 실행을 차단|
|sem.try_acquire()|카운터 감소를 시도. 카운터가 0이어도 차단되지 않음|
|sem.try_acquire_for(relTime)|카운터 감소를 시도, 카운터가 0이면 지금부터 최대 relTime 시간 동안 차단|
|sem.try_acquire_until(absTime)|카운터 감소를 시도, 카운터가 0이면 최대 absTime 시간까지 차단|

## 요약

- 공유 자원에 대한 동시 접근들을 제어하는 데 쓰이는 동기화 메커니즘
- std::counting_semaphore는 카운터를 가진 세마포어
  - 스레드가 이 세마포어를 획득하면 카운터가 감소하고, 해제하면 카운터가 증가함
  - 카운터가 0인 세마포어의 획득을 시도한 스레디는 다른 스레드가 그 세마포어를 해제해서 카운터가 증가될 때까지 차단됨 (설정마다 다름)

## Refs

- [joinc-posix semaphore](https://www.joinc.co.kr/w/Site/system_programing/IPC/semaphores)
- [man7.org - semaphore](https://man7.org/linux/man-pages/man0/semaphore.h.0p.html)

## Ex : posix semaphore

- `#include <semaphore.h>`
- `int sem_init(sem_t *sem, int pshared, unsigned int value);`
  - semapohre 생성
  - sem : 초기화할 세마포어 객체
  - pshared : 0이 아니면 프로세스간 세마포어 공유, 0이면 프로세스 내부에서만 사용
  - value : 세마포어 초기 값
- `sem_t *sem_open(const char *name, int oflag, mode_t mode, unsigned int value);`
  - 이름 있는 semaphore 생성
  - 이름 있는 semaphore는 파일로 생성되며, /dev/shm 에 생성됨 (해당 위치 mount 필요)
- `int sem_wait(sem_t *sem);`
  - 세마포어 값이 0보다 크면 프로세스는 세마포어를 획득함
  - 세마포어 값이 0이라면 세마포어가 0보다 커지거나 시그널이 발생할 때까지 대기
- `int sem_trywait(sem_t *sem);`
  - non-blocking 함수
  - 이외에는 sem_wait과 동일
- `int sem_timedwait(sem_t *sem, const struct timespec *abs_timeout);`
  - timeout 시간 동안의 제한을 가짐 (sec + nsec 조합)
- `int sem_getvalue(sem_t *sem, int *sval);`
  - 현재 세마포어의 값을 획득 (sval)
- `int sem_post(sem_t *sem);`
  - 세마포어 값을 되돌림
  - 세마포어의 값이 1 증가함
- `int sem_destroy(sem_t *sem);`
  - 세마포어 객체 삭제
- `int sem_unlink(const char *name);`
  - 세마포어 삭제
  - 이름 있는 세마포어를 삭제

- 상호배제 알고리즘
- mutex는 key가 1개, semaphore는 key를 설정 가능
- 장점 : critical section 에서 충돌이 나지 않음
- 단점 : 많은 thread에서 block 발생 가능 / CPU 낭비 발생 가능 (세마포어를 획득할 때까지 대기)

```cpp
#include <stdio.h>
#include <semaphore.h>
#include <unistd.h>
#include <stdlib.h>
#include <pthread.h>

#define MAX_THREAD_NUM  2
int count = 0;
sem_t mysem;
void *func(void *data) {
    pthread_t id;
    int tmp;
    id = pthread_self();
    printf("Thread %lu created.\n", id);
    while(true) {
        sem_wait(&mysem);
        tmp = count;
        tmp = tmp + 1;
        sleep(1);
        count = tmp;
        printf("%lu : %d\n", id, count);
        sem_post(&mysem);
        usleep(1);
    }
}
int main(int argc, char **argv) {
    pthread_t p_thread[MAX_THREAD_NUM];
    int thr_id;
    int status;
    int i = 0;

    if (sem_init(&mysem, 0, 1) == -1) {
        perror("Error");
        exit(0);
    }
    for( i = 0; i < MAX_THREAD_NUM; i++) {
        thr_id = pthread_create(&p_thread[i], NULL, func, (void *)&i);
        if (thr_id < 0) {
            perror("thread create error : ");
            exit(0);
        }
    }
    pthread_join(p_thread[0], NULL);
    pthread_join(p_thread[1], NULL);
    return 0;
}
```
