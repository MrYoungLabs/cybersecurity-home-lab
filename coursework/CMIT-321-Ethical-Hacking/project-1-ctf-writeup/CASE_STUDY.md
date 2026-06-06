# Case Study: UMGC CTF Category 4 - Log Analysis

Course: CMIT 321 Ethical Hacking
Project: Project 1 - Capture the Flag Write-Up
Category: Category 4 - Log Analysis
Author: Eric Young
Status: Complete - all 10 challenges solved and verified (100% score)
Last updated: June 2026

---

## Overview

This case study documents how 10 CTF log analysis challenges were solved using Splunk Enterprise. Each challenge required uploading a specific log file, configuring the correct Splunk sourcetype, writing an SPL query to locate the answer, and verifying with the UMGC answer checker.

No flag values or quiz answers are included. This document describes the approach and the SPL patterns used for each challenge.

---

## Scenario Context

The 10 challenges in Category 4 represent a simulated attack scenario reconstructed from log data. Across the 10 log files, the scenario involves:

- A web server receiving traffic from standard browsers and scanning tools (curl, Nikto, Wget)
- An FTP server targeted by a brute force attack against the administrator account
- An SMTP server targeted by an authentication attempt

The challenges move progressively from simple user-agent extraction (identifying tools) to more complex tasks like timeline reconstruction (finding the exact time of a successful brute force login) and SMTP authentication analysis.

---

## Challenge Summaries

### Challenge 01: Tool Identification - curl

**Log type:** IIS access log
**Objective:** Identify the version of the curl HTTP client recorded in the log.

**Approach:** The IIS access log records the HTTP User-Agent header for every request. Searching for events where the user-agent field contained "curl" and then reading the field value directly revealed the version string. Sourcetype ms:iis:auto automatically extracts the cs_User_Agent field.

**SPL pattern:**
```
source="Challenge 01.log" sourcetype=ms:iis:auto
| search cs_User_Agent="curl*"
| table cs_User_Agent
| dedup cs_User_Agent
```

**Answer format:** Version string in X.XX.X format.

---

### Challenge 02: Tool Identification - Nikto

**Log type:** IIS access log
**Objective:** Identify the version of the Nikto web vulnerability scanner recorded in the log.

**Approach:** Nikto embeds its version number in the User-Agent string it sends with HTTP requests. Searching for "Nikto" in the user-agent field and reading the full string gives both the tool name and version.

**SPL pattern:**
```
source="Challenge 02.log" sourcetype=ms:iis:auto
| search cs_User_Agent="*Nikto*"
| table cs_User_Agent
| dedup cs_User_Agent
```

**Why this matters in a SOC context:** Nikto is a well-known web scanning tool. Identifying Nikto user-agents in an IIS access log is a signal that someone is actively probing the web server for vulnerabilities. This would generate a high-priority alert in a production SIEM.

---

### Challenge 03: Tool Identification - Wget

**Log type:** IIS access log
**Objective:** Identify the version of the Wget download utility recorded in the log.

**Approach:** Identical pattern to challenges 01 and 02. Wget also includes its version in the User-Agent header. Searching for "Wget" in cs_User_Agent and reading the field value produces the answer.

**SPL pattern:**
```
source="Challenge 03.log" sourcetype=ms:iis:auto
| search cs_User_Agent="*Wget*"
| table cs_User_Agent
| dedup cs_User_Agent
```

---

### Challenge 04: Occurrence Count - "Mozilla"

**Log type:** IIS access log
**Objective:** Count the total number of times the string "Mozilla" appears in the log file.

**Approach:** A field-count approach (stats count by cs_User_Agent) would only count events where Mozilla is the user-agent, not total string occurrences. The correct approach is to count occurrences of the string within each event's raw text, then sum across all events.

The split-on-raw pattern divides each event by the target string and counts the resulting pieces. Subtracting 1 from the piece count gives the number of occurrences. Stats sum aggregates across all events.

**SPL pattern:**
```
source="Challenge 04.log" sourcetype=ms:iis:auto
| eval count = mvcount(split(_raw,"Mozilla"))-1
| stats sum(count) as total
```

**Cross-validation:** PowerShell Select-String -AllMatches was used to independently verify the count directly from the raw file.

---

### Challenge 05: Occurrence Count - IP Address

**Log type:** IIS access log
**Objective:** Count how many times a specific IP address appears in the log.

**Approach:** Same split-on-raw pattern as Challenge 04, applied to an IP address string. An alternative approach using stats count by c_ip and reading the specific row also works since IIS logs have a native client IP field.

**SPL pattern:**
```
source="Challenge 05.log" sourcetype=ms:iis:auto
| eval count = mvcount(split(_raw,"[REDACTED IP]"))-1
| stats sum(count) as total
```

---

### Challenge 06: Occurrence Count - FTP Status Code 331

**Log type:** IIS FTP log
**Objective:** Count how many times FTP status code 331 appears in the log.

**Approach:** FTP status code 331 means "Password required" - it is the server's response to a valid username submission before the password is sent. A high count of 331 responses indicates many authentication attempts, consistent with a brute force or credential stuffing attack.

