---
layout: post
title:  "DLL Injection"
date:   2018-05-07 18:33:48 -0400
categories: reversing
---

### DLL?
[DLL(Dynamic Link Library)](https://support.microsoft.com/ko-kr/help/815065/what-is-a-dll)는 마이크로소프트 윈도우에서 구현된 동적 링크 라이브러리다. 내부에는 다른 프로그램이 불러서 쓸 수 있는 코드와 데이터, 다양한 함수들을 포함하고 있다. DLL을 사용하면 효율적으로 코드를 모듈화하고 재사용할 수 있으며, 메모리 사용 효율성을 높이고 사용되는 디스크 공간을 줄일 수 있다. 따라서 운영 체제와 프로그램이 더 빠르게 로드 및 실행된다.
DLL의 이점은 다음과 같다.
* 더 적은 리소스 사용
* 모듈식 아키텍쳐 활용
* 손쉬운 배포와 설치

### DLL Injection?
다른 프로세스에 특정 DLL 파일을 강제로 삽입하는 행위를 의미한다. 이를 통해 다른 프로세스의 주소 공간 내에서 DLL을 강제로 로드시킴으로써 코드를 실행시킬 수 있다. 일반적인 DLL Loading과는 다르게 대상이 자신의 프로세스가 아닌 다른 프로세스에 로딩한다. 

### 구현 방법
#### CreateRemoteThread

코드의 실행 흐름을 정리하면 다음과 같다.
1. dwPID를 통해 대상 프로세스의 핸들을 구함, OpenProcess()
2. 대상 프로세스 메모리에 szDLLName 크기만큼 메모리 할당, VirtualAllocEx()
3. 할당 받은 메모리에 DLL 경로 씀, WriteProcessMemory()
4. LoadLibraryA() API 주소 구함, GetModuleHandle(), GetProcessAddress()
5. 대상 프로세스에 스레드 실행, CreateRemoteThread()


~~~c
#include "stdafx.h"
#include "stdio.h"
#include "windows.h"
#include "tlhelp32.h"

#define DEF_PROC_NAME (PROGRAM_NAME)
#define DEF_DLL_PATH  (DLL_PATH)

DWORD FindProcessID(LPCTSTR szProcessName);
BOOL InjectDll(DWORD dwPID, LPCTSTR szDllName);

int main(int argc, char* argv[])
{
  DWORD dwPID = 0xFFFFFFFF;

  // find process
  dwPID = FindProcessID(DEF_PROC_NAME);
  if (dwPID == 0xFFFFFFFF)
  {
    printf("There is no <%s> process!\n", DEF_PROC_NAME);
    return 1;
  }

  // inject dll
  InjectDll(dwPID, DEF_DLL_PATH);

  return 0;
}

DWORD FindProcessID(LPCTSTR szProcessName)
{
  DWORD dwPID = 0xFFFFFFFF;
  HANDLE hSnapShot = INVALID_HANDLE_VALUE;
  PROCESSENTRY32 pe;

  // Get the snapshot of the system
  pe.dwSize = sizeof(PROCESSENTRY32);
  hSnapShot = CreateToolhelp32Snapshot(TH32CS_SNAPALL, NULL); 

  Process32First(hSnapShot, &pe);
  do
  {
    if (!_stricmp(szProcessName, pe.szExeFile))
    {
      dwPID = pe.th32ProcessID;
      break;
    }
  } while (Process32Next(hSnapShot, &pe));

  CloseHandle(hSnapShot);

  return dwPID;
}

BOOL InjectDll(DWORD dwPID, LPCTSTR szDllName)
{
  HANDLE hProcess, hThread;
  LPVOID pRemoteBuf;
  DWORD dwBufSize = lstrlen(szDllName) + 1;
  LPTHREAD_START_ROUTINE pThreadProc;

  if (!(hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, dwPID)))
    return FALSE;

  pRemoteBuf = VirtualAllocEx(hProcess, NULL, dwBufSize, MEM_COMMIT, PAGE_READWRITE);

  WriteProcessMemory(hProcess, pRemoteBuf, (LPVOID)szDllName, dwBufSize, NULL);

  pThreadProc = (LPTHREAD_START_ROUTINE)GetProcAddress(GetModuleHandle("kernel32.dll"), "LoadLibraryA");
  hThread = CreateRemoteThread(hProcess, NULL, 0, pThreadProc, pRemoteBuf, 0, NULL);
  WaitForSingleObject(hThread, INFINITE);

  CloseHandle(hThread);
  CloseHandle(hProcess);

  return TRUE;
}

