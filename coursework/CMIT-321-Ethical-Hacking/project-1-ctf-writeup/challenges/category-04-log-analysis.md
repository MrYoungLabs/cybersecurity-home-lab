# Category 04: Log Analysis

> **CTF Course:** CMIT 321 Ethical Hacking - Project 1 Write-Up
> **Category:** Category 4: Log Analysis
> **Status:** Complete - all 10 challenges solved and verified
> **Last updated:** June 2026

---

## Why I chose Log Analysis

Log Analysis is the single most operationally relevant CTF category for the career path I am targeting: entry-level SOC Analyst and IT Help Desk. In a real SOC environment, Tier 1 analysts spend the majority of their shift reviewing logs inside a SIEM (Security Information and Event Management) platform such as Splunk, Microsoft Sentinel, or IBM QRadar. Every alert they triage is ultimately a log entry. Choosing this category is a direct investment in job-ready skills rather than purely competitive CTF skills.

Additionally, the analytical reasoning required to find an anomaly in a log file; identifying the one suspicious event among thousands of normal ones; is a pattern-recognition skill that transfers directly to real-world threat detection.

---

## Metadata

| Field | Value |
|---|---|
| Source / Platform | UMGC CMIT 321 in-house CTF lab |
| Category | Log Analysis |
| Difficulty (self-rated) | Medium |
| Status | Complete |
| Tools used | Splunk Enterprise (desktop), PowerShell |
| SOC Relevance | High - core Tier 1 analyst daily task |

---

## Background: What is Log Analysis in a CTF context?

In a CTF Log Analysis challenge, the competitor is given one or more log files: typically server access logs, Windows Event Logs, authentication logs (auth.log / secure), firewall logs, or web application logs; and must find a hidden flag or answer a series of investigative questions by examining the log data.

The challenge mirrors a real SOC task: something happened on a system, and the logs are the only record. The analyst must reconstruct what happened, by whom, when, and how.

Common log types encountered:

- Apache / Nginx access logs: HTTP request records showing IP, method, URI, status code, and user-agent
- Windows Event Logs: XML-structured records of system, security, and application events (Event IDs matter: 4624 = logon, 4625 = failed logon, 4688 = process creation, etc.)
- auth.log / /var/log/secure: Linux authentication events including SSH login attempts, sudo usage, and PAM events
- Firewall / IDS logs: connection records with source/destination IP, port, protocol, and allow/deny actions
- Syslog: general-purpose Linux system event log

---

## Tools and Approach

For this challenge set I used two primary tools: Splunk Enterprise (installed locally) and Windows PowerShell. Each challenge required uploading the log file to Splunk, configuring the correct sourcetype so events parsed correctly, and then writing an SPL query to extract the answer. For a couple of early challenges I cross-validated with PowerShell as a sanity check.

The key Splunk configuration lesson across all challenges: sourcetype selection determines how Splunk breaks the file into events. For IIS logs, the ms:iis:auto sourcetype works correctly and maps native fields automatically. For SMTP logs, the generic_single_line sourcetype (configured with "Every Line" event breaking) is required; sourcetypes like misc or misc_text collapse the entire file into just two events, making any search useless.

---

## Challenge Walkthroughs

### Challenge 01: Identify the curl version

**Log type:** IIS access log
**Objective:** Find which version of curl was used by a web client in the log.
**Approach:** Uploaded the log with sourcetype ms:iis:auto, host ctf-lab. Searched for events where the user-agent string contained "curl" and extracted the version number directly from the user-agent field. Understanding what a user-agent string represents and how web clients identify themselves in HTTP logs was essential for this challenge [5].

**Key SPL pattern:**

source="Category_04_Log_Analysis Challenge 01.log" sourcetype=ms:iis:auto
| search cs_User_Agent="curl*"
| table cs_User_Agent
| head 5

Answer format: X.XX.X (version string)
**Status:** Verified by answer checker.

---

### Challenge 02: Identify the Nikto version

**Log type:** IIS access log
**Objective:** Find which version of Nikto web scanner was used to scan this server.
**Approach:** Nikto leaves a distinctive user-agent string in IIS logs. Understanding the structure and purpose of user-agent strings in IIS W3C log format guided the search strategy [5]. Searched for the Nikto user-agent and extracted the version from the field value.

