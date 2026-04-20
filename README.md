### Phase 8: Advanced Traffic Analysis & Applied Cryptography

> [!NOTE]  
> **Objective:** To simulate encrypted Command and Control (C2) infrastructure using deceptive cryptographic material, and subsequently utilize deep packet inspection to hunt the malicious beacon within raw network telemetry.

#### Operation 1: Encrypted C2 Emulation & Infrastructure Debugging
* **Red Team Attack:** Engineered an OpenSSL-based HTTPS listener on Kali Linux. Forged a self-signed 2048-bit RSA X.509 certificate deceptively populated with Microsoft Update metadata (`O=Microsoft Corporation, CN=update.microsoft.com`) to bypass rudimentary SOC analyst inspection.
* **Execution & Environmental Troubleshooting:** * **Challenge 1 (API Mismatch):** The initial deployment utilized the `-SkipCertificateCheck` flag, which failed due to the endpoint running legacy Windows PowerShell (v5.1) rather than PowerShell Core. 
  * **Remediation 1:** Engineered a runtime bypass by manipulating the underlying .NET framework directly (`[Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}`) to force the system to blindly trust the rogue CA.
  * **Challenge 2 (Cryptographic Downgrade):** The execution failed a second time because legacy PowerShell defaulted to the deprecated TLS 1.0 protocol, which the modern OpenSSL listener strictly rejected.
  * **Remediation 2:** Forced the session to upgrade its cryptography to modern standards (`[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12`).
  * **Outcome:** The payload successfully executed, initiating a secure TLS 1.2 handshake to the adversarial listener before gracefully dropping the connection.

#### Operation 2: Deep Packet Inspection & Threat Hunting
* **Blue Team Defense:** Executed raw wire capture (`tcpdump`) during the C2 beaconing phase. Ingested the resulting PCAP into Wireshark for forensic analysis.
* **Analysis & Signal Isolation:**
  * **Initial Hunt:** Applied standard certificate exchange filters (`tls.handshake.type == 11`). Discovered that Windows OS background telemetry (legitimate Microsoft Update traffic) polluted the capture, masking the C2 beacon.
  * **Refined Hunt:** Engineered a highly specific display filter (`ip.addr == 10.10.1.6 and tls.handshake.type == 11`) to eliminate legitimate internet noise. 
  * **Outcome:** Successfully isolated the malicious TLS handshake and extracted the plaintext deceptive certificate strings prior to cryptographic lock, forensically proving the presence of rogue infrastructure.

#### Operation 3: Automated Threat Detection at Scale (Zeek)
* **Blue Team Defense (Automated):** Transitioned from manual packet inspection to automated protocol analysis using Zeek. 
* **Detection Engineering:** Parsed the resulting `x509.log` using `zeek-cut` and developed a custom `awk` pipeline (`awk -F '\t' '$1 == $2'`) to mathematically identify self-signed certificates by isolating rows where the `certificate.issuer` string perfectly matched the `certificate.subject` string.
* **Outcome:** Successfully filtered out thousands of legitimate TLS exchanges and isolated the exact cryptographic signature of the C2 server, alongside known Root CAs (which are inherently self-signed).

<details>
<summary>[+] View Operation Evidence (C2 Generation & PCAP Dissection)</summary>

**C2 Infrastructure Generation (Kali):**
![Openssl Server](./red-team/network-attacks/evidence/openssl_srv.png)

![C2 Cert Generation](./red-team/network-attacks/evidence/kali_tls_c2_capture.png)

**Malicious Certificate Isolation (Wireshark):**
![PCAP Dissection](./blue-team/network-forensics/evidence/wireshark_rogue_cert.png)

**Automated Zeek Threat Hunting (AWK Pipeline):**
![Zeek Automated Parsing](./blue-team/network-forensics/evidence/zeek_detection.png)

![Zeek Automated Parsing](./blue-team/network-forensics/evidence/zeek_true_detection.png)

</details>

### Phase 9: Enterprise Penetration Testing & Cryptanalysis

> [!NOTE]  
> **Objective:** To exploit legacy network resolution protocols (LLMNR/NBT-NS) via Man-in-the-Middle poisoning, harvest cryptographic authentication material from the wire, and execute offline cryptanalysis to recover plaintext credentials.

#### Operation 1: LLMNR Poisoning & Protocol Abuse
* **Red Team Attack:** Deployed `Responder` on the local subnet to aggressively monitor for failed DNS resolution broadcasts. Simulated an endpoint typo (`\\CorpFinanceServer99`), triggering a fallback to LLMNR/NBT-NS.
* **Execution:** `Responder` successfully intercepted the broadcast, masqueraded as the requested server, and forced the Windows endpoint to automatically negotiate authentication, yielding a valid NetNTLMv2 Challenge/Response hash for the target user.

#### Operation 2: Offline Cryptanalysis
* **Red Team Attack:** Extracted the NetNTLMv2 hash from the raw network capture and sanitized the payload for offline cracking to maintain strict OPSEC and avoid endpoint detection mechanisms (e.g., account lockouts, failed login telemetry).
* **Execution:** Utilized `Hashcat` (Mode 5600) paired with a targeted dictionary attack (`rockyou.txt`) to shatter the HMAC-MD5 cryptography. 
* **Outcome:** Successfully reversed the cryptographic response and recovered the plaintext Active Directory credentials.

<details>
<summary>[+] View Operation Evidence (Protocol Poisoning)</summary>

**LLMNR Poisoning & NetNTLMv2 Capture:**
![Responder](./red-team/enterprise-pentest/evidence/responder.png)

![Responder Capture](./red-team/enterprise-pentest/evidence/responder_ntlmv2_capture.png)

![Responder Crack](./red-team/enterprise-pentest/evidence/responder_ntlmv2_crack.png)
*(Note: Cryptanalysis evidence redacted per OPSEC protocols to protect plaintext credential exposure).*
</details>