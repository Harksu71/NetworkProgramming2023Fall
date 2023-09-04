
## 개요
addresses에 interface의 linked-list를 읽어들여 print 한다.

---

#### include 파일 정의
---

```c
#include <sys/socket.h>
#include <netdb.h>
#include <ifaddrs.h> # ip address 출력에 사용
#include <stdio.h>
#include <stdlib.h>
```
---

### 참고
---

https://man7.org/linux/man-pages/man3/getifaddrs.3.html

```c
#include <sys/types.h>
#include <ifaddrs.h>

int getifaddrs(struct ifaddrs **ifap);
void freeifaddrs(struct ifaddrs *ifa);
```

---

#### struct ifaddrs 정의
---

address->ifa_next 의 linked-list 정의

```c
struct ifaddrs {
    struct ifaddrs  *ifa_next;    /* Next item in list */
    char            *ifa_name;    /* NULL terminated Name of interface */
    unsigned int     ifa_flags;   /* Flags from SIOCGIFFLAGS */
    struct sockaddr *ifa_addr;    /* Address of interface */
    struct sockaddr *ifa_netmask; /* Netmask of interface */
    union {
        struct sockaddr *ifu_broadaddr;
                        /* Broadcast address of interface */
        struct sockaddr *ifu_dstaddr;
                        /* Point-to-point destination address */
    } ifa_ifu;
#define              ifa_broadaddr ifa_ifu.ifu_broadaddr
#define              ifa_dstaddr   ifa_ifu.ifu_dstaddr
    void            *ifa_data;    /* Address-specific data */
};
```

```
       SIOCGIFFLAGS, SIOCSIFFLAGS
              Get or set the active flag word of the device.  ifr_flags
              contains a bit mask of the following values:

                                      Device flags
              IFF_UP            Interface is running.
              IFF_BROADCAST     Valid broadcast address set.
              IFF_DEBUG         Internal debugging flag.
              IFF_LOOPBACK      Interface is a loopback interface.
              IFF_POINTOPOINT   Interface is a point-to-point link.
              IFF_RUNNING       Resources allocated.
              IFF_NOARP         No arp protocol, L2 destination address not
                                set.
              IFF_PROMISC       Interface is in promiscuous mode.
              IFF_NOTRAILERS    Avoid use of trailers.
              IFF_ALLMULTI      Receive all multicast packets.
              IFF_MASTER        Master of a load balancing bundle.
              IFF_SLAVE         Slave of a load balancing bundle.
              IFF_MULTICAST     Supports multicast
              IFF_PORTSEL       Is able to select media type via ifmap.
              IFF_AUTOMEDIA     Auto media selection active.
              IFF_DYNAMIC       The addresses are lost when the interface
                                goes down.
              IFF_LOWER_UP      Driver signals L1 up (since Linux 2.6.17)
              IFF_DORMANT       Driver signals dormant (since Linux 2.6.17)
              IFF_ECHO          Echo sent packets (since Linux 2.6.25)
```

- `addresses`변수를 선언하고, `getifaddrs()`함수를 호출하면 함수가 메모리를 할당하고 linkedlist 형태로 interface리스트를 채워 리턴한다. 만약 오류 발생시 -1 리턴하고 정상은 0 리턴
- `address`의 접근은 `addresses`에 대한 linked list를 접근하면 된다.
- linked list 방문을 통해 노드는 address 변수에 받아서 접근한다. 

```c
struct ifaddrs *address = addresses;
while(address) {
    ...
    address = address->ifa_next;
}
  ```  

- 다음 노드 접근은 `address = address->ifa_next`를 통해서 가능
- `address == 0`이면 linked list 끝임.

```c
int main() {

    struct ifaddrs *addresses;

    if (getifaddrs(&addresses) == -1) {
        printf("getifaddrs call failed\n");
        return -1;
    }

    struct ifaddrs *address = addresses;
    while(address) {
        if (address->ifa_addr == NULL) { 
            address = address->ifa_next;
            continue;
        }
        int family = address->ifa_addr->sa_family;
        if (family == AF_INET || family == AF_INET6) {

            printf("%s\t", address->ifa_name);
            printf("%s\t", family == AF_INET ? "IPv4" : "IPv6");

            char ap[100];
            const int family_size = family == AF_INET ?
                sizeof(struct sockaddr_in) : sizeof(struct sockaddr_in6);
            getnameinfo(address->ifa_addr,
                    family_size, ap, sizeof(ap), 0, 0, NI_NUMERICHOST);
            printf("\t%s\n", ap);

        }
        address = address->ifa_next;
    }
```
---

### interface의 address 출력
---

각 address에 대해 
- address family(IPv4, IPv6)를 식별한다. AF_INET는 IPv4, AF_INET6는 IPv6
  이외의 다른 type일 수 있는데 이는 skip
 참고 : https://codebrowser.dev/glibc/glibc/sysdeps/unix/sysv/linux/bits/socket.h.html#97

- `address->ifa_name`로 adapter 이름을 출력하고, 
- `address->ifa_addr->sa_family`로 타입을 출력하고,  
- `getnameinfo()` 와 `address->ifa_addr`를 이용하여 address를 출력한다.
  이때 출력 버퍼는 stack에 `char ap[100]`으로 잡아 함수가 return할때 자동 해제 됨.

```c
// for each address
int family = address->ifa_addr->sa_family;
if (family == AF_INET || family == AF_INET6) {

    printf("%s\t", address->ifa_name);
    printf("%s\t", family == AF_INET ? "IPv4" : "IPv6");

    char ap[100];
    const int family_size = family == AF_INET ?
        sizeof(struct sockaddr_in) : sizeof(struct sockaddr_in6);
    getnameinfo(address->ifa_addr,
            family_size, ap, sizeof(ap), 0, 0, NI_NUMERICHOST);
    printf("\t%s\n", ap);

}
```

---

### addresses 메모리 free
---

`addresses`는 `getifaddrs()` 함수에서 메모리를 할당받았기 때문에 사용후 free 해줘야 함. 따라서 `freeifaddrs()`로 메모리를 해제 한다.

```c
int main() {

    struct ifaddrs *addresses;

    if (getifaddrs(&addresses) == -1) {
        printf("getifaddrs call failed\n");
        return -1;
    }

    ... 

    freeifaddrs(addresses);
    return 0;
}
```