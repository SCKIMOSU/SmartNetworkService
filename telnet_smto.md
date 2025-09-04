# telnet smtp

![image.png](image.png)

![image.png](image%201.png)

![image.png](image%202.png)

## **SMTP(전자메일 전송 프로토콜) 통신 과정**

- Gmail 메일 서버(`smtp.gmail.com`)와 클라이언트(`EHLO example.com`)가 서로 인사하고, 서버가 지원하는 기능을 알려주는 단계

---

### 1. 서버 인사 (Greeting)

```
220 smtp.gmail.com ESMTP ...

```

- Gmail 서버가 연결을 받아들이고, 자신이 **ESMTP**(확장 SMTP)를 지원한다고 알림

---

### 2. 클라이언트 자기소개

```
EHLO example.com

```

- 클라이언트(메일 보내려는 쪽)가 `EHLO` 명령어로 자신을 알리면서, 서버에게 "지원 가능한 기능을 알려 달라"고 요청함.

---

### 3. 서버가 지원 기능 알림

```
250-smtp.gmail.com at your service, [1.209.175.119]
250-SIZE 35882577
250-8BITMIME
250-STARTTLS
250-ENHANCEDSTATUSCODES
250-PIPELINING
250-CHUNKING
250 SMTPUTF8

```

각 항목 의미

- **SIZE 35882577** → 최대 메일 크기는 약 **35MB**까지 허용.
- **8BITMIME** → 일반 ASCII가 아닌 **8비트 문자**(예: 한글, 악센트 있는 문자 등)도 메일에 포함 가능.
- **STARTTLS** → 보안 강화를 위해 **TLS 암호화 연결**로 업그레이드 가능.
- **ENHANCEDSTATUSCODES** → 오류나 상태 코드를 더 자세하게 제공.
- **PIPELINING** → 여러 SMTP 명령을 **한 번에 묶어서 보내기 가능** (속도 향상).
- **CHUNKING** → 메일을 여러 청크(덩어리)로 나눠 전송 가능.
- **SMTPUTF8** → UTF-8 인코딩 지원 (국제 문자, 예: 한국어, 이모지 포함 주소 가능).

---

이 과정은 **메일 전송 준비 과정**으로, 클라이언트와 Gmail 서버가 "어떤 기능을 사용할 수 있는지" 합의하는 단계

## `EHLO example.com` 의미

---

### 1. **EHLO 명령**

- **SMTP**에서는 클라이언트가 서버에 처음 인사할 때 `HELO` 또는 `EHLO`를 사용.
- `EHLO`는 **Extended HELO**의 약자이고, **ESMTP(확장 SMTP)**를 사용하겠다는 의미.
- 즉, 단순한 메일 전송만 하는 것이 아니라, `STARTTLS`, `PIPELINING`, `SIZE` 같은 추가 기능들을 서버가 알려줄 수 있게 함.

---

### 2. **example.com 부분**

- `example.com`은 클라이언트가 **자신의 도메인 이름**을 서버에게 알리는 부분.
- 이 값은:
    - 실제 발송 서버의 도메인일 수도 있고,
    - 테스트 환경에서는 그냥 아무 문자열을 써도 서버가 응답을 해줌.
- 예를 들어 회사 메일 서버라면 보통 `mail.mycompany.com` 같은 실제 도메인을 사용.

---

### 3. 의미

`EHLO example.com` =

 “안녕하세요, 저는 `example.com` 도메인을 가진 클라이언트입니다. **확장 SMTP 기능**을 사용할 준비가 되어 있으니, 서버가 지원하는 기능들을 알려주세요.

---

![image.png](image%203.png)