The same split-on-raw pattern was used. The raw FTP log format places status codes as standalone tokens, so matches are accurate.

**SPL pattern:**
```
source="Challenge 06.log" sourcetype=ms:iis:auto
| eval count = mvcount(split(_raw,"331"))-1
| stats sum(count) as total
```

---

### Challenge 07: Timeline Reconstruction - FTP Brute Force Success Time

**Log type:** IIS FTP log
**Objective:** Identify the exact time the attacker successfully authenticated to the FTP server.

**Approach:** This challenge required distinguishing between two types of successful FTP logins in the same log:

1. Anonymous FTP logins (USER ftp, PASS ..., response 230) - These are normal and appear early in the log.
2. Administrator account login after brute force (USER administrator, many PASS attempts returning 530, then finally 230).

The FTP 230 response code means "User logged in, proceed." The attack is identifiable because it is preceded by a long series of 530 (Login incorrect) responses for the administrator username. The timestamp of the 230 at the end of the brute force sequence is the answer - not the timestamp of any earlier 230 events.

**SPL pattern:**
```
source="Challenge 07.log" sourcetype=ms:iis:auto
| table _time, _raw
| sort _time
```

Then review the output chronologically, identify where 530 responses for "administrator" begin, and find the 230 that ends the sequence.

**Key lesson:** Log analysis requires understanding normal vs. anomalous patterns. The first 230 looks like success but is a normal anonymous session. Context - the surrounding events - is what identifies the attack.

---

### Challenge 08: Geographic Attribution - Country of Origin

**Log type:** IIS log
**Objective:** Determine what country the attacker's IP address is registered to.

**Approach:** Splunk's built-in iplocation command performs a MaxMind GeoIP lookup against any IP field and appends Country, City, lat, and lon fields to each event. Running iplocation on the client IP field and grouping by Country immediately shows the geographic source of the traffic.

**SPL pattern:**
```
source="Challenge 08.log" sourcetype=ms:iis:auto
| iplocation c_ip
| stats count by c_ip, Country
| sort -count
```

**SOC relevance:** Geographic attribution is a standard step in alert triage. Many organizations have geo-blocking policies (e.g., block all inbound traffic from specific countries), and identifying the origin country informs the escalation path and contextualizes the threat.

---

### Challenge 09: SMTP Authentication - Username Identification

**Log type:** SMTP log (IIS SMTP service)
**Objective:** Identify the username used in an SMTP authentication attempt.

**Critical setup note:** SMTP logs must be uploaded with sourcetype generic_single_line using "Every Line" event breaking. Using misc or misc_text collapses the entire file into 2 events and makes searching impossible. The correct configuration produces one Splunk event per log line (hundreds of thousands of events for this file).

**Approach:** SMTP logs record AUTH command attempts. Searching for events containing "AUTH" in the raw log text surfaces the authentication attempt and the associated username.

**SPL pattern:**
```
source="Challenge 09.log" sourcetype=generic_single_line
| search _raw="*AUTH*"
| table _raw
```

---

### Challenge 10: Timeline Reconstruction - SMTP Authentication Success Time

**Log type:** SMTP log
**Objective:** Identify the exact timestamp when the attacker successfully authenticated to the SMTP server.

**Approach:** SMTP response code 235 means "Authentication successful." Searching for events containing "235" returns the successful authentication event. The rex command extracts the raw timestamp from the log line, which must be copied exactly as it appears in the log.

**SPL pattern:**
```
source="Challenge 10.log" sourcetype=generic_single_line
_raw="*235*"
| rex field=_raw "(?P<log_time>\d{2}:\d{2}:\d{2}\.\d+)"
| table _raw, log_time
```

**Critical format note:** The SMTP log uses a dot (.) before the milliseconds in the timestamp, not a colon. The answer must be submitted exactly as it appears in the raw log. Reformatting the timestamp was rejected by the answer checker.

---

## Summary Table

| Challenge | Log Type | Key SPL Technique | Verified |
|---|---|---|---|
| 01 | IIS access | User-agent field search | Yes |
| 02 | IIS access | User-agent field search | Yes |
| 03 | IIS access | User-agent field search | Yes |
| 04 | IIS access | split/_raw occurrence count | Yes |
| 05 | IIS access | split/_raw occurrence count | Yes |
| 06 | IIS FTP | split/_raw occurrence count | Yes |
| 07 | IIS FTP | Timeline sort + context analysis | Yes |
| 08 | IIS | iplocation GeoIP enrichment | Yes |
| 09 | SMTP | AUTH keyword search in _raw | Yes |
| 10 | SMTP | 235 code search + rex timestamp extraction | Yes |

---

## Sourcetype Configuration Reference

| Log Type | Sourcetype | Event Break | Notes |
|---|---|---|---|
| IIS access / FTP logs | ms:iis:auto | Default | Auto-extracts W3C format fields |
| SMTP logs | generic_single_line | Every Line | Required for per-line event breaking |

---

*No flag values, quiz answer keys, or copyrighted UMGC course content are stored in this file.*
