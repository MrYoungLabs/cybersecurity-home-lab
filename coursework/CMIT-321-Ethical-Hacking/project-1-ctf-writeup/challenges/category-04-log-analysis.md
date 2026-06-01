# Category 04 — Log Analysis

> **CTF Course:** CMIT 321 Ethical Hacking — Project 1 Write-Up  
> **Category:** Category 4: Log Analysis  
> **Status:** 🔄 In Progress — write-up being developed  
> **Last updated:** June 2026

---

## Why I chose Log Analysis

Log Analysis is the single most operationally relevant CTF category for the career path I am targeting — entry-level SOC Analyst and IT Help Desk. In a real SOC environment, Tier 1 analysts spend the majority of their shift reviewing logs inside a SIEM (Security Information and Event Management) platform such as Splunk, Microsoft Sentinel, or IBM QRadar. Every alert they triage is ultimately a log entry. Choosing this category is a direct investment in job-ready skills rather than purely competitive CTF skills.

Additionally, the analytical reasoning required to find an anomaly in a log file — identifying the one suspicious event among thousands of normal ones — is a pattern-recognition skill that transfers directly to real-world threat detection.

---

## Metadata

| Field | Value |
|---|---|
| Source / Platform | UMGC CMIT 321 in-house CTF lab |
| Category | Log Analysis |
| Difficulty (self-rated) | Medium |
| Status | In progress |
| SOC Relevance | High — core Tier 1 analyst daily task |

---

## Background: What is Log Analysis in a CTF context?

In a CTF Log Analysis challenge, the competitor is given one or more log files — typically server access logs, Windows Event Logs, authentication logs (auth.log / secure), firewall logs, or web application logs — and must find a hidden flag or answer a series of investigative questions by examining the log data.

The challenge mirrors a real SOC task: something happened on a system, and the logs are the only record. The analyst must reconstruct what happened, by whom, when, and how.

Common log types encountered:

- **Apache / Nginx access logs** — HTTP request records showing IP, method, URI, status code, and user-agent
- **Windows Event Logs** — XML-structured records of system, security, and application events (Event IDs matter: 4624 = logon, 4625 = failed logon, 4688 = process creation, etc.)
- **auth.log / /var/log/secure** — Linux authentication events including SSH login attempts, sudo usage, and PAM events
- **Firewall / IDS logs** — connection records with source/destination IP, port, protocol, and allow/deny actions
- **Syslog** — general-purpose Linux system event log

---

## Initial observations

> *This section will be updated as challenge work progresses. Generic methodology is documented now; specific observations will be added during active challenge work.*

When first opening a log file challenge, the initial recon steps are:

1. **Determine log type** — identify the format (Apache CLF, W3C extended, JSON, Windows EVTX, syslog, etc.) before attempting to parse anything.
2. **Check file size and time range** — a large log file spanning many hours signals a needle-in-a-haystack search; a short log may have a more obvious anomaly.
3. **Look for structural outliers** — lines that are longer than normal, contain unusual characters, have unexpected HTTP status codes (403, 500 bursts), or show timestamps that cluster unusually.
4. **Identify unique values** — run frequency analysis on IP addresses, usernames, user-agents, or URIs to find outliers that appear only once or an unusual number of times.

---

## Tools and methods

| Tool | Purpose | Platform |
|---|---|---|
| `grep` | Filter log lines by keyword, IP, or pattern | Linux terminal |
| `awk` | Field extraction and arithmetic on log columns | Linux terminal |
| `sort` \| `uniq -c` | Frequency analysis — count occurrences of unique values | Linux terminal |
| `cut` | Extract specific fields from delimited log lines | Linux terminal |
| `cat` \| `less` | Initial inspection of log file contents | Linux terminal |
| CyberChef | Decode encoded strings found in log URIs (Base64, URL encoding) | Browser |
| Python (pandas) | Structured analysis of large log files; pivot tables by IP/time | Primary laptop |
| Splunk Free / SIEM | Parse and search structured log data with SPL queries | Lab environment |
| Windows Event Viewer | Navigate and filter Windows EVTX log files | Windows host |
| `jq` | Parse JSON-formatted logs | Linux terminal |

### Key command patterns for log analysis

```bash
# Count unique IP addresses and rank by frequency
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -20

# Find all HTTP 4xx/5xx errors
grep -E ' [45][0-9]{2} ' access.log

# Extract all POST requests (often used in attacks)
grep 'POST' access.log

# Find lines with suspicious URI encoding
grep -i '%2e%2e\|../' access.log

# Search auth.log for failed SSH logins
grep 'Failed password' /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn

# Look for rapid successive login attempts (brute force indicator)
grep 'Failed password' auth.log | awk '{print $1,$2,$3}' | uniq -c | sort -rn
```

