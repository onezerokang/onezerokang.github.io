# [Pintos] 그린반 4팀 주간 공유: User Program

## 1. 공부 방식

-   6월 1일 ~ 6월 7일: gitbook, ppt 자료를 읽으며 구현 -> 시간 내에 다 못하겠다.
-   6월 8일 ~ 6월 10일: blog 코드를 읽고 분석하는 방향으로 선회 -> 디버깅이 어렵다.
-   6월 11일 ~ 6월 12일: 다시 처음부터 구현

## 2. 핀토스에서 프로그램을 실행하는 과정

프로그램을 실행하기 위해서는 디스크에서 실행 가능한 파일을 읽어야 한다.

핀토스에서는 virtual disk(모의 디스크)를 생성하고 유저 프로그램을 해당 디스크에 넣어준 후 실행해야 한다(아마 핀토스가 실제 컴퓨터 하드웨어 위에서 동작하는 것이 아닌 시뮬레이터 위에서 동작하다보니 virtual disk를 만들어주는 것으로 보인다).

다음 명령어를 userprog/build에서 실행할 수 있다.

```zsh
# 10 메가바이트 사이즈의 pintos 파일 시스템 파티션 하나를 포함하는 모의디스크 filesys.dsk를 생성 (실제로 build 디렉터리에 생성된다)
pintos-mkdisk filesys.dsk 10
```

<p align="center"><img src="https://github.com/onezerokang/onezerokang.github.io/assets/60874549/c2298800-6b6b-4fb5-a57b-349ce4c97ae5" width="264px"></p>

<p align="center">_(filesys.dsk가 생성된 모습)_</p>

아래 명령어를 터미널에 입력하면 핀토스가 부팅되고 유저 프로그램을 filesys.dsk에 넣은 후 'args-single onearg'라는 파일을 실행한다.

```zsh
#  filesys.dsk 디스크에 tests/userprog/args-single이라는 파일을 args-single이라는 이름으로 넣은 후(p) onearg라는 인자로 args-single 파일을 실행
pintos --fs-disk filesys.dsk -p tests/userprog/args-single:args-single -- -q -f run 'args-single onearg'
```

pintos의 함수를 보며 어떤 과정을 거쳐 프로그램이 실행되는지 분석해보자.

-   `init.c/main(void)`: 핀토스를 부팅하는 함수
-   `init.c/run_actions(argv)`: 주어진 action을 실행하는 함수(ex: run, put)
-   `init.c/run_task`: 프로그램 실행의 첫 진입점.
-   `process.c/processs_create_initd`: file_name을 복제하고(`load()` 함수와 경쟁하지 않기 위해) `thread_create()` 함수를 호출한다.
-   `thread.c/thread_create`: 스레드 생성 후 initd 함수 실행
-   `process.c/initd`: `process_init()`, `process_exec()` 함수 실행
-   `process.c/process_exec`: `load()` 함수를 호출하고 load에 성공하면 `do_iret` 함수를 호출한다.
-   `process.c/load`: 페이지 테이블을 생성하고, ELF 헤더를 읽고, 파싱한다. ELF의 data를 data segment에 로드한다. 유저 스택을 생성하고 초기화 한다.
-   `thread.c/do_iret`: 커널 모드를 유저 모드로 전환한다.

## 3. Argument Passing

### 3.1. process_exec()

### 3.2. argument_stack()

### 3.3. process_wait()

## 4. 회고

-   정 팀원:
-   이 팀원 2: 프로젝트1 때와 다르게 시간이 많았음에도 불구하고, 스스로 다 구현하지 못해서 아쉬웠다. 그런데 더욱 문제는 완성된 코드들을 참고해서 보았음에도 불구하고 완벽히 이해하지 못하고 넘어와서 마음이 불편하다. 부족함을 많이 느꼈던 시간이었다.
-   강 팀원 3: gitbook을 읽거나 코드 구현을 하다가 모르는 것이 나오면 빠르게 해당 개념을 공부하고 다시 돌아왔어야 했는데, 한번에 너무 많은 것을 공부하려고 하다보니 시간 조절에 실패한 것이 아쉽다. 다음 과제는 이 부분을 주의하며 구현해야겠다.
