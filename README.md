# 🎣Automated Phishing Email Analyser

## 📌<ins>Objective</ins>  
The goal of this project was to build a tool that automates the triage of suspicious emails the way a SOC analyst would, but faster and at scale. In a real SOC environment, analysts receive dozens of phishing alerts per day. This tool reduces manual investigation time by automatically enriching each email with threat intelligence from multiple sources and producing a clear MALICIOUS / SUSPICIOUS / CLEAN verdict with supporting evidence.  
> A Python-based automated phishing analysis tool that parses raw `.eml` files, extracts IOCs, queries multiple OSINT sources, and generates a risk-scored analyst verdict, integrated with Splunk SIEM.

### Skills demonstrated:  
Email header forensics (SPF, DKIM, DMARC analysis)
IOC extraction and threat intelligence enrichment
Python scripting and modular tool development
SIEM integration (Splunk)
Real-world phishing sample analysis

### Required Free API Keys

| Service | Sign Up | Used For |
|---------|---------|----------|
| VirusTotal | virustotal.com | URL/IP/domain reputation |
| AbuseIPDB | abuseipdb.com | IP abuse history |
| AlienVault OTX | otx.alienvault.com | Threat intel pulses |
| URLScan.io | urlscan.io | Live URL scanning |

> **Note:** CyberChef decoder requires no API key as it runs fully offline as a built-in module.


## <ins>Architecture</ins>
```
Raw .eml File
      │
      ▼
┌─────────────────┐
│  Header Parser  │  → SPF/DKIM/DMARC, sending IPs, spoofing detection
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  IOC Extractor  │  → URLs, IPs, domains, attachments, typosquatting
└────────┬────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│        CyberChef Decoder                 │
│  Base64 │ URL Encoding │ Punycode        │
│  fromCharCode │ Hex │ HTML Entities      │
└────────┬─────────────────────────────────┘
         |
         ▼
┌──────────────────────────────┐
│           OSINT Enrichment   |            
│  VirusTotal │ AbuseIPDB │    |
│  URLScan.io │ AlienVault OTX |       
└────────┬─────────────────────┘
         │
         ▼
┌─────────────────────┐
│   Scoring Engine    │  → Risk score 0-100+
└────────┬────────────┘
         │
         ▼
┌───────────────────────────────────────────┐
│  Report Generator                         │
│  → Text report  → JSON report             │
│  → Splunk log (sourcetype=phish_analyzer) │
└───────────────────────────────────────────┘
```
---
## ⚙️<ins>How It Works</ins>  
#### 1. Header Analysis
Parses all email headers to extract security-relevant fields:  
Sender IP addresses from `Received` headers and `X-Originating-IP`  
SPF, DKIM, DMARC authentication results  
Display name vs. actual sender domain mismatch detection  
Reply-To domain mismatch (common phishing tactic)  
Free email provider spoofing as corporate identity  
<img width="1250" height="841" alt="Screenshot 2026-03-17 222312" src="https://github.com/user-attachments/assets/051619ab-a976-4052-a2a2-2e8601219d6b" />

#### 2. IOC Extraction
Automatically pulls all indicators from the email body:  
URLs (including obfuscated and shortened links)  
IP addresses used directly in URLs (raw IP indicator)  
Domain names with suspicious TLD detection  
Attachment metadata and dangerous file extension detection  
Typosquatting detection against known brands  

#### 3. CyberChef Obfuscation Decoder
Automatically detects and decodes obfuscation techniques used by attackers to hide malicious URLs and payloads:

| Technique | Example | What It Reveals |
|-----------|---------|-----------------|
| Base64 in URL | `?hash=bWVyY2lh...` | Hidden victim email address |
| URL Encoding | `washingtonpost.com%20(2).html` | Fake page masquerading as trusted site |
| HTML Entity Encoding | `&#109;&#97;&#105;&#108;` | Obfuscated body content |
| Punycode Domain | `xn--pple-43d.com` | Lookalike brand domain |
| JavaScript fromCharCode | `fromCharCode(104,116,116,112)` | Hidden URLs in scripts |
| Unicode Escapes | `\u0068\u0074\u0074\u0070` | Obfuscated strings |
| Hex Encoding | `\x68\x74\x74\x70` | Hidden payloads |
| Double URL Encoding | `%2520` → `%20` → space | Filter bypass |
| ROT13 | `uggc://rivy.pbz` | Basic string hiding |
| Reversed Strings | `moc.liamg` | Reversed URLs |  
<img width="1394" height="595" alt="Screenshot 2026-03-17 235958" src="https://github.com/user-attachments/assets/709ece95-e8b3-4580-b223-fb1b760b0d42" />  

The attacker pre-loaded the victim's email into the phishing URL so the fake login page would auto-fill their credentials.

#### 4. OSINT Enrichment
Each IOC is queried across 6 threat intelligence sources:  
Source	What It Checks	Free Tier  
VirusTotal	URL, IP, domain reputation across 90+ AV engines	4 req/min  
AbuseIPDB	IP abuse reports and confidence score	1000/day  
AlienVault OTX	Threat intelligence pulse matches	Unlimited  
URLScan.io	Live URL scan with screenshot	Generous  
<img width="1842" height="825" alt="Screenshot 2026-03-17 222437" src="https://github.com/user-attachments/assets/23d6680f-43e6-4b9f-a8fc-58c8d04fc312" />

#### 5. Scoring Engine
Each indicator carries a weighted score. Final scores determine verdict:  
Score	Verdict  
0–29	✅ CLEAN  
30–59	⚠️ SUSPICIOUS  
60–100+	🚨 MALICIOUS  

#### 6. Splunk Integration
Every analysis writes a structured JSON log line to `reports/splunk_alerts.log`. The Splunk Universal Forwarder monitors this file and ingests each verdict into the SIEM automatically.  
Splunk search to view all verdicts:  
```
index=main sourcetype=phish_analyzer 
| table _time, event.verdict, event.risk_score, event.subject, event.from_email
```
<img width="1810" height="800" alt="Screenshot 2026-03-18 001215" src="https://github.com/user-attachments/assets/f74e7dd0-0656-48f4-99a1-e9feb7c1b1ca" />

---

## 🔐 <ins>Why SPF, DKIM & DMARC Matter</ins>  
Email authentication is the first line of defence against phishing. This tool automatically checks all three on every email analysed.

| Protocol | Purpose | If It Fails |
|----------|---------|-------------|
| **SPF** | Verifies the email was sent from an authorised server for that domain | Sender is likely spoofing a domain they don't own |
| **DKIM** | Cryptographic signature proving email content wasn't tampered with in transit | Email may have been modified or forged |
| **DMARC** | Ties SPF and DKIM together & tells receiving servers to reject, quarantine, or report failures | No policy enforced, domain is vulnerable to spoofing |

Together they form the three pillars of email authentication. If all three fail on an incoming email it is a strong indicator the sender is spoofing a domain they don't own.

### 🧪<ins>Real-World Results</ins>
All samples sourced from the phishing_pot public repository are real captured phishing emails.  
Sample	Verdict	Score	Key Findings  
Microsoft Account Alert	🚨 MALICIOUS	100	Display name spoofing, SPF fail, raw IP in URL, AbuseIPDB 100%  
Fake Microsoft Sign-in	🚨 MALICIOUS	100	VT domain 6 detections, Reply-To mismatch (Gmail)  
ADAC Auto Promotion	🚨 MALICIOUS	100	Raw IP URLs, AbuseIPDB 50%, OTX flagged, Seoul IP  
Newsletter (Legitimate)	✅ CLEAN	0	SPF/DKIM/DMARC all passed, no IOCs flagged  
German Prize Scam	🚨 MALICIOUS	100	SPF fail, Moscow IP (AS48347), superkoran.info VT flagged  
Rossmann Fake Promo	🚨 MALICIOUS	100	URL flagged 8/98 VT engines, malicious T.co shortener  
SmartPay Invoice	⚠️ SUSPICIOUS	45	Homoglyph attack — Cyrillic characters impersonating Latin letters  

