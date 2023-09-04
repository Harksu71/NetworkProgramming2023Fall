## 개요
Adapters에 윈도우 NIC Adapter의 linked-list를 읽어들여 print 한다.

---

### _WIN32_WINNT과 Header File define
---
윈도우 프로그래밍시 VC++에 _WIN32_WINNT macro 사용을 통해 적절한 header 파일 로딩 가이드
따라서 `#include` define 전에 _WIN32_WINNT define 필요함.

해당 코드 define은 윈도우 환경 개발 코드임을 정의하기때문에 `_WIN32_WINNT`` define을 통해 윈도우와 리눅스 듀얼 플랫폼 개발 가능

---

#### header 파일 
---

winsock 사용시 `winsock2.h` 사용
본 소스 파일은 network addaptor 접근하기 위해 `iphlpapi.h`를 define 함

`ws2tcpip.h`는 network adapters를 listing 하기 위해 사용. 

`stdio.h` 는 printf()

`stdlib.h`는 memory allocation
`locale.h`는 한글 출력 위해 필요

```c
#ifndef _WIN32_WINNT
#define _WIN32_WINNT 0x0600
#endif

#include <winsock2.h>
#include <iphlpapi.h>
#include <ws2tcpip.h> # getnameinfo 등 address 가져올때 사용
#include <stdio.h>
#include <stdlib.h>
#include "locale.h"   # 한글 출력 위해 필요
```

---

#### 라이브러리 파일
---

`#pragma comment(lib,"ws2_32.lib")` MS VCPP에 실행시 사용할 라이브러리 지정
명시적으로 지정하려면 complie command line에서 `-lws2_32` 정의 한다.

MinGW 환경에서 컴파일

```
gcc win_init.c -o win_init.exe -lws2_32
```

 network library (ws2_32.lib)

 network addaptor library(iphlpapi.lib)

```
#pragma comment(lib, "ws2_32.lib")
#pragma comment(lib, "iphlpapi.lib")
```
---

#### main() 과 WSAStartup(WORD wVersionRequested, LPWSADATA lpWSAData)
---

winsock 라이브러리 로딩및 초기화

```c
int WSAAPI WSAStartup(
  [in]  WORD      wVersionRequested,
  [out] LPWSADATA lpWSAData
);
```

`Winsock 2.2` 와 `WSADATA`를 이용해서 WSAStartup 호출

```c
int main() {

    WSADATA d;
    if (WSAStartup(MAKEWORD(2, 2), &d)) {
        printf("Failed to initialize.\n");
        return -1;
    }
```

---

#### adapters에 adapter linked-list 로딩
---

`adapters` 변수를 위해 메모리 할당하고, `GetAdapterAddresses()` 함수를 호출하여 adapters 값을 불러온다.

오류 발생시 return 하려면 앞에서 호출한 `WSAStartup()`을 cleaning 하기위해 `WSACleanup()` 호출 필요함.

- `asize` 변수에 adapters 를 위한 메모리 할당 사이즈를 잡는다. 여기서는 20KB 정도 잡아준다.
모든 memory 할당시 malloc, calloc등 return value가 NULL이면 메모리 할당 오류가 발생한 경우임. 예외처리가 추가되어야 함. 모든 자원 free 하고 return.

- `GetAdapterAddresses()` 함수의 첫 인자는  network family임  

|family      | 설명 |
| ----------- | ----------- |
|AF_UNSPEC  |V4, V6 둘다|
|AF_INET    |V4|
|AF_INET6   |V6|


- 두번째 인자는 flags 지시자임 `GAA_FLAG_INCLUDE_PREFIX` 를 통해 ip 주소 prefix 조회 가능
- 세번째 인자는 reserved로 0 입력
- 네번째 인자는 adapters 인자를 통해 adapter의 address를 가져온다.
- 마지막 인자는 adapters의 할당 크기인 asize 입력, 만약 크기가 충분하지 않으면 필요한 size를 call-by-reference로 값이 수정됨.

