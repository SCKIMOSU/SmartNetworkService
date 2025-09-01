# Chapter 01: 네트워크와 소켓 프로그래밍

- **학습목표:** TCP/IP 동작 원리, 소켓 개념·특징, 간단한 소켓 프로그램 작성 흐름.
- **TCP/IP 4계층:** 네트워크 접근(물리/드라이버) → 인터넷(IP·라우팅) → 전송(TCP/UDP, 포트) → 응용(HTTP/FTP/SMTP 등).
- **주소 체계:**
    - IP: IPv4(32비트, 예: 147.46.114.70), IPv6(128비트, 예: 2001:0230:…:1111).
    - 포트: 0–1023(Well-known), 1024–49151(Registered), 49152–65535(Dynamic/Private).
- **TCP vs UDP:**
    - TCP: 연결형, 신뢰성/재전송, 바이트 스트림(경계 없음), 유니캐스트.
    - UDP: 비연결, 비신뢰(재전송 없음), 데이터그램(경계 있음), 유니/브로드/멀티캐스트.

# **파이썬 소켓 클라이언트/서버 예제 코드**

---

## 1) 간단한 에코 서버 (server.py)

```python
import socket

HOST = '127.0.0.1'   # localhost (IPv4)
PORT = 9000          # 사용할 포트 번호

# TCP 소켓 생성
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as server_socket:
    server_socket.bind((HOST, PORT))   # IP와 포트 바인딩
    server_socket.listen()             # 클라이언트 접속 대기
    print(f"Server listening on {HOST}:{PORT}...")

    conn, addr = server_socket.accept()  # 연결 수락
    with conn:
        print(f"Connected by {addr}")
        while True:
            data = conn.recv(1024)       # 클라이언트로부터 데이터 수신
            if not data:
                break
            print(f"Received: {data.decode()}")
            conn.sendall(data)           # 받은 데이터를 그대로 클라이언트에게 전송

```

---

## 2) 간단한 에코 클라이언트 (client.py)

```python
import socket

HOST = '127.0.0.1'   # 접속할 서버 IP
PORT = 9000          # 서버가 열어둔 포트

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as client_socket:
    client_socket.connect((HOST, PORT))   # 서버에 연결
    while True:
        msg = input("메시지 입력 (quit 입력시 종료): ")
        if msg.lower() == "quit":
            break
        client_socket.sendall(msg.encode())    # 서버로 데이터 전송
        data = client_socket.recv(1024)        # 서버로부터 응답 수신
        print(f"서버 응답: {data.decode()}")

```

---

## 실행 방법

1. 터미널 2개 열기.
2. 첫 번째 터미널에서 서버 실행:
    
    ```bash
    python server.py
    
    ```
    
3. 두 번째 터미널에서 클라이언트 실행:
    
    ```bash
    python client.py
    
    ```
    
4. 클라이언트 쪽에서 입력한 메시지가 서버를 거쳐 다시 돌아오는 것을 확인.

---

# **파이썬  AI 서버**

클라이언트가 텍스트와 작업 종류(감정분석/요약)를 보내면, 서버가 **Hugging Face Transformers** 모델로 처리해 **JSON 응답**을 돌려줌. (멀티클라이언트 지원)

---

# 1) 서버: AI 소켓 서버 (`ai_server.py`)

