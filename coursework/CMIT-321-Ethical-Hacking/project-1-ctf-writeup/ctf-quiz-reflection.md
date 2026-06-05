# In-House CTF Quiz - Reflection

Course: CMIT 321 Ethical Hacking
Project: Project 1 - Capture the Flag Write-Up
Author: Eric Young
Completed: June 2026

---

**Important:** This file does not contain the quiz questions or the answers themselves. The CTF quiz bank is UMGC course material and is not redistributed. This is a personal reflection on the topics and skills tested, organized to show where my knowledge is strong and where I need to continue developing.

All 10 challenges were answered correctly (100% score, Attempt 14).

---

## Reflection on the 10 Challenge Topics

### Challenge 1: HTTP client tool identification from user-agent strings

**Topic area:** Web log analysis; HTTP User-Agent header interpretation

**First-try result:** Incorrect on initial attempts (format ambiguity). Correct after reading the raw field value carefully.

**Reflection:** I knew that HTTP tools embed version information in the User-Agent header, but I underestimated how important it is to copy the exact string format rather than reformatting it. The lesson here is that log fields must be read precisely - approximating a version number is not the same as reporting it accurately.

**Concept to revisit:** Standard User-Agent string formats for common command-line tools (curl, Wget, Nikto). Know what each looks like so you can recognize it on sight in a log.

---

### Challenge 2: Web vulnerability scanner identification

**Topic area:** Web scanning tool fingerprinting; IDS/IPS evasion detection

**First-try result:** Correct.

**Reflection:** Nikto's user-agent signature is distinctive and well-known. Recognizing scanning tools from their traffic signatures is a core SOC skill. This challenge reinforced that scanning activity is always logged, even when the attacker does not attempt to disguise the tool.

**Concept to revisit:** Other common web scanning tools and their user-agent signatures (Burp Suite, Nessus, OpenVAS). Build a mental library of what scanning traffic looks like.

---

### Challenge 3: File download tool identification

**Topic area:** Web log analysis; download utility fingerprinting

**First-try result:** Correct.

**Reflection:** The same pattern as challenge 1 - different tool, same approach. Repetition built confidence in the user-agent extraction workflow. By the third challenge, the SPL pattern was automatic.

**Concept to revisit:** The significance of seeing automated download tools in web logs. Wget and curl requests are not inherently malicious, but they can indicate reconnaissance, data exfiltration, or automated attack tooling.

---

### Challenge 4: Raw string occurrence counting

**Topic area:** SPL data manipulation; eval and stats command chaining

**First-try result:** Incorrect on first attempt (used wrong SPL approach). Correct after learning the split/mvcount pattern.

**Reflection:** This was the first challenge that required a technique I did not already know. My initial approach (stats count) counted events, not occurrences within events. Understanding the difference between counting events and counting string occurrences inside events was a significant conceptual shift. The split/mvcount pattern is now part of my toolkit.

**Concept to revisit:** The difference between event-level counting (stats count) and intra-event occurrence counting (eval + split + mvcount). Know when each approach is appropriate.

---

### Challenge 5: IP address frequency analysis

**Topic area:** Network log analysis; IP address tracking

**First-try result:** Correct (applied pattern from challenge 4).

**Reflection:** Applying the same pattern to a different target string was straightforward once I understood the approach from challenge 4. The cross-validation with PowerShell gave me confidence in the result before submitting.

**Concept to revisit:** Alternative approaches for IP-based counting in IIS logs - the native c_ip field and stats count by c_ip provide the same result with less complexity. Knowing multiple paths to the same answer is valuable.

---

### Challenge 6: FTP status code analysis

**Topic area:** FTP protocol; server response codes

**First-try result:** Correct.

**Reflection:** FTP status code 331 ("Password required, specify the password") appears once for every username submission in a login sequence. A high count of 331 responses is a reliable indicator of a brute force or credential stuffing attack. This challenge connected protocol knowledge to detection logic in a concrete way.

**Concept to revisit:** The full set of FTP response codes and their security implications. 230 (login success), 331 (password required), 530 (login failed) are the most security-relevant. Know what each code tells you about attacker behavior.

---

### Challenge 7: FTP attack timeline reconstruction