**Key SPL pattern:**

source="Category_04_Log_Analysis Challenge 02.log" sourcetype=ms:iis:auto
| search cs_User_Agent="*Nikto*"
| table cs_User_Agent
| dedup cs_User_Agent

**Status:** Verified by answer checker.

---

### Challenge 03: Identify the Wget version

**Log type:** IIS access log
**Objective:** Find the version of Wget used by a client in the log.
**Approach:** Same pattern as challenges 01 and 02; user-agent string analysis applies uniformly across curl, Nikto, and Wget [5]. Searched for Wget in the user-agent field and pulled the version string.

**Key SPL pattern:**

source="Category_04_Log_Analysis Challenge 03.log" sourcetype=ms:iis:auto
| search cs_User_Agent="*Wget*"
| table cs_User_Agent
| dedup cs_User_Agent

**Status:** Verified by answer checker.

---

### Challenge 04: Count occurrences of "Mozilla"

**Log type:** IIS access log
**Objective:** Determine how many times the word "Mozilla" appears in the log file.
**Approach:** This was a raw string count across the entire file, not a field-count. The key is using split() on _raw to split each event by the target string, then counting how many pieces result. Subtracting 1 gives the number of occurrences per event. Summing across all events gives the total. Understanding the distinction between delimiters (fixed string separators) and expressions (pattern-based matching) informed this approach [1]; regular expression syntax was also consulted to verify the split delimiter behavior [2].

**SPL query used:**

source="Category_04_Log_Analysis Challenge 04.log" sourcetype=ms:iis:auto
| eval mozilla_count = mvcount(split(_raw,"Mozilla"))-1
| stats sum(mozilla_count) as total

**PowerShell cross-validation:**

(Get-Content "the path to my download folder for the CTF file" | Select-String -Pattern "Mozilla" -AllMatches).Matches.Count

**Why mvcount(split(_raw,...)) - 1 works:** split() divides a string by a delimiter and returns a multivalue field. If "Mozilla" appears 3 times in a line, split creates 4 pieces; mvcount() returns 4; subtract 1 to get 3 actual occurrences. This pattern counts occurrences anywhere in raw text, even if the field is not natively extracted [1].

**Status:** Verified by answer checker.

---

### Challenge 05: Count occurrences of IP 192.168.1.50

**Log type:** IIS access log
**Objective:** Determine how many times the IP address 192.168.1.50 appears in the log.
**Approach:** Same split-on-raw pattern as Challenge 04, applied to an IP address string instead of a keyword [1].

**SPL query used:**

source="Category_04_Log_Analysis Challenge 05.log" sourcetype=ms:iis:auto
| eval ip_count = mvcount(split(_raw,"192.168.1.50"))-1
| stats sum(ip_count) as total

**Note on the expression-based approach:** Because IIS logs have a native c_ip field for client IP, an alternative approach is | stats count by c_ip and then reading the row for 192.168.1.50. Both produce the same result, but the eval + split approach is more portable to log types without named IP fields [1][2].

**Status:** Verified by answer checker.

---

### Challenge 06: Count occurrences of FTP status code 331

**Log type:** IIS FTP log
**Objective:** Determine how many times the FTP status code 331 appears in the log.
**Approach:** Same split-on-raw pattern applied to the string "331" [1]. Care was taken not to count partial matches (e.g., lines containing "3310" or "1331"). The raw FTP log format places the status code as a standalone token, so the count is accurate.

**SPL query used:**

source="Category_04_Log_Analysis Challenge 06.log" sourcetype=ms:iis:auto
| eval count_331 = mvcount(split(_raw,"331"))-1
| stats sum(count_331) as total

**Status:** Verified by answer checker.

---

### Challenge 07: Find the time the hacker logged in (FTP)

**Log type:** IIS FTP log
**Objective:** Identify the exact time the attacker successfully authenticated to the FTP server.
**Approach:** This required distinguishing between two types of activity in the log:

1. Anonymous FTP logins (USER ftp / PASS ... / 230) - these were legitimate background noise, not the attack.
2. Brute force attempts against the administrator account (USER administrator / PASS ... / 530 repeated, then finally 230).

