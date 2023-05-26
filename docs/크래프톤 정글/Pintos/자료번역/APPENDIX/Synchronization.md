# Synchronization

스레드 간의 자원 공유가 조심스럽고 제어된 방식으로 이루어지지 않으면 결과는 대개 큰 혼란을 초래합니다.
특히 운영 체제 커널에서는 잘못된 공유가 전체 시스템을 충돌시킬 수 있습니다.
Pintos는 이를 돕기 위해 여러 동기화 원시(primitives)를 제공합니다.

## Disabling Interrupts

> 인터럽트
> 인터럽트는 어떤 신호가 들어왔을 때 CPU를 잠깐 정지시키는 것을 말합ㄴ디ㅏ. 키보드, 마우스 등 I/O 디바이스로 인한 인터럽트, 0으로 숫자를 나누는 산술 연산에서의 인터럽트, 프로세스 오류 등으로 발생합니다.
>
> 인터럽트가 발생되면 인터럽트 핸들러 함수가 모여 있는 인터럽트 벡터로 가서 인터럽트 핸들러 함수가 실행됩니다. 인터럽트 간에는 우선순위가 있고 우선순위에 따라 실행되며 인터럽트는 하드웨어 인터럽트, 소프트웨어 인터럽트 두 가지로 나뉩니다.

가장 원시적인 동기화 방법은 인터럽트를 비활성화하는 것입니다. 즉, CPU가 인터럽트에 응답하지 않도록 일시적으로 막는 것입니다. 인터럽트가 비활성화되면 다른 스레드는 실행 중인 스레드를 선점하지 않습니다. 왜냐하면 스레드의 선점은 타이머 인터럽트에 의해 이루어지기 때문입니다. 일반적으로 인터럽트가 켜져 있다면 실행 중인 스레드는 C 문장 사이나 하나의 실행 내에서라도 언제든지 다른 스레드에 의해 선점될 수 있습니다.

그런데 이는 Pintos가 "선점 가능(preemptible) 커널"이라는 것을 의미합니다. 즉, 커널 스레드는 언제든지 선점될 수 있습니다. 전통적인 유닉스 시스템은 "비선점 불가(nonpreemptible)"인 반면, 커널 스레드는 스케줄러로 명시적으로 호출되는 지점에서만 선점될 수 있습니다. (사용자 프로그램은 두 모델 모두에서 언제든지 선점될 수 있습니다.) 상상하시겠지만, 선점 가능한 커널은 보다 명시적인 동기화를 필요로 합니다.

일반적으로 인터럽트 상태를 직접 설정할 필요는 거의 없을 것입니다. 대부분의 경우에는 다음 섹션에서 설명하는 다른 동기화 원시(primitives)를 사용해야 합니다. 인터럽트를 비활성화하는 주된 이유는 커널 스레드와 외부 인터럽트 핸들러를 동기화하기 위한 것입니다. 외부 인터럽트 핸들러는 슬립할 수 없으므로 대부분의 다른 동기화 방법을 사용할 수 없습니다.

일부 외부 인터럽트는 인터럽트를 비활성화해도 연기할 수 없습니다. 이러한 인터럽트를 비마스킹 인터럽트(non-maskable interrupts, NMIs)라고 하며, 비상 시에만 사용되어야 합니다. 예를 들어 컴퓨터가 화재가 발생했을 때 등입니다. Pintos는 비마스킹 인터럽트를 처리하지 않습니다.

인터럽트를 비활성화하거나 활성화하는 데 사용되는 타입과 함수는 `include/threads/interrupt.h`에 정의되어 있습니다.

---

```c
enum intr_level;
```

> One of INTR_OFF or INTR_ON, denoting that interrupts are disabled or enabled, resepectively.

---

```c
enum intr_level intr_get_level (void)
```

> Returns the current interrupt state.

---

```c
enum intr_level intr_set_level (enum intr_level level);
```

> Turns interrupts on or off according to level. Returns the previous interrupt state.

---

```c
enum intr_level intr_enable (void);

```

> Turns interrupts on. Returns the previos interrupt state.

---

```c
enum intr_level intr_disable (void);

```

