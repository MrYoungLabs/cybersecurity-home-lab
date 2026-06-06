# Challenge Write-Up: Category 1 - Challenge 1

Course: CMIT 321 Ethical Hacking
Project: Project 1 - Capture the Flag Write-Up
Author: Eric Young
Status: Not completed during project window - documented for learning purposes

---

## Challenge Summary

**Category:** Category 1 - Encoding and Encryption
**Challenge:** Decode a Base64-encoded string to recover the hidden value.
**Why not completed:** Time constraints combined with a Splunk tooling limitation that required a different ingestion strategy than I had prepared for.

---

## What the Challenge Asks

The challenge presents a Base64-encoded string and asks the analyst to decode it to recover the hidden flag value. Base64 is a binary-to-text encoding scheme commonly used to encode data for transport over text-based protocols such as email, HTTP headers, and JSON fields. It is not encryption - it provides no confidentiality and is trivially reversible.

---

## Why I Skipped It

My plan was to use Splunk's built-in base64decode() function to decode the string. When I went to build the query, I ran into a key limitation: **Splunk's base64decode() function requires a field name as its argument - it cannot operate on a raw string literal typed directly into a query.**

This meant that to use Splunk to decode the value, I first needed to get the encoded string into a field within an indexed event. My initial approach was to upload the data as raw text, but raw text uploads without a recognized sourcetype do not produce clean, searchable fields.

The correct workaround would have been to either:

1. Upload a minimal log file in IIS or another recognized format, with the Base64 string embedded in a field that ms:iis:auto or a similar sourcetype would parse into a named field (e.g., a custom URI or user-agent entry).
2. Use Splunk's eval to create a field from a literal string and then apply base64decode() to that field.

I did not have time to work through this ingestion problem during the project window. I redirected that time toward completing all 10 Category 4 challenges instead, which was the higher-priority deliverable.

---

## How This Challenge Would Be Solved

### Option 1: Splunk makeresults + eval approach (no log file needed)

Once I identified that eval can create a field from a literal string, the query becomes straightforward:

```
| makeresults
| eval encoded="[BASE64_STRING_HERE]"
| eval decoded=base64decode(encoded)
| table encoded, decoded
```

The makeresults command generates a single synthetic event, eval encoded= assigns the Base64 string to a field, and base64decode() decodes it. No log file upload is required.

### Option 2: Command-line decode (outside Splunk)

Base64 decoding does not require Splunk at all. Any of the following approaches work:

**Linux/macOS terminal:**
```bash
echo "BASE64_STRING" | base64 --decode
```

**PowerShell:**
```powershell
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String("BASE64_STRING"))
```

**Python:**
```python
import base64
base64.b64decode("BASE64_STRING").decode("utf-8")
```

**CyberChef:** gchq.github.io/CyberChef supports Base64 decode as a drag-and-drop recipe. Useful for quick decoding without writing code.

### Option 3: Upload a minimal IIS-format log file

If the challenge specifically requires Splunk as the tool, a minimal IIS W3C log file can be crafted with the Base64 string placed in the cs_uri_query or cs_User_Agent field. Uploading with sourcetype ms:iis:auto would parse it into a named field that base64decode() can target.

---

## Key Concept: Base64 Is Not Encryption

Base64 encoding is frequently misunderstood as a security control. It is not. Base64 encodes binary data as ASCII characters using a 64-character alphabet (A-Z, a-z, 0-9, +, /). It is designed for data transport, not confidentiality.

In a SOC context, Base64 appears in:

- HTTP Basic Authentication headers (credentials encoded as Base64 before transmission over the wire)
- SMTP AUTH commands (as seen in CTF Category 4 Challenge 9 - the username was Base64-encoded)
- Malware command-and-control communications (adversaries use Base64 to obfuscate payloads in network traffic and log entries)
- PowerShell -EncodedCommand executions (a common Windows attack technique: MITRE ATT&CK T1027)

Recognizing and decoding Base64 on sight is a fundamental SOC analyst skill.

---

## Splunk Lesson Learned

The base64decode() limitation - requiring a field rather than a literal string - is a general Splunk pattern. Most SPL eval functions operate on fields, not literals. The makeresults + eval workaround for creating a synthetic field is broadly useful any time you need to apply an SPL function to a value that is not already in an indexed event.

This same workaround applies to other eval functions: urldecode(), md5(), sha1(), and similar functions can all be tested against literal values using the makeresults pattern.

---

## Tools That Could Be Used

| Tool | Approach | Notes |
|---|---|---|
| Splunk Enterprise | makeresults + eval + base64decode() | No log file required; purely query-based |
| CyberChef | From Base64 recipe | Browser-based; supports chained decoding operations |
| PowerShell | System.Convert.FromBase64String() | Built in to Windows; no install required |
| Python | base64.b64decode() | Two lines of code; cross-platform |
| Linux base64 | base64 --decode | Single command; standard on all Linux distributions |

---

## References

[1] Splunk Inc., "base64decode() - Splunk Documentation," Splunk, 2024. [Online]. Available: https://docs.splunk.com/Documentation/Splunk/latest/SearchReference/TextFunctions

[2] GCHQ, "CyberChef - The Cyber Swiss Army Knife," GCHQ, 2024. [Online]. Available: https://gchq.github.io/CyberChef

[3] MITRE Corporation, "T1027: Obfuscated Files or Information," MITRE ATT&CK, 2024. [Online]. Available: https://attack.mitre.org/techniques/T1027/

[4] Microsoft, "Convert.FromBase64String Method," Microsoft, 2024. [Online]. Available: https://learn.microsoft.com/en-us/dotnet/api/system.convert.frombase64string

---

*No flag values, quiz answer keys, or copyrighted UMGC course content are stored in this file. This write-up documents method, tooling, and reasoning only.*
