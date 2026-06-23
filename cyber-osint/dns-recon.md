# dns-recon.md

> *“DNS is the phonebook of the internet, but in OSINT, it's also the blueprint of the castle. The records an organization leaves exposed tell you exactly where their defenses are, and more importantly, where they aren't.”*


## 1. Overview: What is DNS Reconnaissance?

DNS (Domain Name System) reconnaissance is the process of mapping an organization's digital infrastructure by querying DNS records. In OSINT, this is a **critical passive and active phase**—it reveals IP addresses, mail servers, subdomains, cloud providers, third-party services, and historical infrastructure without ever touching the target's web application directly.

For penetration testers and investigators, DNS recon is often the **first actionable footprint**. It translates a domain name (e.g., `target.com`) into a network map: "Which servers talk to the outside world, and what are they running?"

---

## 2. Core DNS Record Types (Cheat Sheet)

Knowing what each record signifies is half the battle. Here is the investigator's perspective:

| Record Type | Purpose | OSINT Value |
| :--- | :--- | :--- |
| **A** | IPv4 address | Direct server location. Unusual A records (e.g., a subdomain pointing to a home IP) indicate shadow IT. |
| **AAAA** | IPv6 address | Modern infrastructure mapping. |
| **MX** | Mail exchange servers | Identifies email providers (e.g., Google Workspace, M365). Also reveals internal mail server naming patterns. |
| **NS** | Name servers | Defines the DNS provider. Points to hosting platforms (e.g., AWS Route53, Cloudflare). |
| **CNAME** | Canonical name (aliases) | Maps subdomains to other domains. Great for discovering CDNs, SaaS apps (e.g., `app.target.com` → `target.azurewebsites.net`). |
| **TXT** | Text metadata | Crucial! Contains **SPF**, **DKIM**, **DMARC** (email security misconfigs), and often **domain ownership verification strings** (e.g., Google Search Console, Stripe). |
| **SOA** | Start of Authority | Contains the primary NS and the admin email. Often reveals the domain administrator's email. |
| **SRV** | Service locators | Points to specific services (e.g., VoIP, Active Directory). Exposes internal service ports. |
| **PTR** | Reverse DNS | Maps an IP back to a domain. Reveals the infrastructure's internal naming schemes. |
| **CAA** | Certificate Authority Authorization | Defines which CAs can issue certs—good for detecting potential SSL/TLS misconfigurations. |

---

## 3. Passive vs. Active DNS Recon

| Approach | Description | Stealth Level | Typical Tools |
| :--- | :--- | :--- | :--- |
| **Passive** | Query historical records and third-party datasets that have *already* queried the DNS. Does NOT send a query to the target's NS. Leaves zero logs. | **Highest** (Undetectable) | SecurityTrails, Censys, crt.sh, DNSdumpster, PassiveTotal. |
| **Active** | Send direct queries to the target's NS or brute-force subdomains. Generates traffic and logs. | **Medium/Low** (Detection possible) | `dig`, `nslookup`, `dnsrecon`, `subfinder`, `amass`, `gobuster` (DNS mode). |

---

## 4. Essential Tools

### 4.1 Command-Line (CLI) Classics
- **`dig`** (Linux/macOS): The Swiss Army knife. Flexible, precise, scriptable.
- **`nslookup`** (Cross-platform): Old but reliable; good for interactive mode.
- **`host`** (Linux): Quick and simple lookups.

### 4.2 Active Enumeration Frameworks
- **`dnsrecon`** (Python): Full-featured; supports zone transfers, brute-force, reverse lookups.
- **`dnsenum`**: Multithreaded perl script for comprehensive enumeration.
- **`fierce`**: Recon on a specific target, attempts to locate non-contiguous IP spaces.
- **`subfinder`** (Go): Fast subdomain enumeration using passive sources (binary).
- **`amass`** (OWASP): The gold standard for thorough mapping. Combines passive and active techniques.