> Turns interrupts off. Returns the previous interrupt state.

## Semaphores

세마포어(Semaphore)는 음수가 아닌 정수와 이를 원자적으로 조작하는 두 개의 연산자로 구성됩니다:

-   "Down" 또는 "P": 값이 양수가 될 때까지 기다린 후 값을 감소시킵니다.
-   "UP" 또는 "V": 값을 증가시킵니다 (그리고 대기 중인 스레드 중 하나를 깨웁니다).

0으로 초기화된 세마포어는 정확히 한 번 발생할 이벤트를 기다리는 데 사용될 수 있습니다. 예를 들어, 스레드 A가 다른 스레드 B를 시작하고 B가 어떤 활동이 완료되었다는 신호를 기다리려고 합니다. A는 0으로 초기화된 세마포어를 생성하고 B에게 시작될 때 전달한 다음 세마포어를 "down"합니다. B가 활동을 완료하면 세마포어를 "up"합니다. 이는 A가 세마포어를 "down"하거나 B가 먼저 "up"하는지에 관계없이 작동합니다.

예를 들어:

```c
struct semaphore sema;

/* Thread A */
void threadA (void) {
    sema_down (&sema);
}

/* Thread B */
void threadB (void) {
    sema_up (&sema);
}

/* main function */
void main (void) {
    sema_init (&sema, 0);
    thread_create ("threadA", PRI_MIN, threadA, NULL);
    thread_create ("threadB", PRI_MIN, threadB, NULL);
}
```

이 예시에서, 스레드 A는 `sema_down()`에서 실행을 중지하고, 스레드 B가 `sema_up()`을 호출할 때까지 기다립니다.

1로 초기화된 세마포어는 일반적으로 리소스에 대한 액세스를 제어하는 데 사용됩니다. 코드 블록이 리소스를 사용하기 시작하기 전에 세마포어를 "down"하고, 리소스 사용이 끝나면 "up"합니다. 이 경우에는 아래에서 설명하는 락(lock)이 더 적합할 수 있습니다.

세마포어는 1보다 큰 값으로 초기화될 수도 있습니다. 그러나 이는 거의 사용되지 않습니다. 세마포어는 Edsger Dijkstra에 의해 발명되었으며 THE 운영 체제에서 처음 사용되었습니다. Pintos의 세마포어 유형과 연산은 `include/threads/synch.h`에 선언되어 있습니다.

---

```c
struct semaphore;
```

> Represents a semaphore.

---

```c
void sema_init (struct semaphore *sema, unsigned value);
```

> Initializes sema as a new semaphore with the given initial value.

---

```c
void sema_down (struct semaphore *sema);
```

> Executes the "down" or "P" operation on sema, waiting for its value to become positive and then decrementing it by one.

---

```c
bool sema_try_down (struct semaphore *sema);
```

> Tries to execute the "down" or "P" operation on sema, without waiting. Returns true if sema was successfully decremented, or false if it was already zero and thus could not be decremented without waiting. Calling this function in a tight loop wastes CPU time, so use sema_down() or find a different approach instead.

---

```c
void sema_up (struct semaphore *sema);
```

> Executes the "up" or "V" operation on sema, incrementing its value. If any threads are waiting on sema, wakes one of them up. Unlike most synchronization primitives, sema_up() may be called inside an external interrupt handler.

세마포어는 인터럽트 비활성화(disabling interrupt)와 스레드의 차단 및 차단 해제(`thread_block()` 및 `thread_unblock()`)를 통해 내부적으로 구현됩니다. 각 세마포어는 대기 중인 스레드의 목록을 유지하며, 이는 `lib/kernel/list.c`에 있는 연결 리스트 구현을 사용합니다.

## Locks

잠금(locks)은 초기 값이 1인 세마포어와 비슷합니다(세마포어에 대해서는 Semaphores를 참조하십시오). 잠금(lock)의 "up" 작업은 "release"라고 불리며, "down" 작업은 "acquire"라고 불립니다.

세마포어와 비교하여 잠금(lock)은 추가적인 제한이 있습니다: 잠금을 획득한 스레드(잠금의 "소유자"라고 불립니다)만이 잠금을 해제할 수 있습니다. 이 제한이 문제가 되면 잠금(lock) 대신 세마포어를 사용하는 것이 좋은 신호입니다.