---

## Investigative approach

### Step 1 — Identify the log type and scope

Before searching for anything, determine what kind of log you are working with. Look at the first few lines to identify the format. Check the timestamp range to understand the time window of the data.

### Step 2 — Frequency analysis on key fields

Run `sort | uniq -c | sort -rn` on IP addresses, usernames, or endpoints. In a real attack or a CTF log, the attacker's activity will often stand out either as *unusually frequent* (brute force, scanning) or *unusually rare* (one-time exfiltration, single successful login after many failures).

### Step 3 — Look for known attack signatures in logs

Common patterns that appear in CTF log challenges and real incidents:

- **Brute force attempt followed by success** — many failed logins from one IP, then one successful login
- **Directory traversal** — URIs containing `../` or URL-encoded equivalents (`%2e%2e%2f`)
- **Web shell upload followed by execution** — POST to an upload endpoint, then GET to an unusual PHP/ASPX path
- **Unusual user-agent strings** — automated scanners (sqlmap, nikto, nmap NSE scripts) leave distinctive signatures
- **Privilege escalation evidence** — in Windows Event Logs, look for Event ID 4672 (special privileges assigned) after a logon event

### Step 4 — Correlate timestamps

Attacks are rarely isolated events. Once a suspicious IP or username is found, filter all log entries associated with that identifier and sort by timestamp to reconstruct the attacker's timeline.

### Step 5 — Decode encoded values

Attackers and CTF challenge designers often hide flags or IOCs (Indicators of Compromise) inside encoded strings in URIs, user-agent fields, or POST bodies. Run suspected encoded strings through CyberChef to check for Base64, URL encoding, or hex encoding.

---

## Knowledge gaps to address

> *This section will be updated with specific difficulties encountered during the actual challenge. The following are anticipated knowledge gaps based on the category type.*

- **Windows Event Log navigation** — EVTX files require either Windows Event Viewer or a dedicated parser (e.g., `python-evtx`, Chainsaw). Learning the most important Event IDs (logon, process creation, privilege use) is a separate study investment.
- **SIEM query language** — while `grep`/`awk` work for small files, real SOC log analysis uses SPL (Splunk), KQL (Microsoft Sentinel), or Lucene (Elastic). Translating command-line analysis into SIEM queries is a skill gap to address.
- **Volume vs. signal** — in large log files, identifying the meaningful event requires both pattern recognition and patience. This skill is built through repetition.

---

## Resources and references

| Resource | Relevance |
|---|---|
| [SANS Log Management Cheat Sheet](https://www.sans.org/posters/log-management-cheat-sheet/) | Reference for log types, retention, and analysis strategy |
| [Windows Security Event Log Encyclopedia](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/) | Lookup reference for Windows Event IDs |
| [Apache Log Format Documentation](https://httpd.apache.org/docs/current/logs.html) | Official reference for Apache Combined Log Format field definitions |
| [CyberChef](https://gchq.github.io/CyberChef/) | Browser-based tool for decoding encoded strings found in log entries |
| [TryHackMe — Introduction to SIEM](https://tryhackme.com/module/introduction-to-siem) | Hands-on SIEM log analysis practice directly applicable to SOC roles |

---

## SOC career connection

Log Analysis is the most directly transferable CTF skill to a Tier 1 SOC Analyst role. The table below maps CTF skills from this category to real SOC job tasks:

| CTF skill practiced | Real SOC application |
|---|---|
| Finding anomalous IP in access log | Alert triage — identify source IP of suspicious activity in SIEM |
| Reconstructing attacker timeline from timestamps | Incident chronology for escalation report |
| Identifying brute force pattern in auth.log | Detecting password spray / brute force alerts |
| Decoding Base64 in URI | Investigating web-based malware delivery or C2 beaconing |
| Correlating multiple log sources | Multi-source correlation in SIEM for lateral movement detection |
| Filtering noise to find the meaningful event | Reducing false positives during alert triage |

---

## Lessons learned / takeaways

> *This section will be updated with specific lessons after completing the challenge.*

Log analysis is a skill built on two foundations: knowing what *normal* looks like, and knowing what *attack patterns* look like. A CTF Log Analysis challenge compresses both into a single exercise. The most important habit to build is structured, systematic investigation — frequency analysis first, then timeline reconstruction, then decoding — rather than reading through logs line by line hoping to spot something.

For a future SOC role, the most valuable follow-on investment from this category is learning at least one SIEM query language (SPL for Splunk or KQL for Microsoft Sentinel), since enterprise log analysis at scale happens inside a SIEM, not in a terminal window.

---

*No flag values, quiz answer keys, or copyrighted UMGC course content are stored in this file. This write-up documents method, tooling, and reasoning only.*
