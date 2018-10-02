---
layout: post
title: "CODEGATE 2018 Redvelvet Writeup"
date: 2018-02-13 09:00:00 -0400
categories: ctf
---

### Redvelvet
```
Downloads/RedVelvet: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked
```

64비트 리눅스 바이너리다. 프로그램을 구경해보면 알맞은 flag를 입력하면 되는 문제인 것 같다.
IDA를 통해 디컴파일을 해보면 함수가 참 많다.

```c
printf((_BYTE *)&loc_4016CF + 1, argv, envp);
fgets(&s, 27, edata);
func1((unsigned int)s, (unsigned int)v13);
func2((unsigned int)v13, (unsigned int)v14);
func3((unsigned int)v14, (unsigned int)v15);
func4((unsigned int)v15, (unsigned int)v16);
func5((unsigned int)v16, (unsigned int)v17);
func6((unsigned int)v17, (unsigned int)v18, (unsigned int)v19);
func7((unsigned int)v19, (unsigned int)v20, (unsigned int)v21);
func8((unsigned int)v21, (unsigned int)v22, (unsigned int)v23);
func9((unsigned int)v23, (unsigned int)v24, (unsigned int)v25);
v4 = ptrace(0, 0LL, 1LL, 0LL);
if ( v4 == -1 )
{
  v8 = *(_BYTE *)v5;
  v9 = v3 + *(_BYTE *)(v5 + 111) == 0;
  *(_BYTE *)(v5 + 111) += v3;
  JUMPOUT(!v9, &unk_401746);
  v6C &= BYTE1(v4);
  JUMPOUT(*(_QWORD *)"ag : ");
}
func10((unsigned int)v25, (unsigned int)v26, (unsigned int)v27);
func11((unsigned int)v27, (unsigned int)v28, (unsigned int)v29);
func12((unsigned int)v29, (unsigned int)v30, (unsigned int)v31);
func13((unsigned int)v31, (unsigned int)v32, (unsigned int)v33);
func14((unsigned int)v33, (unsigned int)v34, (unsigned int)v35);
func15((unsigned int)v35, (unsigned int)v36, (unsigned int)v37);
SHA256_Init(&v11);
v6 = strlen(&s);
SHA256_Update(&v11, &s, v6);
SHA256_Final(v38, &v11);
for ( i = 0; i <= 31; ++i )
  sprintf(&s1[2 * i], "%02x", (unsigned __int8)v38[i]);
  if ( strcmp(s1, s2) )
    exit(1);
  printf("flag : {\" %s \"}\n", &s);
  return 0;
```
사실 하나씩 분석하면서 문제를 풀어도 됬지만 이걸 풀 당시에 너무 귀찮아서 angr 모듈을 이용해서 풀었다.
angr를 통해 가고싶은 주소는 func1,2,3 ... 15를 지나 최종적으로 flag가 출력되는 0x401610이다.
이번에 angr를 처음써보는데 구글에 찾아보니 avoid 주소도 인자로 입력하길래 exit() 함수가 있는 0x401621를 입력했다.
최종적인 페이로드 코드는 다음과 같다.

* payload.py

```python
import angr

def main():
        p = angr.Project("RedVelvet", load_options={'auto_load_libs': False})
        ex = p.surveyors.Explorer(find=(0x401610, ), avoid=(0x401621, 0x40162B))
        ex.run()

        return ex.found[0].state.posix.dumps(0)

if __name__ == '__main__':
        print main()
```

그럼 조금 깨지긴 하지만 알아볼 수 있을 정도의 flag를 찾을 수 있다~