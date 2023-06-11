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
-   `thread.c/do_iret`: 커널 모드를 유저 모드로 전환하고 프로세스를 시작한다.

## 3. Argument Passing

### 3.1. process_exec()

```c
/* Switch the current execution context to the f_name.
 * Returns -1 on fail. */
int
process_exec (void *f_name) {
	char *file_name = f_name;
	bool success;

	/* We cannot use the intr_frame in the thread structure.
	 * This is because when current thread rescheduled,
	 * it stores the execution information to the member. */
	struct intr_frame _if;
	_if.ds = _if.es = _if.ss = SEL_UDSEG;
	_if.cs = SEL_UCSEG;
	_if.eflags = FLAG_IF | FLAG_MBS;

	/* We first kill the current context */
	process_cleanup ();

	char *parse[64];
	char *token, *save_ptr;
	int count = 0;

	for(token = strtok_r(file_name, " ", &save_ptr); token!=NULL; token=strtok_r(NULL, " ", &save_ptr)){
		parse[count++] = token;
	}
	/* And then load the binary */
	success = load (file_name, &_if);


	argument_stack(parse, count, &_if.rsp);
	_if.R.rdi = count;
	_if.R.rsi = (char *)_if.rsp +8 ;

	hex_dump(_if.rsp, _if.rsp, USER_STACK - (uint64_t)_if.rsp, true);

	/* If load failed, quit. */
	palloc_free_page (file_name);

	if (!success)
		return -1;
	/* Start switched process. */
	do_iret (&_if);
	NOT_REACHED ();
}
```

### 3.2. argument_stack()

```c
void argument_stack(char **parse, int count, void **rsp) {
	// 프로그램 이름, 인자 문자열 push
    for (int i = count - 1; i >= 0; i--)
    {
        for (int j = strlen(parse[i]); j >=0 ; j--)
        {
            (*rsp)--;                      // 스택 주소 감소
            **(char **)rsp = parse[i][j]; // 주소에 문자 저장
        }
        parse[i] = *(char **)rsp; // parse[i]에 현재 rsp의 값 저장해둠(지금 저장한 인자가 시작하는 주소값)
    }

 	while ((int)(*rsp) % 8 != 0) { //스택 포인터가 8의 배수가 되도록
        (*rsp)--;  // 스택 포인터를 1바이트씩 이동
        **(uint8_t **)rsp = 0;
    }

    for (int i = count; i >= 0; i--) {
        (*rsp) -= 8;
        if (i == count) //argument의 끝을 나타내는 공백 추가
			**(char ***)rsp = 0;
		else // 각각의 argument가 스택에 저장되어있는 주소 저장
			**(char ***)rsp = parse[i];
    }

    // return address push
    (*rsp) -= 8;
    **(void ***)rsp = 0; // void* 타입의 0 추가
}
```

### 3.3. process_wait()

```c
int
process_wait (tid_t child_tid UNUSED) {
	/* XXX: Hint) The pintos exit if process_wait (initd), we recommend you
	 * XXX:       to add infinite loop here before
	 * XXX:       implementing the process_wait. */
	while(1) {};
	return -1;
}
```

### 3.4. 테스트 결과

![image](https://github.com/onezerokang/onezerokang.github.io/assets/60874549/18937d77-fe81-48d6-8fa9-246309d7e4ef)

## 4. 회고

-   정 팀원: 개념의 틀을 어느정도 잡고 그 개념을 구체화 하는데 코드를 구현하는부분을 활용하려고 했으나, 이번주 개념을 너무 넓은 범위를 공부 하려다가 코드 구현에 시간배분을 많이 못한것 같다. 다음주에는 시간배분을 어떻게 할지 잘 생각해봐야겠다.

-   이 팀원: 프로젝트1 때와 다르게 시간이 많았음에도 불구하고, 스스로 다 구현하지 못해서 아쉬웠다. 그런데 더욱 문제는 완성된 코드들을 참고해서 보았음에도 불구하고 완벽히 이해하지 못하고 넘어와서 마음이 불편하다. 부족함을 많이 느꼈던 시간이었다.
-   강 팀원: gitbook을 읽거나 코드 구현을 하다가 모르는 것이 나오면 빠르게 해당 개념을 공부하고 다시 돌아왔어야 했는데, 한번에 너무 많은 것을 공부하려고 하다보니 시간 조절에 실패한 것이 아쉽다. 다음 과제는 이 부분을 주의하며 구현해야겠다.
