---
layout: post
title: "Reversing.kr Direct3D-FPS"
date: 2018-02-15 19:18:29 -0400
categories: reversing
---

### Direct3D-FPS
문제를 실행하면 인형을 죽이는 fps 게임이 실행된다. 게임 클리어 시, 출력되는 flag를 살펴보면 알아볼 수 없게 되어있다. 또한 이 flag를 디코딩하는 함수도 존재한다. (함수, 변수명은 임의로 변경)

~~~assembly
.text:00F23400 flag_decode     proc near          
.text:00F23400                 push    ecx
.text:00F23401                 call    sub_F23440
.text:00F23406                 cmp     eax, 0FFFFFFFFh
.text:00F23409                 jz      short loc_F2343E
.text:00F2340B                 mov     ecx, eax
.text:00F2340D                 imul    ecx, 210h
.text:00F23413                 mov     edx, dword_1339190[ecx]
.text:00F23419                 test    edx, edx
.text:00F2341B                 jg      short loc_F23435
.text:00F2341D                 mov     dword_1339194[ecx], 0
.text:00F23427                 mov     cl, byte_1339184[ecx]
.text:00F2342D                 xor     flag[eax], cl
.text:00F23433                 pop     ecx
.text:00F23434                 retn
~~~

우선 eax에는 인형의 번호가 저장되고, ecx = eax * 0x210 연산을 진행한다. 그리고 byte_0x1339184[ecx]와 flag[eax]랑 xor 연산을 진행해서 디코딩되어 알아볼 수 있는 flag가 출력된다. 그래서 모든 인형을 죽여보면 "Congratuation!, ~~~ ..."이 출력되지만 몇 가지 완벽하게 디코딩되지 않는다. 

IDA에서 직접 byte_ 1339184[ecx]를 찾아서 flag와 xor 연산을 하면 될 것이라고 생각해서 간단한 코드를 작성했다.
```python
flag = 0x1337028
xor_string = 0x1339184
result = []

for i in range(52) # flag 글자 수가 총 52글자
	result.append(chr(Byte(flag + i) ^ Byte(xor_offset + (i * 0x210))))
	print ''.join(result)
```

이렇게 하면 문제를 해결할 수 있다.

> IDA에서 WinDBG가 개꿀이다. 그리고 Byte("주소")를 하는 것도 이번 기회에 처음 알았다.