### 4.3 Web-Based Passive Sources
- **`crt.sh`**: Certificate Transparency logs. **The #1 source** for finding subdomains.
- **`DNSdumpster`**: Visual mapping and passive records.
- **`SecurityTrails`**: Historical DNS database (API available).
- **`ViewDNS.info`**: Tools like "Reverse IP Lookup" and "DNS History."
- **`ZoomEye` / `Censys`**: Also map DNS fingerprints across the internet.

---

## 5. Step-by-Step Methodology

### Phase 1: Passive Recon (Start Here)
*Goal: Get a baseline without alerting the target.*

1.  **Certificate Transparency (crt.sh):**
    ```bash
    # Query crt.sh for all subdomains issued for target.com
    curl -s "https://crt.sh/?q=%.target.com&output=json" | jq -r '.[].name_value' | sort -u
    ```
    *Why:* Every SSL/TLS certificate is publicly logged. This finds subdomains even if the company forgot to update their own DNS records.

2.  **Historical DNS (SecurityTrails / ViewDNS):**
    - Check for old A records. The company might have moved hosting (e.g., from AWS to Azure), but old IP ranges still exist and may contain forgotten staging environments.

3.  **Reverse IP Lookup (DNSdumpster / ViewDNS):**
    - Check which *other* domains share the same IP. This identifies shared hosting or co-located infrastructure.

### Phase 2: Standard Active Enumeration
*Goal: Get the live, current DNS configuration for the root domain.*

4.  **Gather Root Records:**
    ```bash
    # Query specific records for the apex domain
    dig target.com A
    dig target.com MX
    dig target.com NS
    dig target.com TXT
    dig target.com SOA
    ```

5.  **Analyze the TXT Record (Security Posture):**
    - Extract SPF/DMARC policies. If `v=spf1 +all` exists, the domain is spoofable.
    - Look for `"google-site-verification=..."`—this leaks the authenticated Google Search Console account.

6.  **Attempt a DNS Zone Transfer (AXFR):**
    *Rarely succeeds, but essential to test.*
    ```bash
    # Find NS servers first, then try each
    dig target.com NS
    dig @ns1.target.com target.com AXFR
    ```
    If it works, you have a **complete internal network map**.

### Phase 3: Subdomain Discovery (The Real Treasure)
*Goal: Find dev, staging, admin panels, and backend APIs.*

7.  **Dictionary Brute-Force (Active):**
    Use a robust wordlist (e.g., `subdomains-top1million-50000.txt`).
    ```bash
    # Using dnsrecon
    dnsrecon -d target.com -D /usr/share/wordlists/dns/big.txt -t brt
    
    # Using fierce (helps identify adjacent IPs)
    fierce --domain target.com --wordlist /path/to/wordlist
    ```

8.  **Mass DNS Resolution (Tool-assisted):**
    ```bash
    # Using subfinder (very fast, combines 10+ passive sources)
    subfinder -d target.com -o subs.txt
    
    # Using Amass (intensive, passive + active)
    amass enum -d target.com -o amass_results.txt
    ```

9.  **Resolve IPs for Found Subdomains:**
    ```bash
    cat subs.txt | dnsx -a -resp -o resolved_subs.txt
    ```

### Phase 4: Reverse DNS & IP Mapping
*Goal: Map entire IP ranges to understand infrastructure boundaries.*

10. **Identify CIDR Ranges:**
    - Use `whois` on the target IP to get the ASN (Autonomous System Number).
    - Map all IPs in that ASN.
    ```bash
    whois 1.2.3.4 | grep -i "inetnum" # Finds the range
    ```

11. **PTR (Reverse DNS) Lookup:**
    ```bash
    # Check what the IP resolves back to
    dig -x 1.2.3.4
    ```
    *Why:* Internal naming conventions (e.g., `prod-web-01.internal.target.com`) often appear here, revealing internal architecture.

---

## 6. Advanced Tricks & Patterns