```python
import socket
import threading
import json
from typing import Dict, Any

# transformers는 처음 실행 시 모델을 다운로드합니다.
# 사내/오프라인 환경이면 사전 캐시 필요.
from transformers import pipeline

HOST = "127.0.0.1"
PORT = 9000
BACKLOG = 5
BUFF = 4096
ENC = "utf-8"

# ---- AI 파이프라인 초기화 (필요 시 모델 변경 가능) ----
sentiment = pipeline("sentiment-analysis")  # distilbert-base-uncased-finetuned-sst-2-english
summarizer = pipeline("summarization")      # t5-small/sshleifer-distilbart-cnn-12-6 등 환경에 따라

def handle_request(payload: Dict[str, Any]) -> Dict[str, Any]:
    task = payload.get("task")
    text = payload.get("text", "")

    if not text or not isinstance(text, str):
        return {"ok": False, "error": "text must be a non-empty string"}

    if task == "sentiment":
        result = sentiment(text)[0]   # {'label': 'POSITIVE', 'score': 0.999...}
        return {"ok": True, "task": "sentiment", "label": result["label"], "score": result["score"]}

    elif task == "summary":
        # 매우 긴 텍스트는 간단히 길이 제한. 필요시 슬라이딩 윈도우로 분할 요약 가능.
        max_len = 1500
        input_text = text[:max_len]
        out = summarizer(input_text, max_length=120, min_length=25, do_sample=False)[0]["summary_text"]
        return {"ok": True, "task": "summary", "summary": out}

    else:
        return {"ok": False, "error": f"unknown task: {task}. Use 'sentiment' or 'summary'."}

def client_thread(conn: socket.socket, addr):
    with conn:
        try:
            # 간단한 프로토콜: "한 줄에 하나의 JSON" (newline-delimited JSON)
            buffer = ""
            conn.sendall(b'{"hello":"ai-server","protocol":"ndjson","tasks":["sentiment","summary"]}\n')
            while True:
                data = conn.recv(BUFF)
                if not data:
                    break
                buffer += data.decode(ENC)

                # 줄 단위로 파싱
                while "\n" in buffer:
                    line, buffer = buffer.split("\n", 1)
                    if not line.strip():
                        continue
                    try:
                        payload = json.loads(line)
                        resp = handle_request(payload)
                    except json.JSONDecodeError:
                        resp = {"ok": False, "error": "invalid JSON"}

                    msg = (json.dumps(resp, ensure_ascii=False) + "\n").encode(ENC)
                    conn.sendall(msg)
        except ConnectionResetError:
            pass
        finally:
            print(f"Disconnected: {addr}")

def main():
    print(f"Loading AI pipelines... (first run may download models)")
    # 파이프라인이 위에서 이미 준비되며, 여기서 로딩 메시지를 알림
    print("Starting socket server...")

    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        s.bind((HOST, PORT))
        s.listen(BACKLOG)
        print(f"AI Server listening on {HOST}:{PORT}")

        while True:
            conn, addr = s.accept()
            print(f"Connected: {addr}")
            threading.Thread(target=client_thread, args=(conn, addr), daemon=True).start()

if __name__ == "__main__":
    main()

```

---

# 2) 클라이언트: 테스트 클라이언트 (`ai_client.py`)

```python
import socket
import json

HOST = "127.0.0.1"
PORT = 9000
ENC = "utf-8"

def send_and_recv(sock, payload):
    msg = (json.dumps(payload, ensure_ascii=False) + "\n").encode(ENC)
    sock.sendall(msg)
    data = b""
    # 한 줄(\n) 단위로 응답 받기
    while b"\n" not in data:
        chunk = sock.recv(4096)
        if not chunk:
            break
        data += chunk
    return data.decode(ENC).strip()

def main():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as c:
        c.connect((HOST, PORT))
        # 초기 서버 인사 수신
        print("Server hello:", c.recv(4096).decode(ENC).strip())

        # 1) 감정분석 요청
        payload = {
            "task": "sentiment",
            "text": "I absolutely love this library! It made my day."
        }
        print("REQ(sentiment):", payload)
        print("RES:", send_and_recv(c, payload))

        # 2) 요약 요청
        long_text = (
            "Transformers provide state-of-the-art natural language processing capabilities. "
            "They allow tasks such as summarization, translation, question answering, sentiment analysis, "
            "and more with minimal code. This example shows how to connect a socket-based client to an AI "
            "server that runs transformers pipelines and returns JSON responses."
        )
        payload = {
            "task": "summary",
            "text": long_text
        }
        print("REQ(summary):", payload)
        print("RES:", send_and_recv(c, payload))

        # 3) 에러 예시(알 수 없는 task)
        payload = {"task": "unknown", "text": "hello"}
        print("REQ(unknown):", payload)
        print("RES:", send_and_recv(c, payload))

if __name__ == "__main__":
    main()

```

---

## 실행 방법

1. 의존성 설치

```bash
pip install transformers torch --upgrade

```

- CPU만 있어도 동작. (최초 1회 모델 다운로드)
1. 서버/클라이언트 실행

```bash
python ai_server.py
# 새 터미널
python ai_client.py

```

---

