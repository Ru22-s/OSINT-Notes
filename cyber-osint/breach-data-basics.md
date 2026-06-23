# breach-data-basics.md

> *“Breaches are the OSINT analyst’s archive of human behavior. A password dump is not just credentials—it is a map of how a person thinks, reuses, and trusts.”*


## 1. Overview: What is Breach Data?

In OSINT, **breach data** refers to leaked, stolen, or inadvertently exposed datasets containing Personally Identifiable Information (PII), credentials, financial records, or internal communications. These datasets originate from compromised servers, misconfigured cloud storage, third-party vendor leaks, or insider threats.

For an investigator, breach data is a goldmine—not for exploitation, but for **correlation, pattern discovery, and threat actor profiling**. It allows you to link aliases, discover password reuse (a key pivot technique), map online presence, and identify unauthorized access to an organization's assets.

---

## 2. Types of Breached Data

Different breaches contain different data types. Knowing what you are looking at is critical:

| Data Type | Examples | OSINT Value |
| :--- | :--- | :--- |
| **Credentials** | Email:password, Username:hash | Pivot to other platforms; password reuse discovery |
| **Personal Identifiable Info (PII)** | Full name, SSN, DoB, Address, Phone | Identity mapping, building comprehensive profiles |
| **Financial** | Credit card numbers, transaction logs, banking details | Fraud detection, tracing financial flows |
| **Medical** | Health records, prescriptions, insurance IDs | Dark web market intelligence, insurance fraud |
| **Corporate / Internal** | Source code, internal emails, project plans, employee directories | Corporate espionage indicators, supply chain mapping |
| **Geolocation** | GPS coordinates, IP logs, WiFi BSSIDs | Physical movement tracking, infrastructure mapping |
| **Device Fingerprints** | User-Agent strings, screen resolution, installed fonts | Tracking across accounts without cookies (browser fingerprinting) |

---

## 3. Where to Find Breach Data (Legitimate Sources)

> **OPSEC Rule:** *Never* download full breach dumps containing live PII unless you have explicit legal authorization. Use searchable aggregators or APIs first.

### 3.1 Surface Web (Public & Legal)
- **Have I Been Pwned (HIBP)** — Check if an email or phone number appears in known breaches. Free API available.
- **Dehashed** — Searchable breach database (paid, but widely used by investigators).
- **BreachDirectory** — Similar to Dehashed; API for correlation.
- **Firefox Monitor** — Powered by HIBP.
- **Snusbase** — Another searchable archive.
- **IntelX** — Searchable archive of Telegram, Discord, and private breaches.

### 3.2 Paste Sites (Ephemeral Leaks)
- **Pastebin** — Monitor private/public pastes for credential dumps using scraping tools.
- **Rentry.co** — Increasingly used for fresh dumps.
- **ControlC** — Another common paste site.
- **GitHub Gists** — Developers sometimes accidentally push `.env` or credential files here. Use Github Dorks (e.g., `filename:.env`, `password`, `secret`).

### 3.3 Deep & Dark Web
- **Breach Forums** (e.g., BreachForums successors) — Monitor for new dumps, but **read-only**; do not download full files.
- **Telegram Channels** — Many threat actors release "samples" to build reputation. Monitor via Telescoop or custom scrapers.
- **Tor / I2P Markets** — Where full dumps are sold. Intelligence gathering here requires extreme OPSEC (Whonix/Tails).

### 3.4 Leaked Databases (Dumps)
- **RaidForums archives** (historical) — Often mirrored in research archives.
- **Public S3 Buckets / Cloud storage** — Misconfigurations are found daily via tools like BucketStream.

---

## 4. OSINT Applications of Breach Data

