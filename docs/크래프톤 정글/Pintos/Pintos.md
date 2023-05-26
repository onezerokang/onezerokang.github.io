# Pintos

> -   OS프로젝트는 PintOS의 코드를 직접 수정해가며 진행하는 프로젝트입니다.
> -   PintOS는 2004년 스탠포드에서 만들어진 교육용 운영체제예요. 우리 프로젝트는 이를 기반으로 KAIST 권영진 교수님 주도 하에 만들어진 `KAIST PintOS`로 진행됩니다.

핀토스 프로젝트의 목표는 운영체제를 만들어보며 운영체제의 이해도를 높이는 것이다.

-   [PROJECT 1: THREADS (05/26~06/01) - 1주](./Threads.md)
-   PROJECT 2: USER PROGRAMS (06/02~06/11) - 1.5주
-   PROJECT 3: VIRTUAL MEMORY (06/12~06/25) - 2주
-   PROJECT 4: FILE SYSTEM (06/26~07/02) - 1주

## 작업환경 세팅

-   AWS EC2 Ubuntu 18.04 (x86_64) 인스턴스 생성
-   EC2 접속 후 세팅

```shell
sudo apt update
sudo apt install -y gcc make qemu-system-x86 python3
```

-   pintos 레포지토리 복제

```shell
$ git clone --bare https://github.com/casys-kaist/pintos-kaist.git
$ cd pintos-kaist.git
$ git push --mirror https://github.com/${계정 ID}/pintos-kaist.git
$ cd ..
$ rm -rf pintos-kaist.git
$ git clone https://github.com/${계정 ID}/pintos-kaist.git
```

-   vscode의 Remote - SSH 익스텐션으로 ec2에 접속

    -   Remote - SSH 익스텐션을 설치한다.
    -   vscode 오른쪽 하단의 연결 버튼을 누르고 Connect to Host를 선택한다.
    -   설정해둔 Host가 없다면 Configure SSH Hosts → /Users/name/.ssh/config을 선택한다.
    -   config 파일에 Host를 추가한다.

    ```
    Host alias
        HostName hostname
        User user
    Host pintos
        HostName EC2의 IP 주소
        User ubuntu
        IdentityFile pem 파일 경로
    ```

    -   추가된 Host를 선택하여 ec2에 접속한다.

-   pintos가 제대로 동작하는지 확인(Project 1 기준)

```shell
$ cd pintos-kaist
$ source ./activate
$ cd threads
$ make check
# 뭔가 한참 compile하고 test 프로그램이 돈 후에 다음 message가 나오면 정상
20 of 27 tests failed.
```
