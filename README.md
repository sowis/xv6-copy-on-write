# xv6-copy-on-write
xv6 운영체제 메모리 copy on write 기능 구현

# 개발환경
Vmware/ubuntu-18.04.6-desktop-amd64

# 수정 및 작성한 소스코드
### mmu.h 
* line 4: PGSHIFT 상수 추가

### kalloc.c
* line 26~46: 페이지 참조 변수 배열 및 관리 함수 추가
* line 58: kinit1() 함수에 변수 초기화 과정 추가
* line 76: freerange() 함수에 페이지 참조 수 초기화 과정 추가
* line 84~108: kfree() 함수에 페이지 참조 카운트가 1이상이면 참조 카운트를  1 줄이고, 0이면 페이지를 free하는 과정 추가
* line 113~129: kalloc() 함수에 메모리 할당시 해당 페이지의 참조 카운트를 1로 설정하고, 총 사용가능한 페이지 개수를 한개 줄이는 과정 추가
* line 131~135: 현재 비어있는 페이지 수를 확인하는 함수 추가
* line 138~140: 시스템 콜 래퍼함수 정의

### vm.c
* line 315~345: copyuvm() 함수에서 페이지 테이블을 복사하지 않고, write기능 제거 후 참조 카운터 증가, 이후 참조로 전달하도록 변경
* line 388~414: pagefault() 함수 추가

### defs.h
* line 71~74: 함수 선언
* line 192: 함수 선언

### trap.c
* line 50~52: 페이지폴트시 pagefault() 함수를 호출하도록 트랩 연결

### syscall.h
* line 23: 시스템콜 번호 추가

### syscall.c
* line 106: 시스템 콜 선언
* line 130: 함수포인터 테이블 등록
### user.h
* line 26: 시스템 콜을 유저에게 보이도록 함
### usys.S
* line 32: 매크로 등록
### Makefile
* line 184: 유저 프로그램 등록
* line 254: 유저 프로그램 등록
### copy_on_write_test.c (신규)
* 테스트용 유저 프로그램 작성


# 발생한 문제 및 수정 방법
* 문제1. 제공된 kfree() 함수로 수정 시 컴파일 불가
* 해결1. pgrefcount[] getter/setter의 lock 문제로 보임. getter/setter대신pgrefcount[]에 직접 접근함으로서 해결 

<br>

* 문제 2. 제공된 kalloc() 함수가 메모리를 할당하지 못할 경우에도 num_free_pages변수 값이 줄어듦
* 해결 2. 메모리 할당이 가능한 경우에만 num_free_pages값이 줄어들도록 수정

<br>

* 문제 3. 제공된 copyuvm() 함수에 mem 변수가 존재하지 않음
* 해결 3. mem변수가 사용되는 mappages() 함수는 pte에 가상메모리주소와 물리메모리주소 매핑정보를 추가해주는 함수다. copy-on-write 매커니즘에서는 메모리 복사가 아닌, 참조만 전달하면 되므로 원본 물리 메모리 주소를 전달해주면 된다. 따라서 mem변수 대신 pa(기존 물리 메모리 주소)를 사용하였다.

<br>

* 문제 4. 제공된 소스코드 중 pagefault() 함수의 va변수가 재정의됨
* 해결 4. 정의를 한번만 하도록 변경

# 실행화면

<image src="readme_images/1.png">

Fork 이후 기존 메모리에 write하기 전에는 free page의 개수가 56665개지만, write 이후 free page의 개수가 56664개로 1개 줄어든 것을 볼 수 있다.

따라서 fork시에는 두 프로세스의 서로 다른 가상메모리가 같은 물리메모리를 참조하고 있지만, 한쪽에서 write를 하게 되면 그 순간 물리 메모리 페이지가 분리된다는 것을 확인할 수 있다.

Fork 전과 후의 free page의 차이(56733 -> 56665)는 프로세스 메타데이터로 인한 차이라고 생각된다.