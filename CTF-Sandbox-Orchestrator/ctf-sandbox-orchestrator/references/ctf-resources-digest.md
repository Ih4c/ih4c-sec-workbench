# CTF Resources Quick Reference

> Curated from [awesome-ctf-resources](https://github.com/devploit/awesome-ctf-resources) and [awesome-ctf](https://github.com/apsdehal/awesome-ctf)
> Organized by CTF challenge category, keeping only the most practical tools and resources.

---

## General Frameworks

| Tool | Purpose | Link |
|------|------|------|
| Pwntools | Exploit development framework (Python) | https://github.com/Gallopsled/pwntools |
| ctf-tools | One-click CTF tool suite installer | https://github.com/zardus/ctf-tools |
| Ciphey | AI-powered automatic decryption | https://github.com/ciphey/ciphey |
| CyberChef | Online encode/decode and crypto | https://gchq.github.io/CyberChef/ |

---

## Web

### Tools
| Tool | Purpose |
|------|------|
| Burp Suite | HTTP intercept / replay / scan |
| SQLMap | SQL injection |
| XSStrike | XSS detection |
| dirsearch | Directory discovery |
| JWT_Tool | JWT attacks |
| SSRFmap | SSRF exploitation |

### Common Challenge Points
- SQL injection (union / blind / time-based blind / stacked)
- XSS (reflected / stored / DOM)
- SSRF (intranet probing / cloud metadata)
- File upload (extension / MIME / content detection bypass)
- Deserialization (PHP / Java / Python pickle)
- Template injection (SSTI)
- JWT forgery / key confusion

### Payload References
- https://github.com/swisskyrepo/PayloadsAllTheThings
- https://book.hacktricks.wiki/

---

## Reverse Engineering

### Tools
| Tool | Purpose |
|------|------|
| IDA Pro / Ghidra | Decompilation |
| radare2 / r2 | CLI analysis |
| angr | Symbolic execution |
| Frida | Dynamic hooking |
| GDB + pwndbg | Debugging |
| uncompyle6 | Python decompilation |
| jadx | Android decompilation |
| dnSpy | .NET decompilation |

### Common Challenge Points
- Algorithm reconstruction (encryption / encoding / custom)
- Anti-debugging / anti-VM bypass
- Packers / obfuscation (UPX / VMProtect / OLLVM)
- Constraint solving via symbolic execution
- Dynamic hooking to bypass checks
- Go / Rust reversing (symbol recovery)

---

## Pwn

### Tools
| Tool | Purpose |
|------|------|
| Pwntools | Exploit authoring |
| GDB + pwndbg/GEF | Debugging |
| ROPgadget | ROP chain construction |
| one_gadget | libc one-shot gadgets |
| checksec | Mitigation detection |
| LibcSearcher | libc version identification |

### Common Challenge Points
- Stack overflow (ret2text / ret2libc / ret2shellcode / ROP)
- Heap exploitation (UAF / double free / tcache / fastbin)
- Format string (arbitrary read/write)
- Integer overflow
- Kernel pwn (privilege escalation / race conditions)
- Sandbox escape (seccomp bypass)

### Common Payload Patterns
```python
# ret2libc template
from pwn import *
elf = ELF('./vuln')
libc = ELF('./libc.so.6')
p = process('./vuln')
# leak libc base → calculate system/binsh → overwrite ret
```

---

## Crypto

### Tools
| Tool | Purpose |
|------|------|
| SageMath | Mathematical computation |
| RsaCtfTool | Automated RSA attacks | 
| hashcat/john | Hash cracking |
| CyberChef | Encode/decode |
| z3 (SMT solver) | Constraint solving |

### Common Challenge Points
- RSA (small public exponent / common modulus / Wiener / Coppersmith)
- AES (ECB / CBC padding oracle / bit flipping)
- Classical ciphers (Caesar / Vigenere / transposition)
- Hash length extension attack
- Elliptic curves (ECDSA nonce reuse)
- Lattice cryptography (LLL / CVP)

---

## Forensics

### Tools
| Tool | Purpose |
|------|------|
| Volatility | Memory forensics |
| Autopsy/Sleuth Kit | Disk forensics |
| Wireshark | Traffic analysis |
| binwalk | Firmware / file extraction |
| foremost | File recovery |
| exiftool | Metadata extraction |

### Common Challenge Points
- Memory dump analysis (processes / credentials / malicious code)
- PCAP traffic analysis (HTTP / DNS / TCP reassembly)
- Filesystem analysis (deleted file recovery / hidden partitions)
- Log analysis (web logs / system logs)
- Disk image analysis

---

## Misc/Stego

### Tools
| Tool | Purpose |
|------|------|
| StegSolve | Image steganography analysis |
| zsteg | PNG/BMP steganography |
| steghide | JPEG steganography |
| Audacity | Audio analysis |
| strings/xxd | Basic analysis |
| file/binwalk | File type identification |

### Common Challenge Points
- LSB steganography (least significant bits of images)
- File header repair / concatenation
- QR codes / barcodes
- Audio spectrogram steganography
- ZIP fake encryption / known-plaintext attack
- Encoding identification (Base64 / Hex / Morse / Braille)

---

## Online Platforms

| Platform | Highlights | Link |
|------|------|------|
| CTFTime | Event calendar + writeups | https://ctftime.org/ |
| HackTheBox | Hands-on practice boxes | https://www.hackthebox.com/ |
| TryHackMe | Guided learning | https://tryhackme.com/ |
| PicoCTF | Beginner-friendly | https://picoctf.org/ |
| pwnable.kr | Pwn-focused | http://pwnable.kr/ |
| cryptopals | Crypto-focused | https://cryptopals.com/ |
| OverTheWire | War series challenges | https://overthewire.org/ |
| Root-Me | Comprehensive challenges | https://www.root-me.org/ |

---

## Writeup Resources

| Resource | Link |
|------|------|
| CTFTime Writeups | https://ctftime.org/writeups |
| 0xdf hacks stuff | https://0xdf.gitlab.io/ |
| LiveOverflow (YouTube) | https://www.youtube.com/c/LiveOverflow |
| John Hammond (YouTube) | https://www.youtube.com/c/JohnHammond010 |
| IppSec (HTB walkthrough) | https://www.youtube.com/c/ippsec |
