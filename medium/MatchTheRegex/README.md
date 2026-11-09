# CTF Write-up: MatchTheRegex

## 1. Challenge Information

| Field | Details |
|---|---|
| Challenge | MatchTheRegex |
| Platform | CyLab Security Academy |
| Original Event | picoCTF 2023 |
| Category | Web Exploitation |
| Difficulty | Medium |
| Technique | Regular Expression Matching |
| Status | Solved |

## 2. Challenge Objective

The challenge provides a simple webpage containing:
- A text input field.
- A Submit button.
- A server-side validation mechanism.

The objective was to identify the expected regular expression and submit an input that satisfies the pattern to retrieve the flag.

## 3. Initial Reconnaissance

Opened the challenge webpage and tested a basic input:

```text
abc
```

The application returned:

```text
wrong match! Try again!
```

This indicated that the input did not satisfy the expected pattern.

### Inspecting the webpage

Opened browser Developer Tools:

1. Pressed `F12`.
2. Navigated to the Elements tab.
3. Inspected the HTML structure.
4. Located the inline JavaScript function responsible for submitting the input.

Discovered the following code:

```javascript
function send_request() {
    let val = document.getElementById("name").value;

    // ^a.....y!?

    fetch("/flag?input="+val)
        .then(res => res.text())
        .then(res => {
            const res_json = JSON.parse(res);
            alert(res_json.flag);
            return false;
        });

    return false;
}
```

**Important discovery:** The JavaScript comment exposed a regular expression clue.

## 4. Regular Expression Analysis

Discovered pattern:

```regex
^a.....y!?
```

### Pattern breakdown

| Component | Explanation |
|---|---|
| `^` | Matches the beginning of the string |
| `a` | Requires the character `a` |
| `.` | Matches any single character except a newline by default |
| `.....` | Five wildcard characters |
| `y` | Requires the character `y` |
| `!` | Literal exclamation mark |
| `?` | Makes the preceding `!` optional |

### Visual representation

```text
^ a . . . . . y !?
  | |         |  |
  | |         |  └── Optional !
  | |         └───── Required y
  | └─────────────── Five wildcard characters
  └───────────────── Required first character
```

The expected basic structure is:

```text
a + five characters + y + optional !
```

## 5. Exploitation

After understanding the pattern, constructed test inputs.

### Payload 1

```text
a12345y
```

Reasoning:
- Starts with `a`.
- Contains five characters: `12345`.
- Ends with `y`.
- Does not include the optional `!`.

### Payload 2

```text
abcdefy!
```

Reasoning:
- Starts with `a`.
- Contains five characters: `bcdef`.
- Followed by `y`.
- Includes the optional `!`.

Both inputs successfully matched the expected pattern.

## 6. Flag Captured

```text
academy{succ3ssfully_matchtheregex_081ae5fb}
```

## 7. Additional Learning

### A. What is a Regular Expression?

A Regular Expression (Regex) is a sequence of characters used to describe a search pattern.

Regex is commonly used for:
- Input validation.
- Email validation.
- Password validation.
- Searching and extracting text.
- Filtering application data.

### B. Important Regex Metacharacters

| Symbol | Meaning | Example |
|---|---|---|
| `.` | Any single character (except newline by default) | `a.c` |
| `^` | Start of string | `^admin` |
| `$` | End of string | `admin$` |
| `*` | Zero or more repetitions | `ab*` |
| `+` | One or more repetitions | `ab+` |
| `?` | Zero or one repetition | `colou?r` |
| `{n}` | Exactly n repetitions | `a{3}` |
| `[abc]` | Any one listed character | `[abc]` |
| `[0-9]` | Any digit | `[0-9]` |
| `\d` | Digit | `\d` |
| `\w` | Word character | `\w` |
| `\s` | Whitespace | `\s` |

### C. Difference Between `^` and `$`

Consider:

```regex
^admin$
```

This matches exactly:

```text
admin
```

But not:

```text
admin123
myadmin
```

Why?

- `^` anchors the match at the beginning.
- `$` anchors the match at the end.

Our challenge pattern did not contain `$`, so it did not explicitly require the entire input to end after the optional exclamation mark.

### D. Regex Matching in JavaScript

JavaScript provides the `RegExp.test()` method.

Example:

```javascript
const pattern = /^a.....y!?/;

console.log(pattern.test("a12345y"));
```

Output:

```text
true
```

It returns a Boolean:
- `true` means a match was found.
- `false` means no match was found.

## 8. Security Lessons

1. Client-side JavaScript can expose useful information about application validation.
2. Browser Developer Tools are valuable during web application reconnaissance.
3. HTML comments and JavaScript comments can accidentally reveal implementation details.
4. Regular expressions can be used to enforce input formats.
5. Client-side validation should never be trusted as the only security control.
6. Server-side validation must independently verify user input.

## 9. Tools Used

- Brave Browser
- Browser Developer Tools
- Elements Inspector
- JavaScript inspection
- Regular Expression analysis

## 10. Final Takeaways

This challenge demonstrated how to inspect a web application, identify a regular expression clue, understand its components, and construct valid input.

**Main technique learned:** Regular Expression Analysis.

**Key discovery:** `^a.....y!?`

**Result:** Successfully retrieved the challenge flag.