## 프로토콜 요약(NDJSON)

- 클라이언트 → 서버: 한 줄에 하나의 JSON
    - `{"task":"sentiment","text":"..."}`
    - `{"task":"summary","text":"..."}`
- 서버 → 클라이언트: 한 줄 JSON 응답
    - 감정분석: `{"ok":true,"task":"sentiment","label":"POSITIVE","score":0.999...}`
    - 요약: `{"ok":true,"task":"summary","summary":"..."}`

---

# 코드 설명

---

# 🖥️ 서버 코드 (`ai_server.py`)

### 1. 기본 설정

```python
HOST = "127.0.0.1"
PORT = 9000
BACKLOG = 5
BUFF = 4096
ENC = "utf-8"

```

- **HOST**: 서버가 바인딩할 IP (localhost)
- **PORT**: 열어둘 포트 (9000)
- **BACKLOG**: 동시에 대기할 수 있는 클라이언트 연결 수
- **BUFF**: 한 번에 읽어올 데이터 크기 (4096바이트)
- **ENC**: 인코딩 방식

---

### 2. AI 모델 로딩

```python
from transformers import pipeline

sentiment = pipeline("sentiment-analysis")
summarizer = pipeline("summarization")

```

- **pipeline**: Hugging Face에서 제공하는 고수준 API.
- `sentiment`: 문장의 긍/부정 감정 판별.
- `summarizer`: 긴 텍스트를 요약.

---

### 3. 요청 처리 함수

```python
def handle_request(payload: Dict[str, Any]) -> Dict[str, Any]:
    task = payload.get("task")
    text = payload.get("text", "")

```

- 클라이언트에서 보낸 JSON을 읽음.
- `task` 값에 따라 작업 분기:
    - `"sentiment"` → 감정 분석 결과 반환
    - `"summary"` → 텍스트 요약 결과 반환
    - 그 외 → 오류 응답

---

### 4. 클라이언트 연결 처리

```python
def client_thread(conn: socket.socket, addr):
    with conn:
        buffer = ""
        conn.sendall(b'{"hello":"ai-server","protocol":"ndjson","tasks":["sentiment","summary"]}\n')

```

- 연결된 클라이언트와 통신을 담당하는 함수.
- 처음 연결되면 "환영 메시지" 전송 (NDJSON 프로토콜 안내).
- 이후 `\n`(개행) 단위로 JSON을 주고받음.

---

### 5. 메인 서버 실행

```python
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.bind((HOST, PORT))
    s.listen(BACKLOG)
    while True:
        conn, addr = s.accept()
        threading.Thread(target=client_thread, args=(conn, addr), daemon=True).start()

```

- TCP 소켓 생성 → IP/포트 바인딩 → `listen()`으로 대기.
- 클라이언트가 연결하면 `accept()`로 받아서 스레드 실행.
- 즉, **멀티클라이언트** 처리 가능.

---

# 📱 클라이언트 코드 (`ai_client.py`)

### 1. 서버 연결

```python
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as c:
    c.connect((HOST, PORT))

```

- 서버(`127.0.0.1:9000`)에 접속.
- 연결 성공 시 서버의 인사 메시지를 수신.

---

### 2. 요청 보내기

```python
payload = {"task": "sentiment", "text": "I absolutely love this library!"}
msg = (json.dumps(payload) + "\n").encode(ENC)
c.sendall(msg)

```

- JSON 형식으로 요청(`task`, `text`) 작성.
- 문자열 끝에 `\n`을 붙여 **NDJSON 프로토콜** 맞춤.

---

### 3. 응답 받기

```python
data = b""
while b"\n" not in data:
    chunk = c.recv(4096)
    data += chunk

```

- 서버도 `\n` 단위로 응답을 주기 때문에, 줄바꿈이 나올 때까지 수신.
- 수신된 JSON을 `json.loads()`로 파싱하면 결과 확인 가능.

---

# 🔄 통신 프로토콜 요약

- **클라이언트 → 서버**: `{"task":"sentiment","text":"hello world"}\n`
- **서버 → 클라이언트**:
    - 감정분석: `{"ok":true,"task":"sentiment","label":"POSITIVE","score":0.999}\n`
    - 요약: `{"ok":true,"task":"summary","summary":"..."}\n`

---
