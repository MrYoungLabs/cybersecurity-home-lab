# Methodology: Category 4 Log Analysis CTF

Course: CMIT 321 Ethical Hacking
Project: Project 1 - Capture the Flag Write-Up
Author: Eric Young
Last updated: June 2026

---

## Overview

This document describes the methodology I used to complete all 10 challenges in CTF Category 4: Log Analysis. It covers environment setup, the repeatable workflow applied to each challenge, SPL command-building strategy, and the lessons I learned about effective log investigation in Splunk.

The goal is not to document what the answers were, but to document how I developed a repeatable, transferable approach to log analysis that applies to both CTF and real SOC work.

---

## Environment Setup

### Tool: Splunk Enterprise (desktop)

All 10 challenges were completed using Splunk Enterprise installed locally on a Windows workstation. Splunk Enterprise provides the same core search and indexing functionality as Splunk Cloud, with the advantage of offline operation and full control over data ingestion settings.

**Why Splunk Enterprise over Splunk Cloud for this lab:**

The local installation allowed unrestricted file uploads without cloud storage quotas or network latency affecting results. It also made it easier to re-upload and re-index files with different sourcetypes when initial configurations produced incorrect event counts.

**Setup steps taken:**

1. Splunk Enterprise installed at default path, accessible at http://localhost:8000
2. Default index used for all uploads (no custom index required)
3. Host field set to "ctf-lab" for all uploads to keep challenge data identifiable

---

## Repeatable Workflow

Each of the 10 challenges followed this workflow:

### Step 1: Identify the log file

Each challenge on the UMGC CTF page provides a specific log file to download. The filename, file type, and log format are all signals used to determine the correct Splunk configuration in the next step.

### Step 2: Determine the correct sourcetype

This was the most critical configuration decision for each challenge. The wrong sourcetype produces malformed events, incorrect field extraction, and inaccurate search results.

Two sourcetypes were used across all 10 challenges:

**ms:iis:auto** - Used for IIS access logs and IIS FTP logs (challenges 01 through 08). This sourcetype auto-detects IIS W3C format fields and maps them to Splunk field names including cs_User_Agent, c_ip, sc_status, cs_method, and cs_uri_stem.

**generic_single_line** - Used for SMTP logs (challenges 09 and 10). This sourcetype, configured with "Every Line" event breaking, creates one Splunk event per log line. SMTP logs are not IIS format and do not benefit from IIS field extraction. Using misc or misc_text for SMTP logs produces only 2 events for the entire file, making any search useless.

### Step 3: Upload the log file

Using Settings > Add Data > Upload in Splunk Enterprise:

- Set sourcetype as determined in Step 2
- Set host to "ctf-lab"
- Use Default index
- For generic_single_line: verify event break setting is "Every Line" before finalizing

### Step 4: Verify event count

After upload, run a basic search for the source file and check the event count. A correctly ingested IIS log of several thousand lines should produce thousands of events - not 2. If only 2 events appear, the sourcetype is wrong. Re-upload with the correct sourcetype.

Example verification query:
```
source="<filename>.log" | stats count
```

### Step 5: Identify the relevant SPL approach for the question

Before writing a query, I determined what kind of answer the question was asking for:

- **String/version extraction:** Search for a known user-agent keyword, then table the field containing the version
- **Occurrence count:** Use the split-on-raw pattern to count string occurrences across all events
- **Timestamp extraction:** Sort by time, filter by event type (success code), extract timestamp from _raw
- **GeoIP lookup:** Use the iplocation command to enrich IP fields with geographic data

### Step 6: Write and run the SPL query

Queries were built incrementally - start with a broad search to confirm data is present, then narrow down with filters, then add eval/stats/rex to extract the specific value needed.

### Step 7: Cross-validate when uncertain

For count-based questions (Q4, Q5), I cross-validated Splunk results using PowerShell's Select-String -AllMatches to count pattern occurrences directly in the raw file. When both results agreed, I was confident in the answer.

### Step 8: Verify with the UMGC answer checker

The UMGC CTF answer checker gives immediate feedback (correct/incorrect) without revealing the answer. I used this to confirm before moving to the next challenge.

---

## SPL Command-Building Strategy

My initial approach to SPL was to look up commands individually and try to construct queries from scratch. This was inefficient. The more effective approach I settled on was:

1. **Understand what fields are available first.** Run `| fields *` or `| table *` to see what field names the sourcetype has already extracted. For IIS logs, ms:iis:auto provides cs_User_Agent, c_ip, sc_status, and others that make many searches trivial.

2. **Use _raw as a fallback.** If a field is not extracted, the full original log line is always available in _raw. Commands like rex and split work on _raw when named fields are not available.

3. **Build incrementally.** Write the simplest version of the query that returns any results, then add pipes to transform the output toward the answer. Never write a 5-pipe query from scratch.

4. **The split-and-count pattern for occurrence counting:**

```
| eval count = mvcount(split(_raw,"TARGET_STRING"))-1
| stats sum(count) as total
```

This pattern works for any string that needs to be counted across an entire log file, regardless of field structure. It was used for challenges 04, 05, and 06.

5. **The rex extraction pattern for timestamp parsing:**

```
| rex field=_raw "(?P<fieldname>PATTERN)"
```

Used when Splunk does not auto-extract the value needed. For SMTP logs with custom timestamp formats, rex was the only reliable way to pull the exact timestamp string from the raw log line.

---

## Key Lessons About Sourcetype Selection

The most operationally significant lesson from this project was how much sourcetype selection matters. The same log file will produce completely different (and wrong) results depending on the sourcetype used.

For SMTP logs specifically: the file has hundreds of thousands of lines, but using misc or misc_text collapsed the entire file into 2 events because those sourcetypes use multi-line event breaking by default. Switching to generic_single_line with Every Line event breaking produced 376,000+ events - one per log line - and made all searches functional.

This lesson applies directly to real SOC work. Onboarding a new log source to a production SIEM requires the same decision: which parser or sourcetype correctly represents how this log is structured? A misconfigured sourcetype means your security monitoring has a blind spot.

---

## PowerShell as a Cross-Validation Tool

For challenges involving raw occurrence counts, PowerShell provided an independent verification method that did not rely on Splunk's parsing or field extraction:

```powershell
(Get-Content "C:\path\to\file.log" | Select-String -Pattern "TARGET" -AllMatches).Matches.Count
```

This counts every occurrence of a string pattern in a file, line by line, regardless of file format. When this count matched the Splunk result, I was confident both tools were reading the data correctly.

---

## Tools Used

| Tool | Purpose |
|---|---|
| Splunk Enterprise (local) | Primary log analysis platform - ingestion, indexing, SPL search |
| Windows PowerShell | Cross-validation of string counts; direct file analysis |
| UMGC CTF Answer Checker | Immediate feedback on submitted answers |
| MaxMind GeoIP (via Splunk iplocation) | Geographic enrichment of attacker IP addresses |

---

*No flag values, quiz answer keys, or copyrighted UMGC course content are stored in this file.*
