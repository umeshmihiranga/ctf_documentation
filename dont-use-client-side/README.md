# picoCTF — dont-use-client-side

## 📌 Challenge Information

- **Category:** Web Exploitation
- **Difficulty:** Easy
- **Event:** picoCTF 2019
- **Challenge:** `dont-use-client-side`

---

## 🎯 Objective

Break into a "secure" login portal and recover the password.

---

## 🔎 Enumeration / Initial Inspection

The challenge presents a login page with a password field and a `verify` button.

Instead of immediately trying to brute-force the password, inspect the page source and JavaScript using **Developer Tools**.

The page contains an inline JavaScript function:

```javascript
function verify() {
    checkpass = document.getElementById("pass").value;
    split = 4;

    if (checkpass.substring(0, split) == 'pico') {
        if (checkpass.substring(split*6, split*7) == 'eb02') {
            if (checkpass.substring(split, split*2) == 'CTF{') {
                if (checkpass.substring(split*4, split*5) == 'ts_p') {
                    if (checkpass.substring(split*3, split*4) == 'lien') {
                        if (checkpass.substring(split*5, split*6) == 'lz_2') {
                            if (checkpass.substring(split*2, split*3) == 'no_c') {
                                if (checkpass.substring(split*7, split*8) == 'b45}') {
                                    alert("Password Verified")
                                }
                            }
                        }
                    }
                }
            }
        }
    }
    else {
        alert("Incorrect password");
    }
}
````

---

## 🧩 Analysis

The important variable is:

```javascript
split = 4;
```

This means the password is divided into **4-character chunks**.

The JavaScript checks different positions of the password.

| Position | JavaScript condition | Value  |
| -------- | -------------------- | ------ |
| 0–3      | `substring(0, 4)`    | `pico` |
| 4–7      | `substring(4, 8)`    | `CTF{` |
| 8–11     | `substring(8, 12)`   | `no_c` |
| 12–15    | `substring(12, 16)`  | `lien` |
| 16–19    | `substring(16, 20)`  | `ts_p` |
| 20–23    | `substring(20, 24)`  | `lz_2` |
| 24–27    | `substring(24, 28)`  | `eb02` |
| 28–31    | `substring(28, 32)`  | `b45}` |

Although the conditions appear in a confusing order, the `substring()` positions reveal the correct order.

Reconstructing the password:

```text
pico
CTF{
no_c
lien
ts_p
lz_2
eb02
b45}
```

Therefore:

```text
picoCTF{no_clients_plz_2eb02b45}
```

---

## 🚩 Flag

```text
picoCTF{no_clients_plz_2eb02b45}
```

---

## 🧠 What I Learned

### Client-Side Authentication

The main vulnerability is that the password verification logic exists entirely in the client's JavaScript.

Anyone can:

1. Open Developer Tools.
2. Inspect the JavaScript.
3. Read the password validation logic.
4. Reconstruct the correct password.

There is no real secret being protected on the server.

### Security Lesson

**Never implement authentication or authorization entirely on the client side.**

Client-side JavaScript can be inspected and modified by the user.

Sensitive security checks should always be performed **server-side**.

---

## 🛠️ Tools Used

* Web Browser
* Developer Tools
* JavaScript source inspection

---

## 🔑 Key Takeaway

> **Never trust the client.**

Anything delivered to the browser—including JavaScript, HTML, and client-side validation logic—should be considered visible and potentially modifiable by an attacker.

````