Pintos에서의 잠금(lock)은 "재귀적(recursive)"이 아닙니다. 즉, 현재 잠금을 보유한 스레드가 해당 잠금을 다시 획득하려고 하는 것은 오류입니다. 잠금(lock)의 유형과 함수는 `include/threads/synch.h`에 선언되어 있습니다.

---

```c
struct lock
```

> Represents a lock.

---

```c
void lock_init (struct lock *lock);
```

> Initializes lock as a new lock. The lock is not initially owned by any thread.

---

```c
void lock_acquire (struct lock *lock);
```

> Acquires lock for the current thread, first waiting for any current owner to release it if necessary.

---

```c
bool lock_try_acquire (struct lock *lock);
```

> Tries to acquire lock for use by the current thread, without waiting. Returns true if successful, false if the lock is already owned. Calling this function in a tight loop is a bad idea because it wastes CPU time, so use lock_acquire() instead.

---

```c
void lock_release (struct lock *lock);
```

> Releases lock, which the current thread must own.

---

```c
bool lock_held_by_current_thread (const struct lock *lock):
```

> Returns true if the running thread owns lock, false otherwise. There is no function to test whether an arbitrary thread owns a lock, because the answer could change before the caller could act on it.

## Monitors

모니터(monitor)는 세마포어나 잠금보다 더 높은 수준의 동기화 형태입니다. 모니터는 동기화되는 데이터와 잠금인 모니터 잠금(monitor lock), 그리고 하나 이상의 조건 변수로 구성됩니다. 스레드가 보호되는 데이터에 접근하기 전에 먼저 모니터 잠금을 획득해야 합니다. 그런 다음 "모니터 안에 있다"고 표현됩니다. 모니터 안에 있는 동안, 스레드는 보호되는 모든 데이터를 자유롭게 검사하거나 수정할 수 있습니다. 보호된 데이터에 대한 접근이 완료되면, 모니터 잠금을 해제합니다.

조건 변수(condition variable)는 모니터 내의 코드가 조건이 참이 될 때까지 대기할 수 있도록 합니다. 각 조건 변수는 추상적인 조건과 관련이 있습니다. 예를 들어, "일부 데이터가 처리를 위해 도착했다" 또는 "사용자의 마지막 키 입력으로부터 10초가 경과했다"와 같은 조건입니다. 모니터 내의 코드가 조건이 참이 될 때까지 기다려야 하는 경우, 관련된 조건 변수에 "대기"하며, 이는 잠금을 해제하고 조건이 신호를 받을 때까지 대기합니다. 반대로, 조건 중 하나를 참으로 만들었다면, 이는 대기 중인 스레드 중 하나를 깨우거나 해당 조건을 모두 깨우기 위해 "신호"를 보냅니다.

모니터에 대한 이론적인 기반은 C.A.R. 호어(C.A.R. Hoare)에 의해 제시되었습니다. 그들의 실제적인 사용법은 후에 Mesa 운영 체제에 대한 논문에서 자세히 설명되었습니다. 조건 변수의 유형과 함수는 `include/threads/synch.h`에 선언되어 있습니다.

---

```c
struct condition;
```

> Represents a condition variable.

---

```c
void cond_init (struct condition *cond);
```

> Initializes cond as a new condition variable.

---

```c
void cond_wait (struct condition *cond, struct lock *lock);
```

> Atomically releases lock (the monitor lock) and waits for cond to be signaled by some other piece of code. After cond is signaled, reacquires lock before returning. lock must be held before calling this function. Sending a signal and waking up from a wait are not an atomic operation. Thus, typically cond_wait()'s caller must recheck the condition after the wait completes and, if necessary, wait again. See the next section for an example.

---

```c
void cond_signal (struct condition *cond, struct lock *lock);
```

> If any threads are waiting on cond (protected by monitor lock lock), then this function wakes up one of them. If no threads are waiting, returns without performing any action. lock must be held before calling this function.

---

