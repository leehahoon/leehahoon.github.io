---
layout: post
title: "Reversing.kr AutoHotKey1"
date: 2018-02-21 04:12:31 -0400
categories: reversing
---

### AutoHotkey1
문제와 함께 주어지는 readme.txt를 읽어보면 DecryptKey와 EXE's Key를 찾아야 한다고 나와있다. 그 값들은 md5로 해시화 되어 있다. 

프로그램을 실행하면 무언가를 입력하는 창이 나온다. 하지만 upx로 패킹되어 있어 언패킹을 한 후, 프로그램을 실행하면 EXE Corrupted라는 문구가 출력된다. (이후, 직접 손으로 OEP를 찾을 생각을 못하고 upx로 언패킹한 바이너리로 삽질했다.)

EXE Corrupted라는 문구가 위치한 주소에 BP를 걸고 계속 실행시켜보면 레지스터에 Key가 2개 모두 출력된다~