| Application | Method |
| :--- | :--- |
| **Account Pivoting** | Find an email in a breach → Check if the associated password is reused on LinkedIn or other targets. |
| **Alias Clustering** | Same phone number or recovery email appearing across multiple dumps links different aliases together. |
| **Password Analysis** | Study password patterns (capitalization, suffixes, pet names) to predict passwords for new targets. |
| **Compromise Verification** | When an organization suspects a breach, OSINT analysts check dumps for their specific domain. |
| **Threat Actor Attribution** | Threat actors often use the same handles across breach forums; leaked private messages reveal affiliations. |
| **Social Engineering Defense** | Identify what personal data employees have exposed (e.g., birthday, mother's maiden name) to warn against spear-phishing. |

---

## 5. Working with Breach Dumps (Parsing & Validation)

If you have **authorized, sandboxed access** to a dataset (e.g., for internal red-team exercises or a CTF), here is how to handle it efficiently:

### 5.1 File Identification
Breach dumps come in various formats:
- **CSV / TSV** — Standard delimiter-separated files.
- **JSON** — Structured but heavy.
- **SQL (.sql)** — Database exports; import into SQLite or MySQL.
- **Combined / TXT** — Raw `email:password` per line.
- **7z / RAR / ZIP** — Almost always compressed. Use `7z x -p` if password-protected (never run untrusted binaries).

### 5.2 Parsing with Command Line

**For CSV/TSV (using `awk`, `cut`, `grep`):**
```bash
# Extract only emails from a comma-separated dump
cut -d',' -f1 breach.csv > emails.txt

# Filter for a specific domain (e.g., target.com)
grep -i "@target.com" breach.csv

# Sort and count unique entries
sort breach.txt | uniq -c | sort -nr
```

**For JSON (using `jq` — essential tool):**
```bash
# Extract emails and passwords from a nested JSON array
cat breach.json | jq -r '.[] | "\(.email):\(.password)"' > extracted.txt

# Find entries with a specific domain
cat breach.json | jq '.[] | select(.email | contains("@target.com"))'
```

**For SQL Dumps (using SQLite):**
```bash
# Import into SQLite for fast querying
sqlite3 breach.db
> .mode csv
> .import dump.csv breached_table
> SELECT email FROM breached_table WHERE email LIKE '%@target.com';
```

### 5.3 Hash Identification & Cracking (Authorized Only)
Dumps often store hashed passwords (MD5, SHA-1, bcrypt, NTLM).

- **Identify the hash type:** Use `hashid` or `hash-identifier`.
- **Crack (if authorized):** Use `hashcat` or `john`.
```bash
# Example: Crack MD5 hashes with rockyou.txt
hashcat -m 0 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt
```
> **CRITICAL:** Only crack hashes if you own the accounts or have explicit written permission. This is illegal in many jurisdictions otherwise.

### 5.4 Validation (Checking if a credential is "live")
A dumped credential might be old. Validate before reporting:
1.  Check the breach's **date** (look for timestamps in the dump).
2.  Use the HIBP API to confirm the email appears in recent breaches.
3.  Check if the password matches the hash pattern (e.g., if the dump says `password:5f4dcc...` you can crack it, but **never** attempt to log into a live account unless authorized—that crosses into illegal "hacking").

---

## 6. OPSEC & Handling Guidelines (NON-NEGOTIABLE)

Breach data contains **real, non-consensually exposed PII** of innocent people. Mishandling it is both unethical and often illegal.

- [ ] **Never download full dumps** on your primary machine or network. Use an isolated VM without internet access (air-gapped or filtered).
- [ ] **Never re-share** or redistribute breach data. Sharing raw dumps makes you a distributor of stolen data.
- [ ] **Anonymize when reporting**: If you must reference a finding (e.g., "we found employee emails exposed"), redact the full credential. Show only the domain and the breach name.
- [ ] **Purge after use**: Unless required for long-term intelligence (and legally held), securely delete dumps using `shred -vfz` or `srm` after extraction.
- [ ] **Don't click links** within dumps—they may contain tracking pixels that alert the threat actor.
- [ ] **Assume the dump is booby-trapped**: Malware analysts often find embedded malicious payloads in CSV files (via Excel macro exploits).

---

## 7. Essential Tools

| Tool | Purpose |
| :--- | :--- |
| **jq** | Parse JSON dumps like a pro |
| **grep / ripgrep (rg)** | Ultra-fast text searching through massive dumps |
| **7zip / p7zip** | Extract encrypted archives |
| **hashcat** | Password cracking (authorized use only) |
| **John the Ripper** | Alternative hash cracker |
| **sqldiff / sqlite3** | Query SQL dumps quickly |
| **HIBP API** | Quick breach presence check for emails |
| **Dehashed / Snusbase** | Commercial search interfaces (safer than raw dumps) |
| **Metadata Anonymization Toolkit (MAT)** | Clean metadata before sharing any extracted findings |
| **VS Code / Sublime** | View large files with syntax highlighting (install `Large File` extensions) |

---

## 8. Legal & Ethical Considerations

> **Ignorance of the law is not a defense.**

- **US:** The Computer Fraud and Abuse Act (CFAA) criminalizes unauthorized access. Possession of stolen credentials can be construed as intent to commit fraud.
- **EU:** GDPR imposes fines for mishandling EU citizen data. Even *viewing* certain types of exposed data without a lawful basis can violate privacy regulations.
- **Global:** Many countries treat downloaded breach dumps as "stolen property."

**Safe Harbors for OSINT Investigators:**
1.  Use **API lookups** (HIBP, Dehashed) rather than downloading full datasets.
2.  Work only on **sample snippets** provided for research (e.g., 100 records for pattern analysis).
3.  Get **written authorization** from your client or employer before handling any raw breach data.
4.  If you stumble upon a live, unredacted dump of your own organization's data, contact the CSIRT team immediately—do not investigate it further without their direction.

---

## 9. Quick Tips for the Field

1.  **Always check the date first.** A 2012 breach has minimal value for active password pivots, but great value for historical alias mapping.
2.  **Look for recovery fields.** Breaches with `alt_email` or `phone` fields are priceless for linking multiple identities.
3.  **Password analysis:** The most common passwords (e.g., `123456`, `password`, `qwerty`) tell you about the *population*, but the *unique* passwords (e.g., `P@ssw0rd123!`) tell you about the *individual*.
4.  **Combine breach data with OSINT profile data.** If a LinkedIn profile says "Works at Acme Corp since 2020," and a 2021 breach shows that email, you have a high-confidence employment verification.
5.  **Don't forget combo-lists.** These are curated lists of `email:password` aggregated from *multiple* breaches. They are the most dangerous and useful for pivot analysis.

---

> **Final Warning:** Breach data is volatile and toxic. Handle it like hazardous material—with protective gear (OPSEC), clear protocols (Logs), and a hazmat suit (Legal authorization). The goal is intelligence, not infiltration.