만약 adapters의 버퍼 크기가 작으면 `ERROR_BUFFER_OVERFLOW`의 오류 코드 리턴과 `asize`가 필요 크기로 수정됨

```c

    DWORD asize = 440;
    PIP_ADAPTER_ADDRESSES adapters;
    do {
        adapters = (PIP_ADAPTER_ADDRESSES)malloc(sizeof(IP_ADAPTER_ADDRESSES_LH));

        printf("IP_ADAPTER_ADDRESSES_LH(%d)", sizeof(IP_ADAPTER_ADDRESSES_LH));
        if (!adapters) {
            printf("Couldn't allocate %ld bytes for adapters.\n", asize);
            WSACleanup();
            return -1;
        }

        printf("adapters size(%d)\r\n", asize)
        int r = GetAdaptersAddresses(AF_UNSPEC, GAA_FLAG_INCLUDE_PREFIX, 0,
                adapters, &asize);
        if (r == ERROR_BUFFER_OVERFLOW) {
            printf("GetAdaptersAddresses wants %ld bytes.\n", asize);
            free(adapters);
            adapters = NULL;
        } else if (r == ERROR_SUCCESS) {
            break;
        } else {
            printf("Error from GetAdaptersAddresses: %d\n", r);
            free(adapters);
            adapters = NULL;
            WSACleanup();
            return -1;
        }
    } while (!adapters);
```

---

#### adapters linked list loop
---

각 node를 접근하기 위해 `adapter` 변수 정의하고 각 adapter node에 대해 처리함.
loop는 `adapter` 가 `NULL`이 되는 마지막 노드에서 종료
```c
PIP_ADAPTER_ADDRESSES adapter = adapters;
while (adapter) {
    ...
    adapter = adapter->Next;
}
```

`adapter->FriendlyName`는 adapter의 이름을 string으로 출력 가능
이때 `adapter` 이름이 한글인 경우가 있다.  `setlocale(LC_ALL, "korean");`이 한글출력이 가능하도록 함.


- 각 adapter에 설정된 address는 `adapter->FirstUnicastAddress`에 linked list로 저장되어 있음.

마찬가지로 각node에 대해 linked-list를 처리해야함. 마지막 node는 NULL 체크로 확인.

```c
PIP_ADAPTER_UNICAST_ADDRESS address = adapter->FirstUnicastAddress;
while (address) {
    ...
    address = address->Next;
}
```

- `address->Address.lpSockaddr->sa_family`변수에 주소의 타입(IPv4는 AF_INET, IPv6는 AF_INET6)
- `ap`버퍼에 `getnameinfo()`함수를 이용하여 ip 주소를 formatting 하여 수록한다.
- ap값을 printf()하여 주소 출력

```c
    PIP_ADAPTER_ADDRESSES adapter = adapters;
    while (adapter) {
        printf("\nAdapter name: %S\n", adapter->FriendlyName);

        PIP_ADAPTER_UNICAST_ADDRESS address = adapter->FirstUnicastAddress;
        while (address) {
            printf("\t%s",
                    address->Address.lpSockaddr->sa_family == AF_INET ?
                    "IPv4" : "IPv6");

            char ap[100];

            getnameinfo(address->Address.lpSockaddr,
                    address->Address.iSockaddrLength,
                    ap, sizeof(ap), 0, 0, NI_NUMERICHOST);
            printf("\t%s\n", ap);

            address = address->Next;
        }

        adapter = adapter->Next;
    }
```
---

#### Clean-up
---

프로그램 종료시에 이전에 할당한 메모리 free 하고 `WSACleanup()` 호출한다.

```c
    free(adapters);
    WSACleanup();
    return 0;
}
```

---

### 참고
---

https://docs.microsoft.com/ko-kr/windows/win32/api/iphlpapi/nf-iphlpapi-getadaptersaddresses?redirectedfrom=MSDN