**Topic area:** Log chronology; attack sequence analysis; false positive avoidance

**First-try result:** Incorrect on first attempt (submitted the wrong 230 event). Correct after distinguishing attacker from anonymous traffic.

**Reflection:** This was the most analytically challenging challenge in the category. The log contained normal activity alongside the attack, and the attack's successful event looked identical to normal successful events when viewed in isolation. Context - the pattern of 530 failures preceding the 230 - was the only way to distinguish the attack. This mirrors real SOC work more closely than any other challenge: alerts must be evaluated in context, not in isolation.

**Concept to revisit:** False positive reduction strategies. Understand what constitutes a baseline for a given service (anonymous FTP logins are normal; administrator account brute force is not). Practice writing detection rules that account for normal baseline activity.

---

### Challenge 8: Geographic attribution via GeoIP enrichment

**Topic area:** Threat intelligence; IP geolocation; iplocation command

**First-try result:** Correct.

**Reflection:** The iplocation command abstracts away the GeoIP database complexity. From an analyst perspective, the skill is knowing to run it and how to interpret the Country field. The more important underlying skill is understanding that IP geolocation is probabilistic, not definitive - the country tells you where the IP is registered, not necessarily where the attacker is physically located (VPNs and Tor complicate this).

**Concept to revisit:** Limitations of IP geolocation for attribution. Supplement with threat intelligence feeds for higher-confidence attribution. Know when to escalate geographic findings and when to treat them as weak signals.

---

### Challenge 9: SMTP authentication log analysis

**Topic area:** Email server logs; SMTP protocol; authentication attempt detection

**First-try result:** Incorrect on first three attempts (sourcetype misconfiguration). Correct after resolving ingestion issue.

**Reflection:** Most of the difficulty here was a configuration problem, not an analysis problem. The wrong sourcetype produced 2 events from a file with hundreds of thousands of lines, and no SPL query was going to find anything useful in 2 events. The lesson: always validate ingestion before assuming the data is queryable. Check event counts, check that the data looks right in _raw, and only then run analysis queries.

**Concept to revisit:** SMTP response codes and what they reveal about email-based attacks. 220 (service ready), 235 (authentication successful), 535 (authentication failed). These are the SMTP equivalents of FTP's 230/530 sequence.

---

### Challenge 10: SMTP success event timestamp extraction

**Topic area:** Custom log parsing; regex field extraction; timestamp format awareness

**First-try result:** Incorrect on first attempt (timestamp format error). Correct after reading the format from _raw.

**Reflection:** The answer checker rejected a reformatted timestamp but accepted the exact string from the log. This enforced the habit of copying field values directly from raw log data rather than interpreting or reformatting them. In incident reporting, timestamp accuracy is critical - even a formatting difference can create ambiguity in a timeline.

**Concept to revisit:** The rex command and named capture group syntax. Practice writing regex patterns for common log timestamp formats. Also review Splunk's TIME_FORMAT settings for understanding how the platform interprets timestamps from different log sources.

---

## Overall Takeaways

**Strongest areas:**

Log pattern recognition and user-agent analysis (challenges 1-3) came naturally. Protocol knowledge for FTP and SMTP (challenges 6-7, 9-10) was solid after some review. Geographic analysis (challenge 8) was straightforward using Splunk's built-in tools.

**Weakest areas:**

SPL technique selection (challenge 4) was my biggest gap going in. Knowing individual commands but not knowing which combination to apply to a given problem slowed me down. Sourcetype validation (challenge 9) is now a reflex, but it wasn't before this project.

**Most important lesson:**

Context matters more than any single data point. The correct answer to challenge 7 required understanding the surrounding events, not just finding the first matching event. That is the difference between a false positive and a true positive, and it is the core analytical skill for SOC work.

**Action items for continued development:**

Practice Splunk Boss of the SOC (BOTS) challenges to build SPL speed and analytical pattern recognition. Review FTP and SMTP protocol documentation to strengthen protocol-level detection knowledge. Study the MITRE ATT&CK techniques for brute force (T1110) and valid accounts (T1078) to connect CTF observations to real threat frameworks.

---

*No flag values, quiz answer keys, or copyrighted UMGC course content are stored in this file.*