- **Cloud Detection:** If CNAME points to `amazonaws.com`, `azurewebsites.net`, or `googleapis.com`—they are in the cloud. Check if the S3 bucket or Azure blob is misconfigured (e.g., `target-bucket.s3.amazonaws.com` might be open).
- **Staging Branches:** Look for `dev.target.com`, `test.target.com`, `staging.target.com`—these often have weaker security and use default credentials.
- **Admin Panels:** Check for `admin.target.com`, `cpanel.target.com`, `mail.target.com` (often a distinct mail server with a webmail login).
- **Internal Subdomains:** NS records pointing to `dns1.target.com` and `dns2.target.com` suggest fully self-hosted DNS (often revealing internal naming conventions like `infra-dc1`).

---

## 7. Analysis & Correlation

Once you have the data, map it:

1.  **Hosting Provider:** Who hosts them? (AWS, DigitalOcean, OVH, on-premise).
2.  **Email Provider:** Do MX records point to Google, M365, or a self-hosted Exchange?
3.  **CDN Presence:** Are they behind Cloudflare/CloudFront? If Yes, the A record will show a CDN IP (not the origin IP). Work on finding the origin IP via Censys or old historical records (SecurityTrails).
4.  **Entry Points:** Which subdomains handle authentication? `login.target.com`, `api.target.com`. These are high-priority targets for vulnerability scanning.
5.  **Zone Transfer:** If it works, document it immediately as a Critical finding—it exposes the entire internal network layout.

---

## 8. OPSEC for DNS Reconnaissance

- [ ] **Rate Limiting**: When performing active brute-force, use a delay (`-D` in dnsrecon, `-t` for threads) to avoid flooding the NS and triggering DDoS protection (Cloudflare will block you).
- [ ] **Resolvers**: Use custom, high-availability resolvers (e.g., `1.1.1.1`, `8.8.8.8`) rather than targeting the target's NS for every query—this often bypasses NS logging.
- [ ] **VPN/Tor**: Use a VPN for active enumeration. Passive enumeration (crt.sh, SecurityTrails) is safe via the clearnet.
- [ ] **False Positives**: Cloud services (AWS) recycle IPs often. Always correlate the PTR record and current HTTP banner to confirm the target actually owns the IP right now.
- [ ] **Wildcard DNS**: If the target uses `*.target.com` wildcards, brute-force returns false positives (every host resolves). Check for wildcard by pinging a random string (e.g., `xysd4fs.target.com`). If it resolves, remove that IP from the results or compare differences.

---

## 9. DNSSEC & Security Observations

- Check if DNSSEC is enabled (`dig target.com DNSKEY`). If not, the zone is vulnerable to cache poisoning (less relevant for OSINT, but good to note).
- If records are **too clean** or hidden behind a proxy, the target has good network hygiene—focus more on historical leaks (crt.sh) for subdomains that may not be fully protected.

---

## 10. Quick Commands Reference Sheet

| Task | Command |
| :--- | :--- |
| **Get all common records** | `dig target.com ANY` *(Note: often disabled)* |
| **Check specific NS** | `dig @8.8.8.8 target.com A` |
| **Subdomain Brute Force** | `dnsrecon -d target.com -D wordlist.txt -t brt` |
| **Zone Transfer test** | `dig @ns.target.com target.com AXFR` |
| **Massive Subdomain via crt.sh** | `curl -s "https://crt.sh/?q=%25.target.com" \| grep -oP "(?<=\<TD\>)[^<]+" \| grep -v "target.com" \| sort -u` |
| **Reverse PTR** | `dig -x IP_ADDRESS` |
| **Bulk resolve** | `cat subdomains.txt \| while read d; do host $d; done` |

---

> **Final Warning:** DNS recon is the **noisiest** part of OSINT if done actively. A single aggressive brute-force against a Cloudflare-protected site will lead to a CAPTCHA or IP ban. Stick to passive sources for 80% of your results, and use active enumeration only for the final 20% validation. Know the laws—probing DNS in some countries is considered a precursor to hacking.
