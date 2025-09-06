# WSAStartup()

## WSAStartup

- `WSAStartup`은 Windows 소켓 API(WinSock)를 초기화하기 위해 사용하는 함수
- C/C++에서 네트워크 프로그래밍을 할 때 반드시 처음에 호출해야 하는 API

---

### 1. 함수 정의

```c
int WSAStartup(
  WORD wVersionRequested,
  LPWSADATA lpWSAData
);

```

- **wVersionRequested**
    
    사용할 Winsock의 버전을 지정 (예: `MAKEWORD(2, 2)` → Winsock 2.2 요청)
    
- **lpWSAData**
    
    `WSADATA` 구조체 포인터. 실제 사용 가능한 소켓 버전, 시스템 정보 등이 채워짐.
    

---

### 2. 사용

1. 프로그램 시작 시 `WSAStartup()` 호출
    
    ```c
    WSADATA wsaData;
    int result = WSAStartup(MAKEWORD(2, 2), &wsaData);
    if (result != 0) {
        printf("WSAStartup 실패: %d\n", result);
        return 1;
    }
    
    ```
    
2. 소켓 API 사용 (`socket()`, `bind()`, `connect()` 등)
3. 사용이 끝나면 `WSACleanup()` 호출해 자원 해제

---

### 3. 코드

```c
#include <winsock2.h>
#include <stdio.h>

int main() {
    WSADATA wsaData;
    int iResult;

    // WinSock 2.2 초기화
    iResult = WSAStartup(MAKEWORD(2, 2), &wsaData);
    if (iResult != 0) {
        printf("WSAStartup 실패: %d\n", iResult);
        return 1;
    }

    printf("WinSock 초기화 성공!\n");

    // 소켓 프로그래밍 코드 작성...

    // 자원 해제
    WSACleanup();
    return 0;
}

```

---

### 4. 정리

- `WSAStartup` → **윈도우에서 소켓 라이브러리를 초기화하는 함수**
- 호출하지 않으면 소켓 관련 함수(`socket`, `send`, `recv`) 사용 불가
- 보통 `WSACleanup()`과 쌍으로 사용

---

## `WSAStartup` "A" 의미

- `WSAStartup`은 WinSock API(Windows Sockets API)의 약자
    - **WS** → Windows Sockets
    - **A** → **API (Application Programming Interface)**
    - **Startup** → 초기화
    - `WSAStartup`은 “Windows Sockets API Startup”이라는 뜻으로, **윈도우 소켓 API를 초기화한다**

---

- 유사한 함수 이름 :
    - `WSACleanup()` → Windows Sockets API Cleanup
    - `WSAGetLastError()` → Windows Sockets API Get Last Error
- `WSA...`로 시작하는 함수들은 모두 **Windows Sockets API**에 속함

---

## **`WSADATA` 구조체**

- `WSAStartup()` 호출 시 두 번째 인자
- **윈속(WinSock) 라이브러리 초기화 정보와 환경 세부사항**
- `WSADATA` 구조체는 **Winsock 버전, 지원 정보, 설명/상태 문자열**을 담는 초기화 결과 컨테이너

---

- `WSADATA` 구조체 정의 (C/C++ winsock2.h 기준)

```c
typedef struct WSAData {
  WORD           wVersion;       // 요청한 Winsock 버전
  WORD           wHighVersion;   // 지원 가능한 가장 높은 Winsock 버전
  unsigned short iMaxSockets;    // (옛날에 사용) 동시에 지원 가능한 소켓 수
  unsigned short iMaxUdpDg;      // (옛날에 사용) 최대 UDP 데이터그램 크기
  char FAR       *lpVendorInfo;  // 벤더(제조사) 정보 문자열
  char           szDescription[WSADESCRIPTION_LEN+1]; // 구현 설명
  char           szSystemStatus[WSASYS_STATUS_LEN+1]; // 상태 문자열
} WSADATA, *LPWSADATA;

```

---

- 주요 멤버
1. **wVersion**
    - `MAKEWORD(2,2)` 같은 요청 버전 값이 저장됨
    - 예: Winsock 2.2 요청 시 `0x0202`
2. **wHighVersion**
    - 해당 시스템에서 지원 가능한 **최대 버전**
    - 예: 2.2까지 지원 가능하면 `0x0202`
3. **iMaxSockets**
    - 동시에 생성 가능한 최대 소켓 개수
    - **Winsock 2.0 이상에서는 의미 없음** (호환성 때문에 남아 있음)
4. **iMaxUdpDg**
    - 지원 가능한 UDP 데이터그램 최대 크기
    - 역시 **현재는 거의 사용 안 됨**
5. **lpVendorInfo**
    - 네트워크 공급자(Vendor)가 제공하는 부가 정보 포인터 (거의 null)
6. **szDescription**
    - Winsock 구현에 대한 간단한 설명 문자열
    - 예: `"WinSock 2.0"` 같은 텍스트
7. **szSystemStatus**
    - 현재 소켓 시스템 상태 문자열
    - 예: `"Running"`

---

- 내용 출력

```c
WSADATA wsa;
WSAStartup(MAKEWORD(2,2), &wsa);

printf("Winsock Version: %d.%d\n", LOBYTE(wsa.wVersion), HIBYTE(wsa.wVersion));
printf("Description: %s\n", wsa.szDescription);
printf("System Status: %s\n", wsa.szSystemStatus);

```

- 출력

```
Winsock Version: 2.2
Description: WinSock 2.0
System Status: Running

```

---