```c
void cond_broadcast (struct condition *cond, struct lock *lock);
```

> Wakes up all threads, if any, waiting on cond (protected by monitor lock lock). lock must be held before calling this function.

### Monitor Example

모니터의 고전적인 예는 "생산자(producer)" 스레드가 문자를 작성하고 "소비자(consumer)" 스레드가 문자를 읽는 버퍼를 처리하는 것입니다. 이를 구현하기 위해서는 모니터 잠금(lock) 외에도 `not_full`과 `not_empty`라고 불리는 두 개의 조건 변수가 필요합니다.

```c
    char buf[BUF_SIZE];     /* Buffer. */
    size_t n = 0;         /* 0 <= n <= BUF SIZE: # of characters in buffer. */
    size_t head = 0;        /* buf index of next char to write (mod BUF SIZE). */
    size_t tail = 0;         /* buf index of next char to read (mod BUF SIZE). */
    struct lock lock;         /* Monitor lock. */
    struct condition not_empty; /* Signaled when the buffer is not empty. */
    struct condition not_full;     /* Signaled when the buffer is not full. */

    ...initialize the locks and condition variables...

    void put (char ch) {
      lock_acquire (&lock);
      while (n == BUF_SIZE)    /* Can't add to buf as long as it's full. */
        cond_wait (&not_full, &lock);
      buf[head++ % BUF_SIZE] = ch;    /* Add ch to buf. */
      n++;
      cond_signal (&not_empty, &lock);    /* buf can't be empty anymore. */
      lock_release (&lock);
    }

    char get (void) {
      char ch;
      lock_acquire (&lock);
      while (n == 0)        /* Can't read buf as long as it's empty. */
        cond_wait (&not_empty, &lock);
      ch = buf[tail++ % BUF_SIZE];    /* Get ch from buf. */
      n--;
      cond_signal (&not_full, &lock);    /* buf can't be full anymore. */
      lock_release (&lock);
    }
```

위의 코드가 완전히 정확하게 동작하려면 `BUF_SIZE`가 `SIZE_MAX + 1`로 나누어 떨어져야 함에 유의하십시오. 그렇지 않으면 head가 0으로 되돌아갈 때 처음에 실패할 수 있습니다. 실제로는 `BUF_SIZE`는 보통 2의 거듭제곱으로 설정됩니다.

## Optimization Barriers

최적화 장벽(optimization barrier)은 컴파일러가 장벽을 통해 메모리 상태에 대한 가정을 하지 못하도록 하는 특수한 문장입니다. 최적화 장벽을 통해 컴파일러는 장벽을 넘어서 변수의 읽기 또는 쓰기를 재배치하지 않거나, 장벽을 통과한 변수의 값을 변경하지 않는다고 가정하지 않습니다. 다만, 주소가 절대 사용되지 않는 지역 변수에 대해서는 예외입니다. Pintos에서는 `include/threads/synch.h`에서 최적화 장벽으로 사용되는 `barrier()` 매크로를 정의하고 있습니다.

최적화 장벽을 사용하는 이유 중 하나는 데이터가 컴파일러의 인식 없이 비동기적으로 변경될 수 있는 경우입니다. 다른 스레드나 인터럽트 핸들러에 의해 데이터가 변경될 수 있습니다. `devices/timer.c`의 `too_many_loops()` 함수가 예시입니다. 이 함수는 다음과 같이 타이머 틱이 발생할 때까지 루프에서 바쁜 대기(busy-waiting)를 합니다:

```c
/* 타이머 틱을 기다립니다. */
int64_t start = ticks;
while (ticks == start)
    barrier();
```

루프 내에 최적화 장벽이 없다면, 컴파일러는 루프가 종료되지 않을 것이라고 결론을 내릴 수 있습니다. 왜냐하면 start와 ticks는 처음에 동일하고, 루프 자체에서 이들을 변경하지 않기 때문입니다. 이 경우, 컴파일러는 함수를 무한 루프로 "최적화"할 수 있으며, 이는 확실히 원하지 않는 결과입니다.

최적화 장벽은 다른 컴파일러 최적화를 피하기 위해 사용될 수도 있습니다. `devices/timer.c`의 `busy_wait()` 함수도 예시입니다. 다음과 같은 루프를 포함하고 있습니다:

