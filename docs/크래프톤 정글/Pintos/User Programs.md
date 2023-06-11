# PROJECT 2: USER PROGRAMS

핀토스 2주차 과제인 User Programs의 목표는 유저 프로그램이 시스템 콜을 통해 pintos와 상호작용할 수 있도록 하는 것이다.
따라서 아래 5 단계의 서브 목표를 구현해야 한다.

-   Argument Passing
-   User Memory Access
-   System Calls
-   Process Termination Message
-   Deny Write on Executables

## Background

### 프로그램 실행 과정

1. 디스크에서 실행 가능한 파일을 읽는다
2. 프로그램 실행을 위해 메모리를 할당한다.
3. 프로그램에 매개변수를 전달한다.
4. 유저 프로그램으로 컨텍스트 스위치

### Pintos filesystem

프로그램을 실행하기 위해서는 디스크에서 실행 가능한 파일을 읽어야 한다.

핀토스에서는 virtual disk(모의 디스크)를 생성한 후 유저 프로그램을 해당 디스크에 넣어준 후 실행해야 한다(아마 핀토스가 실제 컴퓨터 하드웨어 위에서 동작하는 것이 아닌 시뮬레이터 위에서 동작하다보니 virtual disk를 만들어주는 것으로 보인다).

다음 명령어를 userprog/build에서 실행할 수 있다.

```zsh
# 10 메가바이트 사이즈의 pintos 파일 시스템 파티션 하나를 포함하는 모의디스크 filesys.dsk를 생성 (실제로 build 디렉터리에 생성된다)
pintos-mkdisk filesys.dsk 10
```

(filesys.dsk가 생성된 모습)

```zsh
#  filesys.dsk 디스크에 tests/userprog/args-single이라는 파일을 args-single이라는 이름으로 넣은 후(p) onearg라는 인자로 args-single 파일을 실행
pintos --fs-disk filesys.dsk -p tests/userprog/args-single:args-single -- -q -f run 'args-single onearg'
```

### Running a program in pintos

아래 명령어를 터미널에 입력하면 핀토스가 부팅되고 filesys.dsk에 있는 'args-single onearg'라는 파일을 실행한다.

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
-   `process.c/process_exec`: `load()` 함수를 호출하고 load에 성공하면 do_ret 함수를 호출한다.
-   `process.c/load`: 페이지 테이블을 생성하고, ELF 헤더를 읽고, 파싱한다. ELF의 data를 data segment에 로드한다. 유저 스택을 생성하고 초기화 한다.
-   `thread.c/do_iret`: 커널 모드를 유저 모드로 전환한다.

## Argument Passing

목표: process_exec() 함수에 있는 "유저 프로그램"을 위한 인자를 세팅

### x86-64 시스템에서의 호출 규약

1. 유저 레벨 어플리케이션은 %rdi, %rsi, %rdx, %rcx, %r8, %r9 시퀸스를 전달하기 위해 정수 레지스터를 사용한다.
2. 호출자는 다음 인스트럭션의 주소를 스택에 푸시하고, 피호출자의 첫번쨰 인스트럭션으로 점프한다. CALL이라는 x86-64 인스트럭션이 이를 수행한다.
3. 피호출자가 실행된다.
4. 만약 피호출자가 리턴 값을 가지고 있다면, 리턴 값은 레지스터 RAX에 저장된다.
5. 피호출자는 x86-64 인스트럭션인 RET을 이용해서, 스택에 푸시했던 리턴 주소를 pop하고 그 주소가 가리키는 곳으로 점프한다.

예시 그림으로 넣기

### 프로그램 시작과 관련된 디테일들

1. 명령을 단어들로 쪼갭시다. `/bin/ls`, `l`, `foo`, `bar` 이렇게
2. 이 단어들을 스택의 맨 처음 부분에 놓습니다. 순서는 상관 없습니다. 왜냐면 포인터에 의해 참조될 예정이기 때문이죠.
3. 각 문자열의 주소 + 경계조건을 위한 널포인터를 스택에 오른쪽→왼쪽 순서로 푸시합니다.
   \*\*이들은 `argv`의 원소가 됩니다.
   널포인터 경계는 `argv[argc]` 가 널포인터라는 사실을 보장해줍니다. C언어 표준의 요구사항에 맞춰서 말이죠.
   그리고 이 순서는 `argv[0]`이 가장 낮은 가상 주소를 가진다는 사실을 보장해줍니다.
   또한 word 크기에 정렬된 접근이 정렬되지 않은 접근보다 빠르므로, 최고의 성능을 위해서는 스택에 첫 푸시가 발생하기 전에 스택포인터를 8의 배수로 반올림하여야 합니다.
4. `%rsi` 가 `argv` 주소(`argv[0]`의 주소)를 가리키게 하고, `%rdi` 를 `argc` 로 설정합니다.
5. 마지막으로 가짜 “리턴 어드레스”를 푸시합니다 : entry 함수는 절대 리턴되지 않겠지만, 해당 스택 프레임은 다른 스택 프레임들과 같은 구조를 가져야 합니다.
