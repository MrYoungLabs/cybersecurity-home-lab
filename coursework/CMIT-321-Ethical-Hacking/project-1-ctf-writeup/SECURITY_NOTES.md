# Security Notes: CTF Category 4 - Log Analysis

Course: CMIT 321 Ethical Hacking
Project: Project 1 - Capture the Flag Write-Up
Author: Eric Young
Last updated: June 2026

---

## Purpose of This File

This file records security observations, lessons learned, and skill gaps identified during the Category 4 CTF challenges. It is a working document for self-improvement, not a formal report. The goal is to track where my knowledge is strong, where it is weak, and what I am doing about it.

---

## Lessons Learned

### Lesson 1: Sourcetype selection is a security monitoring blind spot

The most operationally significant lesson from this project was how a misconfigured sourcetype creates a monitoring gap. Uploading an SMTP log with the wrong sourcetype (misc or misc_text) collapsed hundreds of thousands of log lines into 2 events. Any SPL query run against that data would return false results. In a production SOC, this would mean an entire log source appears to be ingested but is actually unreadable - a silent blind spot.

**Implication for career:** Real SOC work involves onboarding new log sources constantly. Every new data source requires validating that the parser is correctly configured and that event counts match expectations. This lesson made the validation step feel real, not academic.

### Lesson 2: Normal events and attack events coexist in the same log

In Challenge 07 (FTP brute force), the log contained normal anonymous FTP sessions mixed in with the brute force attack. The first successful login (FTP 230) in the log was a normal session, not the attack. Submitting the wrong timestamp was an easy mistake to make without reading the context around the event.

**Implication for career:** Alert triage in a SOC requires understanding what normal looks like on a given system. A 230 FTP response is not inherently an attack signal. The attack was identifiable only because of the sequence: many 530 failures for a specific username, followed by a 230. Context is everything in log analysis.

### Lesson 3: Raw timestamps must be copied exactly

Challenge 10 required submitting a timestamp in HH:MM:SS.mmm format with a dot before milliseconds. The SMTP log used that exact format. Reformatting it (using a colon instead of a dot) was rejected. The correct approach is to copy the value directly from _raw and not reformat.

**Implication for career:** When writing incident reports or escalation summaries, timestamps must be accurate and unambiguous. Altering timestamp format without noting the change introduces ambiguity and can affect incident timeline accuracy.

### Lesson 4: SPL is learned best through problem-driven practice

Going into this project, I knew individual SPL commands from reading documentation, but could not connect them into functional queries efficiently. The learning happened through solving specific problems - Challenge 04 forced me to learn split() and mvcount(), Challenge 10 forced me to learn rex field extraction with named capture groups.

**Implication for career:** Splunk proficiency is best built through regular hands-on use, not classroom study. Volunteering for SIEM-related tasks in any role - even Tier 1 alert triage - will accelerate SPL skill development faster than studying in isolation.

---

## Skill Assessment

### Strong areas

**Log pattern recognition.** After working through 10 challenges in the same category, I developed an intuitive sense for what each log type looks like and how to navigate it. I can identify IIS W3C format, FTP status codes, and SMTP response codes without needing to look them up.

**SPL string operations.** The split(), mvcount(), and rex commands are now part of my usable toolkit. I can write occurrence-count queries and regex field extractions reliably.

**Sourcetype-aware ingestion.** I know the difference between how IIS logs and SMTP logs need to be configured in Splunk and why the difference matters.

**Cross-tool validation.** Using PowerShell to independently verify Splunk results built confidence in my answers and is a transferable habit for any data analysis task.

### Weak areas

**SPL command chaining under time pressure.** I can write correct queries given enough time, but building complex multi-pipe queries quickly is still slow. In a real SOC, analysts are expected to triage dozens of alerts per shift. Speed matters.

**Splunk native field extraction awareness.** Early in the project I wrote raw-text searches for things that ms:iis:auto had already extracted as named fields. I was building the harder query when an easier one already worked. The correct starting point is always: what fields does this sourcetype provide?

**Regex pattern writing.** The rex command requires knowing regular expression syntax. I can write basic patterns for standard formats (IP addresses, timestamps, version strings), but complex regex for irregular log formats still requires significant trial and error.

**SMTP log format familiarity.** IIS log formats are well-documented and widely used. SMTP logs were less familiar, and I spent extra time figuring out the structure before I could query them effectively.

---

## Improvement Plan

### Short-term (next 30 days)

Spend 15-20 minutes per day on Splunk Boss of the SOC (BOTS) challenges. These are purpose-built Splunk datasets that require realistic SPL queries to answer investigative questions. This directly addresses the "slow SPL command chaining" weakness by building query muscle memory through repetition.

Complete the free Splunk Fundamentals 1 course available through Splunk Training. The course reinforces sourcetype configuration, field extraction, and the full SPL pipeline in a structured way.

Practice the rex command specifically - write 5 different regex patterns per week for different log formats to build regex fluency.

### Medium-term (next semester)

Apply log analysis skills to home lab projects. Set up a local SIEM (Splunk free license supports up to 500MB/day) and ingest real logs from a home network or a VM running web services. Seeing real log data - not CTF data - accelerates pattern recognition for what normal looks like.

Study the MITRE ATT&CK framework techniques that map to log analysis: T1110 (brute force), T1078 (valid accounts), T1071 (application layer protocol). Knowing the TTP names gives me vocabulary to describe what I found in incident notes.

### Long-term (entry-level SOC role readiness)

Obtain CompTIA Security+ certification. The log analysis and incident response domains on Security+ align directly with what this CTF category tested.

Practice writing clear, concise incident notes. Log analysis skill is only useful if the findings can be communicated. Writing short summaries of "what I found, when, from which IP, by what method" is a skill to build alongside the technical work.

---

## SOC Career Connections

| CTF skill practiced | Real SOC application |
|---|---|
| User-agent extraction from IIS logs | Identifying scanning tools and anomalous clients in web access logs |
| FTP brute force detection (530 > 230 sequence) | Credential attack detection; password spray identification |
| SMTP authentication analysis | Email-based attack investigation; business email compromise triage |
| GeoIP enrichment with iplocation | Geographic context for alert triage; geo-blocking policy enforcement |
| Timeline reconstruction | Incident chronology for escalation reports and post-incident reviews |
| Sourcetype validation | Data onboarding quality assurance for new SIEM log sources |
| Cross-tool validation with PowerShell | Defensive verification: confirming a finding before escalating |
| Regex field extraction | Parsing custom log formats that SIEM does not auto-recognize |

---

## References

[1] Splunk Inc., "Splunk Enterprise Documentation," Splunk, 2024. [Online]. Available: https://docs.splunk.com/Documentation/Splunk

[2] MITRE Corporation, "MITRE ATT&CK Framework," MITRE, 2024. [Online]. Available: https://attack.mitre.org

[3] Splunk Inc., "Boss of the SOC (BOTS) Dataset," Splunk, 2024. [Online]. Available: https://github.com/splunk/botsv3

[4] MaxMind, "GeoLite2 Free Geolocation Data," MaxMind, 2024. [Online]. Available: https://dev.maxmind.com/geoip/geolite2-free-geolocation-data

---

*No flag values, quiz answer keys, or copyrighted UMGC course content are stored in this file.*