```c
while (loops-- > 0)
  barrier();
```

이 루프의 목적은 loops 값을 원래 값에서 0까지 줄여가며 바쁜 대기(busy-waiting)하는 것입니다. 최적화 장벽이 없으면, 컴파일러는 루프를 완전히 제거할 수 있습니다. 왜냐하면 유용한 결과를 생성하지 않고 부작용도 없기 때문입니다. 최적화 장벽은 컴파일러에게 루프 본문이 중요한 영향을 미친다고 가정하도록 합니다.

마지막으로, 최적화 장벽은 메모리 읽기 또는 쓰기의 순서를 강제로 지정하는 데 사용될 수 있습니다. 예를 들어, "기능"을 추가한다고 가정해보겠습니다. 타이머 인터럽트가 발생할 때마다 전역 변수 `timer_put_char`에 있는 문자를 콘솔에 출력하지만, 전역 부울 변수 `timer_do_put`이 true일 때에만 출력합니다. 이 때, `x`를 출력하기 위해 최적화 장벽을 사용하는 것이 가장 좋습니다:

```c
timer_put_char = 'x';
barrier ();
timer_do_put = true;
```

최적화 장벽이 없으면, 코드는 버그가 됩니다. 왜냐하면 컴파일러는 순서를 유지할 이유가 없다고 판단하여 연산의 순서를 변경할 수 있기 때문입니다. 이 경우, 컴파일러는 할당의 순서가 중요하다는 사실을 알지 못하기 때문에 최적화기가 순서를 바꿀 수 있습니다. 실제로 이렇게 할 지 여부를 알 수 없으며, 컴파일러에게 다른 최적화 플래그를 전달하거나 다른 버전의 컴파일러를 사용하면 다른 동작이 나올 수도 있습니다.

다른 해결책은 할당문 주변에서 인터럽트를 비활성화하는 것입니다. 이는 재배치는 방지하지만, 할당문 사이에 인터럽트 핸들러가 개입하지 못하게 합니다. 또한 인터럽트를 비활성화하고 다시 활성화하는 데 추가적인 실행 시간 비용이 발생합니다:

```c
enum intr_level old_level = intr_disable ();
timer_put_char = 'x';
timer_do_put = true;
intr_set_level (old_level);
```

두 번째 해결책은 `timer_put_char`와 `timer_do_put`의 선언을 `volatile`로 표시하는 것입니다. 이 키워드는 변수가 외부에서 관찰 가능하며 최적화의 여지를 제한한다고 컴파일러에게 알려줍니다. 그러나 `volatile`의 의미는 명확하게 정의되어 있지 않으므로 일반적인 해결책으로는 좋지 않습니다. 기본 Pintos 코드는 전혀 `volatile`을 사용하지 않습니다.

마지막으로, 잠금(lock)은 인터럽트를 방지하거나 잠금이 유지되는 영역에서 코드의 재배치를 방지하지 않기 때문에 해결책이 될 수 없습니다:

```c
lock_acquire (&timer_lock);        /* 잘못된 코드 */
timer_put_char = 'x';
timer_do_put = true;
lock_release (&timer_lock);
```

컴파일러는 외부에서 정의된 함수를 호출하는 것을 한정된 형태의 최적화 장벽으로 취급합니다. 구체적으로 컴파일러는 외부에서 정의된 함수가 정적으로 또는 동적으로 할당된 데이터 및 주소가 취해진 지역 변수에 액세스할 수 있다고 가정합니다. 이는 명시적인 장벽을 생략할 수 있는 이유 중 하나이며, Pintos에는 명시적인 장벽이 거의 없는 이유 중 하나입니다.

같은 소스 파일에 정의된 함수나 소스 파일에서 포함된 헤더에 정의된 함수는 최적화 장벽으로 신뢰할 수 없습니다. 함수 정의 이전에 함수를 호출하는 경우에도 마찬가지입니다. 왜냐하면 컴파일러는 최적화를 수행하기 전에 전체 소스 파일을 읽고 구문 분석하기 때문입니다.