~~~

> 코드출처: 리버스코어 (ReverseCore)

FindProcessID()에서 [CreateToolhelp32Snapshot()](https://msdn.microsoft.com/ko-kr/library/windows/desktop/ms682489(v=vs.85).aspx)를 이용해 찾고자하는 프로세스의 PID를 구한다. InjectDLL()에선 구한 PID를 통해 해당 프로세스의 핸들을 구한다. 이후, 대상 프로세스에 삽입할 DLL 경로를 쓰기 위해 메모리 공간을 할당하고 그 공간에 쓴다. 그리고 DLL을 로드하기 위해 [LoadLibraryA()](https://msdn.microsoft.com/en-us/library/windows/desktop/ms684175(v=vs.85).aspx)의 주소를 구하고 [CreateRemoteThread()](https://msdn.microsoft.com/ko-kr/library/windows/desktop/ms682437(v=vs.85).aspx)를 이용해 이를 실행한다. 이를 통해 DLL를 원하는 프로세스에 삽입할 수 있다.

실습에 사용된 DLL의 코드는 다음과 같다.
~~~c
#include "stdio.h"
#include "windows.h"

BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved)
{
  HANDLE hThread = NULL;

  switch (fdwReason)
  {
  case DLL_PROCESS_ATTACH:
    CloseHandle(hThread);
    MessageBox(NULL, "This is DLL Injection Test!", "LEEHAHOON", MB_OK);
    break;
  }

  return TRUE;
}
~~~

#### Registry AppInit_DLLs
윈도우 운영체제에는 레지스트리라고 불리는 데이터베이스가 존재한다. 이 DB에는 하드웨어, 소프트웨어에 관한 설정, 선택항목뿐만 아니라 사용자 PC에 대한 정보도 들어있다. 이를 사용하기 전에는 구성 설정을 담는데에 INI 파일을 이용했지만 찾기 쉽지 않아서 레지스트리를 도입했다.
여러 레지스트리 중 DLL Injection에 사용될 키는 AppInit_DLLs 이다. 위치는 "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Windows\AppInit_DLLs"에 존재한다. 

AppInit_DLLs를 이용한 DLL Injection은 실행되는 모든 프로세스에 레지스트리에 적힌 DLL을 로드하므로 조심해서 사용해야 한다.