### Notable Finding - Homoglyph Attack
Sample `sample-5113` demonstrated a sophisticated homoglyph attack where the attacker used Cyrillic Unicode characters visually identical to Latin letters to fake the display name "SmartPay InvoicePPL". The tool flagged this via display name analysis despite the characters appearing legitimate at a glance.  
<img width="1207" height="798" alt="Screenshot 2026-03-17 225502" src="https://github.com/user-attachments/assets/efaf1be4-16bf-4edb-b46f-7dc4d0f8180e" />

<ins>Detection Accuracy</ins>
6/7 malicious emails correctly identified as MALICIOUS
1/7 legitimate email correctly identified as CLEAN (0 false positives)  
> This is more important than you would think as you need to know if the analyser can tell the diference between an attacker and a clean email.

<img width="746" height="884" alt="Screenshot 2026-03-17 222921" src="https://github.com/user-attachments/assets/602af0f0-cb65-4ad8-9041-fbd03b0027e1" />  
<img width="1250" height="841" alt="Screenshot 2026-03-17 222312" src="https://github.com/user-attachments/assets/a46a4185-750f-4e5c-899b-e275db66aae0" />  

Multi-source enrichment consistently corroborated verdicts across independent sources

---

### 📁 Project Structure
```
phish-analyzer/
├── phish_analyzer.py          # Main entry point
├── config.json.example        # API key template (keys not included)
├── requirements.txt           # Python dependencies
├── splunk_integration.conf    # Splunk forwarder configuration
├── modules/
│   ├── header_parser.py       # Email header forensics
│   ├── ioc_extractor.py       # URL/IP/domain extraction
│   ├── vt_checker.py          # VirusTotal API integration
│   ├── cyberchef_decoder.py   # Obfuscation decoder (Base64, URL encoding, Punycode, Hex, Unicode, ROT13, fromCharCode)
│   ├── osint_enrichment.py    # AbuseIPDB, OTX, URLScan, IPInfo, PhishTank
│   ├── indicator_engine.py    # Risk scoring and verdict engine
│   └── report_generator.py   # Report output + Splunk log
├── samples/
│   └── test_phish.eml         # Sample phishing email for testing
├── reports/                   # Generated reports (auto-created)
└── FINDINGS.md                # Analysis results summary
```
---
### Splunk Integration
Add to Splunk Universal Forwarder `inputs.conf`:
```
[monitor:///path/to/phish-analyzer/reports/splunk_alerts.log]
index = main
sourcetype = phish_analyzer
disabled = false
```
<img width="711" height="103" alt="Screenshot 2026-03-17 224451" src="https://github.com/user-attachments/assets/243da2b6-a66b-4830-8033-0ce5907644ad" />

### 📊 <ins>Example json Report Output</ins>

<img width="725" height="884" alt="Screenshot 2026-03-17 224621" src="https://github.com/user-attachments/assets/e815bf1f-160a-4a4c-be67-dda150093e23" />
<img width="508" height="207" alt="Screenshot 2026-03-17 224642" src="https://github.com/user-attachments/assets/d6b955d8-a138-4a14-8bd1-637f17dd525f" />

## ⚒️ <ins>Future Improvements</ins>  
WhoisXML domain age integration (flag domains < 30 days old)  
Attachment sandboxing via any.run or hybrid-analysis API  
MITRE ATT&CK TTP mapping for each detection indicator  
Splunk dashboard with verdict trends and top IOCs  

Built as part of a SOC Analyst Level 1 portfolio. Demonstrates automated email triage, multi-source threat intelligence enrichment, IOC extraction, and SIEM integration.

---
