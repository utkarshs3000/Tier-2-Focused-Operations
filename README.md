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

<details>
<summary>[+] View Operation Evidence (C2 Generation & PCAP Dissection)</summary>

**C2 Infrastructure Generation (Kali):**
![C2 Cert Generation](./red-team/network-attacks/evidence/kali_tls_c2_capture.png)

**Malicious Certificate Isolation (Wireshark):**
![PCAP Dissection](./blue-team/network-forensics/evidence/wireshark_rogue_cert.png)
</details>