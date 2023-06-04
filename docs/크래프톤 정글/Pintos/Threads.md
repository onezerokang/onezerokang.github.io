# PROJECT 1: THREADS

PintOS의 첫번째 프로젝트는 최소의 기능만 하는 thread system을 확장하면서 스레드와 동기화에 대해 이해하는 것이다.

다음은 threads 프로젝트에서 구현해야 할 목표들이다.

-   Alarm clock
-   Priority Scheduling

## 자주 사용되는 함수

-   `thread_current()`: 현재 실행중인 스레드를 반환한다.
-   `intr_disable()`: 인터럽트를 비활성화 하고 이전 인터럽트 상태를 반환한다.
-   `intr_set_level()`: 인터럽트를 인자로 주어진 값으로 설정하고 이전 인터럽트 상태를 반환한다.
-   `list_insert_ordered()`: list에 값을 넣고 정렬한다.
-   `schedule()`: ...

## Alarm clock

목표: busy waiting 방식으로 구현된 alarm을 sleep list를 이용하여 sleep/wakeup 방식으로 수정하기

추가, 수정해야 할 부분

-   `struct list sleep_list`:
-   `thread_init()` 함수
-   `timer_sleep()` 함수
-   `timer_interrupt()` 함수

### busy waiting 알람의 문제점

다음은 busy waiting 방식으로 구현된 알림이 동작하는 과정이다.

1. 스레드에서 `timer_sleep(int64_t ticks)` 함수가 호출된다.
2. 스레드는 깨어나는 시점, ticks에 도달할 때까지 loop를 돌며 스레드를 ready list에 넣는다.

이러한 방식은 CPU 사이클을 소모하기 떄문에 비효율적이다.

### sleep list를 이용한 sleep/wakeup 알람

1. 스레드에서 `timer_sleep(int64_t ticks)` 함수가 호출된다.
2. 스레드의 상태를 blocked로 변경하고 sleep_list에 넣는다.
3. `timer_interrupt()`가 호출될 때마다 현재 ticks와 wakeup_tick을 비교한다.
4. 깨어나야할 스레드가 존재한다면 해당 스레드의 상태를 ready로 변경하고 ready list에 넣는다.

### 구현

코드와 설명 입력할 것

## Priority scheduling

목표

추가, 수정해야할 함수

###

### Change Synchronization

### Priority Donation