#### DLL Injection with User APC
일반적으로 DLL Injection을 수행할때에는 위에서 이야기한 CreateRemoteThread()를 이용한 방법이 보편적이다. 이 방법은 간단하면서도 신뢰할 수 있지만 몇 가지 단점이 존재한다. 대상 프로세스를 OpenProcess로 핸들을 구할 때, 광범위한 Access 권한을 요구한다. 또한 많은 안티 디버깅 도구들은 CreateRemoteThread()를 관찰하기 때문에 쉽게 탐지 당할 수 있다.
DLL Injection을 수행하는 또 다른 방법은 바로 APC를 이용하는 것이다. APC는 Asynchronous Procedure Call의 약자로 해석하면 비동기 프로시저 호출이란 의미이다. [QueueUserAPC()](https://msdn.microsoft.com/ko-kr/library/windows/desktop/ms684954(v=vs.85).aspx)를 사용하여 대상 프로세스에서 실행중인 모든 스레드를 연결하여 DLL을 로드하도록 수행하면 우리가 원하는 DLL Injection을 할 수 있다. 이를 통해 안티 디버깅 도구들을 우회하고 적은 권한만으로도 DLL Injection을 할 수 있다. 하지만 QueueUserAPC()를 호출했다고 해서 곧바로 사용자가 지정한 thread가 수행되는 것이 아닌, alertable 상태가 될 때 호출된다. 이는  SleepEx(), WaitForSingleObjectEx(), WaitForMultipleObjectsEx(), MsgWaitForMultipleObjectsEx()와 같은 함수를 사용하면 alertable 상태가 되어 스레드를 사용자가 조정할 수 있게 된다.

~~~c
#include "stdafx.h"
#include "stdio.h"
#include "windows.h"
#include "tlhelp32.h"
#include <vector>

#define DEF_PROC_NAME (PROGRAM_NAME)
#define DEF_DLL_PATH  (DLL_PATH)

using std::vector;

DWORD FindProcessID(LPCTSTR szProcessName, vector<DWORD> &tids);
BOOL InjectDll(DWORD dwPID, LPCTSTR szDllName, vector<DWORD> &tids);

int main(int argc, char* argv[])
{
  DWORD dwPID = 0xFFFFFFFF;
  vector<DWORD> tids;

  // find process
  dwPID = FindProcessID(DEF_PROC_NAME, tids);
  if (dwPID == 0xFFFFFFFF)
  {
    printf("There is no <%s> process!\n", DEF_PROC_NAME);
    return 1;
  }

  // inject dll
  InjectDll(dwPID, DEF_DLL_PATH, tids);

  return 0;
}

DWORD FindProcessID(LPCTSTR szProcessName, vector<DWORD> &tids)
{
  DWORD dwPID = 0xFFFFFFFF;
  HANDLE hSnapShot = INVALID_HANDLE_VALUE;
  PROCESSENTRY32 pe;
  THREADENTRY32 te;

  // Get the snapshot of the system
  pe.dwSize = sizeof(PROCESSENTRY32); // 296d (128h)
  te.dwSize = sizeof(THREADENTRY32);
  hSnapShot = CreateToolhelp32Snapshot(TH32CS_SNAPALL, NULL); // ProcessID = 0

                                // find process
  Process32First(hSnapShot, &pe);
  do
  {
    if (!_stricmp(szProcessName, pe.szExeFile))
    {
      dwPID = pe.th32ProcessID;
      if (Thread32First(hSnapShot, &te)) {
        do {
          if (te.th32OwnerProcessID == dwPID) {
            tids.push_back(te.th32ThreadID);
            printf("%d\n", te.th32ThreadID);
          }
        } while (Thread32Next(hSnapShot, &te));
      }
      break;
    }
  } while (Process32Next(hSnapShot, &pe));

  CloseHandle(hSnapShot);

  printf("%d\n", dwPID);

  return dwPID;
}

BOOL InjectDll(DWORD dwPID, LPCTSTR szDllName, vector<DWORD> &tids)
{
  HANDLE hProcess, hThread;
  LPVOID pRemoteBuf;
  DWORD dwBufSize = lstrlen(szDllName) + 1;
  LPTHREAD_START_ROUTINE pThreadProc;

  if (!(hProcess = OpenProcess(PROCESS_VM_WRITE | PROCESS_VM_OPERATION, FALSE, dwPID)))
    return FALSE;

  pRemoteBuf = VirtualAllocEx(hProcess, NULL, dwBufSize, MEM_COMMIT, PAGE_READWRITE);

  WriteProcessMemory(hProcess, pRemoteBuf, (LPVOID)szDllName, dwBufSize, NULL);

  pThreadProc = (LPTHREAD_START_ROUTINE)GetProcAddress(GetModuleHandle("kernel32.dll"), "LoadLibraryA");
  //hThread = CreateRemoteThread(hProcess, NULL, 0, pThreadProc, pRemoteBuf, 0, NULL);
  for (int i = 0; i < tids.size(); i++) {
    hThread = OpenThread(THREAD_SET_CONTEXT, FALSE, tids[i]);
    if (hThread) {
      QueueUserAPC((PAPCFUNC)pThreadProc, hThread, (ULONG_PTR)pRemoteBuf);
    }
  }

  CloseHandle(hProcess);

  return TRUE;
}


~~~

코드는 CreateRemoteThread()를 이용한 방법과 비슷하지만, 대상 프로세스의 스레드를 구하고, 이를 QueueUserAPC()를 통해 호출한다는 점이 다르다.

> [참고 URL](http://blogs.microsoft.co.il/pavely/2017/03/14/injecting-a-dll-without-a-remote-thread/)

