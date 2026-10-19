# Penetration Testing Report — Footprinting & Reconnaissance Phase

**Mediroza General Hospital** · Batch B082 · Week 4
**Target:** `https://medirozahospital.com`
**Client:** Mediroza General Hospital
**Engagement Type:** Black-box Penetration Test
**Authorization:** Written permission granted (educational engagement — Networkwalks)
**Report Phase:** Milestone 1 — Reconnaissance & Footprinting (pre-exploitation)

---

## Table of Contents

1. [Liability Disclaimer](#1-liability-disclaimer)
2. [Introduction](#2-introduction)
3. [Tools Used](#3-tools-used)
4. [Activities Performed](#4-activities-performed)
5. [Risk Analysis / Impact](#5-risk-analysis--impact)
6. [Recommendations](#6-recommendations)
7. [Conclusion](#7-conclusion)
8. [Evidence Collected](#8-evidence-collected)

---

## 1. Liability Disclaimer

This report documents activities performed against `medirozahospital.com`, a target authorised for security testing as part of a controlled Networkwalks educational engagement. All reconnaissance described below was conducted only against this designated target, with written permission already secured.

These materials are for education and training purposes only. The techniques described must never be applied to any system without explicit written permission from the system owner. Unauthorised access or scanning of systems is a criminal offence in most jurisdictions, even where no damage occurs.

---

## 2. Introduction

This report covers the footprinting and reconnaissance phase (Milestone 1) of a black-box penetration test against `medirozahospital.com`, conducted as part of the Networkwalks internship programme. The objective of this phase was to gather as much publicly available information as possible about the target — its domain registration, DNS and mail infrastructure, hosting environment, web server technology, and any exposed files or directories — before any exploitation was attempted.

All commands were executed from a Kali Linux environment. Every step below includes the exact command used, the result observed, supporting evidence in the [Evidence](#8-evidence-collected) section, and a short note on why the finding is relevant from an attacker's perspective.

---

## 3. Tools Used

| Tool | Purpose |
|---|---|
| **WHOIS** | Identify domain registrar, registration/expiry dates, name servers and registrant privacy status. |
| **nslookup** (A / NS / MX / TXT) | Resolve the domain's IP address and enumerate name servers, mail servers and SPF/TXT records. |
| **dnsrecon** | Automated DNS enumeration — SOA, A, TXT (DMARC), and SRV records. |
| **Google Dorking** | Search engine queries (`site:`, `inurl:`) to check for publicly indexed sensitive paths. |
| **Nmap** | Full TCP port scan with service/version detection (`-sV -sC -p-`) against the target host. |
| **curl -I** | Inspect raw HTTP response headers returned by the web server. |
| **WhatWeb** | Fingerprint web server software, redirects, and HTTP response characteristics. |
| **wafw00f** | Detect whether a Web Application Firewall (WAF) is protecting the site, and identify its vendor. |
| **Nikto** | Web server scanner — checks for missing security headers, SSL configuration and common misconfigurations. |
| **Gobuster** | Directory/file brute-forcing to discover hidden paths and sensitive files on the web server. |
| **ffuf** | Fast web fuzzer, used as a second directory-discovery pass to cross-check Gobuster's results. |
| **theHarvester** | OSINT aggregation — emails, subdomains, hosts and IP addresses linked to the target domain. |

---

## 4. Activities Performed

### 4.1 WHOIS Lookup

`whois medirozahospital.com` was run to identify domain registration details. The domain is registered via **NameCheap, Inc.** (IANA ID 1068), created on **2026-08-14** with an expiry date of **2027-08-14**. Authoritative name servers are `dns1.namecheaphosting.com` and `dns2.namecheaphosting.com`. The registrant, admin and technical contact details are all redacted behind NameCheap's "Withheld for Privacy LLC" proxy service, based in Lewes, Delaware — this limits the OSINT value of registrant information but confirms the hosting/registrar relationship.

### 4.2 DNS Resolution (nslookup)

`nslookup` resolved `medirozahospital.com` to **199.188.201.16**. Targeted record-type lookups were then run:

- `nslookup -type=NS` → `dns1.namecheaphosting.com`, `dns2.namecheaphosting.com`
- `nslookup -type=MX` → three mail exchangers on `jellyfish.systems` (mx1, priority 5; mx2, priority 10; mx3, priority 20)
- `nslookup -type=TXT` → SPF record: `v=spf1 +a +mx +ip4:199.188.201.7 +ip4:199.188.201.38 include:spf.web-hosting.com ~all`

### 4.3 DNS Enumeration (dnsrecon)

`dnsrecon -d medirozahospital.com` performed general DNS enumeration, returning SOA, A and TXT (DMARC: `v=DMARC1; p=none;`) records, and discovering four SRV records for CalDAV/CardDAV services (`_caldav._tcp`, `_caldavs._tcp`, `_carddav._tcp`, `_carddavs._tcp`), all pointing to 199.188.201.16 on ports 2079/2080. No DNSSEC response was returned, indicating the domain is not DNSSEC-signed. The presence of calendar/contact (CalDAV/CardDAV) service records suggests the hosting platform runs a broader collaboration suite beyond just the website.

### 4.4 Google Dorking

A targeted search (`site:medirozahospital.com inurl:admin`) was used to check for any publicly indexed administration pages. No indexed admin pages were returned, which is a positive finding — it does not mean no admin interface exists, only that none is currently indexed by Google.

### 4.5 Port and Service Scanning (Nmap)

`nmap -sV -sC -p- medirozahospital.com -oA nmap-full` performed a full TCP port scan with service and script detection. Five ports were found open:

- **25** (SMTP, tcpwrapped)
- **80** (HTTP — openresty/1.31.1.1)
- **443** (HTTPS — TLS certificate CN=medirozahospital.com, SAN covering www.medirozahospital.com, valid Aug 2026–Feb 2027)
- **2082** and **2095** (both tcpwrapped, commonly associated with cPanel/webmail services on shared hosting)

Reverse DNS for 199.188.201.16 returned `server274-1.web-hosting.com`, confirming the site sits on shared hosting infrastructure rather than a dedicated server.

### 4.6 HTTP Header Inspection (curl)

`curl -I medirozahospital.com` returned `HTTP/1.1 200 OK` with `Server: openresty/1.31.1.1`, along with `Cache-Control: private, no-cache, no-store, must-revalidate, max-age=0` and a non-standard `cf-edge-cache: no-cache` header — the latter hints at a Cloudflare-adjacent edge caching layer in front of the origin.

### 4.7 Technology Fingerprinting (WhatWeb)

`whatweb medirozahospital.com` confirmed the site issues a 301 redirect from HTTP to HTTPS, and that the live site is fronted by a **LiteSpeed** server with a `PoweredBy: LiteSpeed` header and a distinctive `x-turbo-charged-by` uncommon header (a LiteSpeed cache signature). The root path and www subdomain both returned 403 Forbidden to direct requests, and the site was geolocated to the United States.

### 4.8 WAF Detection (wafw00f)

`wafw00f https://www.medirozahospital.com` identified that the site is protected by a **LiteSpeed Web Application Firewall** (LiteSpeed Technologies). An initial run against the bare domain (without www) timed out and reported the site as appearing down, which is itself a useful data point — it suggests the WAF/edge layer treats the www and non-www hosts differently, or rate-limits unrecognised request patterns.

### 4.9 Web Server Scanning (Nikto)

`nikto -h https://medirozahospital.com` confirmed the TLS certificate details, found no accessible CGI directories, and flagged that the server's own `Server` header discloses an internal-looking IP reference. It also reported three missing recommended security headers: `Permissions-Policy`, `Strict-Transport-Security` and `Content-Security-Policy`.

### 4.10 Directory and File Enumeration (Gobuster & ffuf)

`gobuster dir -u https://medirozahospital.com -w /usr/share/wordlists/dirb/common.txt` was run to brute-force common directories and files. Several sensitive paths returned **403 Forbidden** rather than 404, confirming their presence while blocking direct access:

- `/.htpasswd`
- `/.htaccess`
- `/.git/HEAD`
- `/.svn`
- `/.svn/entries`

A number of cPanel-style user-home redirects (`~adm`, `~apache`, `~amanda`, `~admin`, `~administrator`, `~bin`, and others) were also discovered, each 301-redirecting to the main site — consistent with the shared-hosting environment identified by Nmap and reverse DNS.

A parallel `ffuf` run against the same wordlist was also attempted as a cross-check; this run returned 0 req/sec and repeated errors across all progress, most likely due to WAF-driven rate-limiting or connection throttling rather than a configuration issue, and should be re-run with an increased timeout and a lower thread count if a confirmed result is required.

### 4.11 OSINT Aggregation (theHarvester)

`theHarvester -d medirozahospital.com -l 50 -b all` was run across all available OSINT sources. Most third-party API sources (BeVigil, BufferOverun, Brave Search, Zoomeye, BuiltWith, Chaos, Fofa, HaveIBeenPwned, SecurityScorecard) failed due to missing API keys, which is expected in this lab environment and does not reflect a target weakness. The sources that did respond returned:

- **1** email address — `info@medirozahospital.com`
- **1** IP address — `199.188.201.16`
- **2** hosts — `ftp.medirozahospital.com`, `mail.medirozahospital.com`
- **0** LinkedIn-linked people
- **0** compromised-credential hits via Hudson Rock

---

## 5. Risk Analysis / Impact

Based on the information collected during this footprinting phase, the following observations were identified. These are reconnaissance-stage findings, not confirmed vulnerabilities — no exploitation was attempted during this phase. The presence of a software version, open port or DNS record does not by itself confirm that a system is vulnerable; it indicates areas that merit further, authorised testing (covered in later milestones of this engagement).

| # | Finding | Evidence / Observation | Potential Impact | Risk |
|---|---|---|---|---|
| 1 | DNS infrastructure fully enumerable | dnsrecon and nslookup returned NS, MX, SOA, SPF/TXT, DMARC and SRV (CalDAV/CardDAV) records without restriction. | Builds a detailed map of mail, calendar and DNS infrastructure that can be used to plan further attacks (e.g. mail server targeting, SPF/DMARC abuse). | 🟠 Medium |
| 2 | Web server and hosting stack disclosed | curl and WhatWeb identified openresty/1.31.1.1 and LiteSpeed server headers; WhatWeb also flagged a non-standard `x-turbo-charged-by` header. | Version and stack disclosure allows an attacker to search for known vulnerabilities affecting that specific software. | 🟠 Medium |
| 3 | cPanel-related management ports open | Nmap (`-p-`) identified ports 2082 and 2095 open in addition to 25, 80 and 443. | These ports are commonly associated with cPanel/webmail services; if reachable and misconfigured, they widen the attack surface beyond the main website. | 🟠 Medium |
| 4 | Internal hosting hostname exposed via reverse DNS | Nmap's reverse DNS lookup returned `server274-1.web-hosting.com` for 199.188.201.16. | Reveals shared-hosting infrastructure details that can help identify the hosting provider and correlate with other tenants. | 🟢 Low |
| 5 | WAF technology identifiable | wafw00f identified the site as protected by a LiteSpeed WAF. | Knowing the WAF vendor lets an attacker tailor payloads or evasion techniques to that specific product. | 🟢 Low |
| 6 | Version-control and credential-related paths present | Gobuster identified `/.git/HEAD`, `/.svn`, `/.svn/entries`, `/.htpasswd` and `/.htaccess` on the server (all returned 403, i.e. present but currently blocked). | If permissions are ever relaxed or misconfigured, these paths could expose source history or credential files. | 🟠 Medium |
| 7 | Missing recommended HTTP security headers | Nikto reported the absence of `Strict-Transport-Security`, `Content-Security-Policy` and `Permissions-Policy` headers. | Weakens defence-in-depth and increases exposure to downgrade, clickjacking and injection-based attacks. | 🟢 Low |
| 8 | Public email and subdomains enumerable | theHarvester identified `info@medirozahospital.com` and the hosts `ftp.medirozahospital.com` and `mail.medirozahospital.com`. | Provides targets for phishing/social-engineering and additional attack surface beyond the main domain. | 🟢 Low |

**Risk level key:** 🔴 Critical · 🟠 Medium · 🟢 Low

---

## 6. Recommendations

- **Restrict or monitor access to cPanel/webmail management ports (2082, 2095)** — ensure they are not reachable from the public internet unless required, and are protected by IP allow-listing or a VPN.
- **Harden DNS exposure** — review whether CalDAV/CardDAV SRV records and full SPF/DMARC details need to remain publicly queryable at their current verbosity, and consider enabling DNSSEC.
- **Suppress or genericise version-disclosing headers** (`Server`, `x-turbo-charged-by`) at the web server/CDN layer to reduce fingerprinting value for attackers.
- **Add the missing security headers** identified by Nikto: `Strict-Transport-Security`, `Content-Security-Policy` and `Permissions-Policy`.
- **Remove version-control and credential artifacts** (`.git`, `.svn`, `.htpasswd`, `.htaccess`) from the production web root entirely, rather than relying on a 403 response, which still confirms their presence.
- **Review and restrict cPanel-style username enumeration** via redirected `~user` paths, as these can help an attacker build a list of valid system usernames.
- **Periodically review WAF logs** for anomalous or high-volume fuzzing traffic (as generated by tools like Gobuster/ffuf during this assessment) to validate detection and alerting coverage.
- **Continue performing authorised, scoped reconnaissance and penetration testing** on a regular basis to catch new exposure before it is found externally.

---

## 7. Conclusion

This footprinting and reconnaissance phase against `medirozahospital.com` combined DNS, WHOIS, web-technology, WAF, and directory-enumeration tooling to build a detailed picture of the target's external attack surface without performing any exploitation. The engagement confirmed a shared-hosting environment behind a LiteSpeed WAF, an openresty/LiteSpeed web stack, exposed DNS and mail infrastructure, several blocked-but-present sensitive files, and a handful of cPanel-related management ports. None of these findings are vulnerabilities in isolation, but together they meaningfully narrow the attack surface and inform the next phase of the engagement, in which the `/patient/` portal discovered via `robots.txt` was targeted directly for authentication bypass and data extraction.

---

## 8. Evidence Collected

**Figure 1** — `whois medirozahospital.com`: registrar, registration/expiry dates, name servers.
![WHOIS registration details](evidence/fig01-whois-registration.png)

**Figure 2** — WHOIS registrant/admin contact details, redacted via privacy proxy.
![WHOIS registrant details](evidence/fig02-whois-registrant.png)

**Figure 3** — `nslookup medirozahospital.com`: A record resolution to 199.188.201.16.
![nslookup A record](evidence/fig03-nslookup-a-record.png)

**Figure 4** — `nslookup -type=MX`: mail exchanger records on jellyfish.systems.
![nslookup MX record](evidence/fig04-nslookup-mx.png)

**Figure 5** — `nslookup -type=NS`: authoritative name servers.
![nslookup NS record](evidence/fig05-nslookup-ns.png)

**Figure 6** — `nslookup -type=TXT`: SPF record.
![nslookup TXT record](evidence/fig06-nslookup-txt.png)

**Figure 7** — `dnsrecon` full DNS enumeration output, including SRV records.
![dnsrecon output](evidence/fig07-dnsrecon.png)

**Figure 8** — Google dork `site:medirozahospital.com inurl:admin`: no indexed admin pages.
![Google dork results](evidence/fig08-google-dork.png)

**Figure 9** — `nmap -sV -sC -p-` full port scan results.
![Nmap scan results](evidence/fig09-nmap-scan.png)

**Figure 10** — `curl -I` HTTP response headers.
![curl headers](evidence/fig10-curl-headers.png)

**Figure 11** — WhatWeb technology fingerprint output.
![WhatWeb output](evidence/fig11-whatweb.png)

**Figure 12** — wafw00f WAF detection: LiteSpeed WAF identified.
![wafw00f output](evidence/fig12-wafw00f.png)

**Figure 13** — Nikto scan output: SSL info and missing security headers.
![Nikto output](evidence/fig13-nikto.png)

**Figure 14** — Gobuster directory enumeration: sensitive files and user-home redirects.
![Gobuster output](evidence/fig14-gobuster.png)

**Figure 15** — ffuf fuzzing run (incomplete / rate-limited).
![ffuf output](evidence/fig15-ffuf.png)

**Figure 16** — theHarvester OSINT run, part 1.
![theHarvester run](evidence/fig16-theharvester-run.png)

**Figure 17** — theHarvester OSINT summary: emails, IPs and hosts found.
![theHarvester summary](evidence/fig17-theharvester-summary.png)

---

## Report Information

| Field | Value |
|---|---|
| Program | Cybersecurity Internship Programme — Networkwalks |
| Batch | B082 \| Week 4 |
| Phase | Milestone 1 — Footprinting & Reconnaissance |
| Status | Complete — proceeding to Milestone 4 (full penetration testing report) |