Reading the IIS FTP log field layout - specifically the common fields recorded per line such as timestamp, client IP, method, and status code - was necessary to correctly interpret the log structure and isolate the attacker session [4]. The key was looking for the FTP 230 response code (successful login) after a long series of 530 (failed login) attempts for the administrator account.

**SPL query used:**

source="Category_04_Log_Analysis Challenge 07.log" sourcetype=ms:iis:auto
| table _time, _raw
| sort _time

Then filtered visually for the administrator 230 event to find the exact timestamp.

**Important distinction learned:** The first 230 event in the log is an anonymous FTP login, not the attacker. The attack starts later when brute force begins against the administrator account. Submitting the anonymous login timestamp was incorrect; the correct answer is the timestamp of the administrator's successful 230 after the brute force sequence.

**Status:** Verified by answer checker. Answer format: HH:MM:SS.

---

### Challenge 08: Identify the country of origin of the attack

**Log type:** IIS log
**Objective:** Determine what country the attacking IP address originates from.
**Approach:** Upload the log with sourcetype ms:iis:auto. Knowledge of which IIS log fields carry IP address data (specifically c-ip for client IP) informed the correct field to pass to iplocation [4]. Used Splunk's built-in iplocation command to perform a GeoIP lookup on the attacker's IP address.

**SPL query pattern:**

source="Category_04_Log_Analysis Challenge 08.log" sourcetype=ms:iis:auto
| iplocation c_ip
| stats count by c_ip, Country
| sort -count

**What iplocation does:** Splunk includes a MaxMind GeoIP database. The iplocation command takes an IP field as input and enriches each event with geographic metadata. This is a core SOC skill: mapping attacker IP addresses to geographic origin is a standard step in incident triage.

**Status:** Verified by answer checker.

---

### Challenge 09: Identify the SMTP user attempting to log in

**Log type:** SMTP log (IIS SMTP service)
**Objective:** Find the full username string that was used in the SMTP authentication attempt.
**Approach:** SMTP authentication logs record the AUTH command and the Base64-encoded credentials. The challenge required identifying the login attempt in the SMTP log. Text search was used to locate the AUTH event.

**Important upload note:** SMTP logs must be uploaded with the generic_single_line sourcetype using the "Every Line" event break setting. Using misc or misc_text collapses the entire file into 2 events; generic_single_line produces one event per log line, giving correct search results (376,000+ events for this log file).

**Key SPL pattern:**

source="Category_04_Log_Analysis Challenge 09.log" sourcetype=generic_single_line
| search _raw="*AUTH*"
| table _raw

**Status:** Verified by answer checker.

---

### Challenge 10: Find the time the hacker logged in (SMTP)

**Log type:** SMTP log
**Objective:** Identify the exact timestamp when the attacker successfully authenticated to the SMTP server.
**Approach:** SMTP response code 235 indicates successful authentication ("235 authenticated"). Searched for events containing "235", then used rex to extract the timestamp from the raw log line.

**SPL query used:**

source="Category_04_Log_Analysis Challenge 10.log" sourcetype=generic_single_line
_raw="*235*"
| rex field=_raw "(?P<log_time>d{2}:d{2}:d{2}.d+)"
| table _raw, log_time

**Answer format lesson learned:** The answer must be submitted exactly as it appears in the raw log. The SMTP log uses a dot (.) as the separator before milliseconds. Submitting with a colon before the milliseconds was rejected. Always copy the timestamp directly from _raw rather than reformatting it.

**Status:** Verified by answer checker. Answer format: HH:MM:SS.mmm (dot before milliseconds, not colon).

---

## Key SPL Patterns Learned

This challenge set introduced several SPL techniques that are directly applicable to real SOC work:

### 1. Raw string counting with split and mvcount

| eval count = mvcount(split(_raw,"TARGET_STRING"))-1
| stats sum(count) as total

Use this when you need to count occurrences of a string anywhere in an event, not just in a named field [1][2].

### 2. Regex field extraction with rex

| rex field=_raw "(?P<fieldname>PATTERN)"

Use this to extract structured data from unstructured raw text. Critical for log types where Splunk does not auto-extract the fields you need.

### 3. GeoIP enrichment with iplocation

| iplocation IPFIELD
| stats count by IPFIELD, Country

Appends geographic data to events using the built-in MaxMind database. Standard first step when investigating external IP addresses in a SOC context.

### 4. Timeline reconstruction

| table _time, _raw
| sort _time

Chronological ordering of events to reconstruct attacker timelines. Used in Q7 to trace the brute force sequence and find the successful login.

---

## Sourcetype Reference for These Challenges

| Log type | Correct sourcetype | Event break setting | Notes |
|---|---|---|---|
| IIS access logs (Q1-Q8) | ms:iis:auto | Default | Auto-extracts IIS W3C fields |
| SMTP logs (Q9-Q10) | generic_single_line | Every Line | misc / misc_text produces only 2 events - wrong |

---

## Lessons Learned

The biggest technical lesson from this category was that sourcetype configuration is not a minor detail - it determines whether your data is searchable at all. Uploading an SMTP log with the wrong sourcetype collapses 376,000 log lines into 2 unusable blobs. No SPL query will save a bad ingestion configuration.

The biggest analytical lesson was the difference between correlation and causation in log data. In Challenge 07, the first successful FTP login in the file looked like the attack; it was actually a normal anonymous FTP session. The real attack was the brute force sequence that appeared later. Correct log analysis requires understanding context: what is normal for this server, and what deviates from that baseline?

My biggest gap going into this challenge was understanding how to configure and chain SPL commands together. I could look up individual commands like rex, eval, and split, but connecting them into a cohesive query that extracted exactly what I needed required trial-and-error. I also discovered I was not effectively using Splunk's native field extraction for IIS logs; I spent time building raw-text searches when ms:iis:auto had already parsed the fields I needed. Starting each challenge by understanding what fields the sourcetype provides would have cut my solving time significantly.

---

## SOC Career Connection

| CTF skill practiced | Real SOC application |
|---|---|
| Searching for specific strings across large log files | Alert triage: finding the relevant event in a high-volume data stream |
| Counting occurrences with eval + stats | Baseline deviation detection: how many times should this normally appear? |
| Timeline reconstruction from _time | Incident chronology for escalation report writing |
| Identifying brute force in FTP auth logs | Detecting password spray / brute force alerts in SIEM |
| iplocation for GeoIP lookup | Standard first step when triaging alerts with external IP addresses |
| Distinguishing anonymous vs. authenticated login | Reducing false positives; understanding what normal activity looks like |
| Configuring sourcetypes for correct ingestion | Real-world data onboarding to SIEM: incorrect sourcetype = no searchable data |
| Extracting timestamps with rex | Parsing custom log formats that SIEM doesn't auto-recognize |

---

## References

[1] M. Jaeger, "Delimiters vs. Expressions for Log Parsing," Atera, Aug. 2024. [Online]. Available: https://www.atera.com/blog/logs/delimiters

[2] W3Schools, "Python RegEx," W3Schools. [Online]. Available: https://www.w3schools.com/python/python_regex.asp

[3] E. Beth, "Mastering Tree Structures: Binary Tree versus BST in Data Structures," May 13, 2025. [Online]. Available: https://www.geeksforgeeks.org/binary-tree-vs-binary-search-tree

[4] L. Danielson, "What Are IIS Logs?," Huntress Security 101, Oct. 3, 2025. [Online]. Available: https://www.huntress.com/security-101/topic/what-are-iis-logs

[5] M. Burgres, "What Is a User Agent?," Huntress Security 101, Jul. 29, 2025, updated Jul. 27, 2026. [Online]. Available: https://www.huntress.com/security-101/what-is-a-user-agent

[6] Splunk Inc., "SPL2 Search Reference," Splunk Documentation. [Online]. Available: https://docs.splunk.com/Documentation/Splunk/latest/SearchReference/WhatsInThisManual

[7] MaxMind, "GeoLite2 Free Geolocation Data," MaxMind. [Online]. Available: https://dev.maxmind.com/geoip/geolite2-free-geolocation-data

---

No flag values, quiz answer keys, or copyrighted UMGC course content are stored in this file. This write-up documents method, tooling, and reasoning only.
