# Asterion Group Internal Archive Breach — HR-017

## Overview

**Challenge:** Asterion Group Internal Archive Breach (HR-017)
**Category:** Web Application Security / Multi-Stage Attack
**Environment:** Authorized TCQ 2026 challenge environment

### Vulnerability / Technique Chain

```text
SQL Injection
      ↓
Authentication Bypass
      ↓
Account Enumeration
      ↓
Unauthorized Account Access
      ↓
Sensitive Information Discovery
      ↓
Burp Suite Traffic Analysis
      ↓
Information Disclosure
      ↓
Base64 Decoding
      ↓
Credential / Passphrase Recovery
      ↓
Steganographic Extraction
      ↓
Audit Record Recovery
```

---

# 1. Objective

The challenge required investigating an internal security case and recovering information hidden across multiple stages of the application.

Unlike a single-vulnerability challenge, the solution required chaining several weaknesses together.

---

# 2. Initial Attack Surface

The first identified attack surface was the application's login functionality.

The authentication mechanism accepted user-controlled input that was incorporated into a database query.

This created an opportunity to test the login endpoint for SQL injection.

---

# 3. SQL Injection

Testing revealed that the login functionality was susceptible to SQL injection.

The successful technique involved manipulating the query logic to bypass the intended authentication condition and use record positioning to enumerate accounts.

The original POC used a payload containing:

```text
' OR 1=1 LIMIT 1 OFFSET [VALUE] --
```

The exact competition payload/value is intentionally omitted from this public version.

The POC records that this technique successfully reached the dashboard associated with a suspended employee account.

---

# 4. Authentication Bypass

The SQL injection changed the behavior of the application's authentication query.

Conceptually:

```text
Normal Login
     ↓
Username + Password
     ↓
Database Query
     ↓
Authentication Decision
```

The injected input altered this flow:

```text
Attacker Input
     ↓
SQL Query Manipulation
     ↓
Authentication Condition Bypassed
     ↓
Account Returned
     ↓
Dashboard Access
```

---

# 5. Account Enumeration

The SQL injection technique was then used with record-offset manipulation.

Conceptually:

```text
Record 0
   ↓
Record 1
   ↓
Record 2
   ↓
Record 3
   ↓
...
```

Changing the `OFFSET` value allowed the challenge records to be investigated systematically.

The successful attack reached the account associated with the challenge's target suspended employee.

---

# 6. HR-017 Investigation

After reaching the dashboard, the next stage was application-level investigation.

The security case **HR-017** contained information about retained evidence.

One referenced artifact was:

```text
london.jpeg
```

The case notes indicated that the file size was inconsistent with the visible image data.

This was an important clue that the file could contain additional hidden information.

---

# 7. Burp Suite Traffic Interception

At this stage, **Burp Suite** was configured as an HTTP proxy.

The purpose was to inspect the application's requests and responses rather than relying solely on what was rendered in the browser.

### Investigation process

```text
Browser
   ↓
Burp Suite Proxy
   ↓
HTTP Request
   ↓
Challenge Application
   ↓
HTTP Response
   ↓
Burp Response Analysis
```

The raw HTML response was inspected.

A hidden developer comment was identified in the response.

The comment contained a Base64-encoded value described by the challenge as an evidence recovery key.

---

# 8. Information Disclosure

The discovery of sensitive information in an HTML developer comment represented an information-disclosure weakness.

Although HTML comments are not normally displayed as visible page content, they are still delivered to the client.

Therefore:

```text
Hidden From User Interface
        ≠
Hidden From Attacker
```

Anything delivered to a browser can potentially be inspected by the client.

This is why passwords, API keys, recovery keys, internal credentials, and other secrets should never be embedded in client-accessible source code.

---

# 9. Base64 Analysis

The discovered value was Base64 encoded.

Base64 is an encoding mechanism, not encryption.

The value was decoded using a standard Linux utility.

### Sanitized command

```bash
echo "[BASE64_VALUE_REDACTED]" | base64 -d
```

The decoded result produced a passphrase that was required for the next stage.

The original POC records the decoded value, but that secret is intentionally redacted from this public repository.

---

# 10. Steganographic Investigation

The next clue was the `london.jpeg` evidence file.

Because the challenge indicated that the file contained more data than was visually apparent, steganography was investigated.

The `steghide` utility was used to attempt extraction.

### Sanitized command

```bash
steghide extract -sf london.jpeg -p "[PASSPHRASE_REDACTED]"
```

The extraction successfully produced an audit-record file.

---

# 11. Audit Record

The extracted file contained information required for the final challenge objective.

The investigation process was:

```text
london.jpeg
     ↓
Steganographic Analysis
     ↓
Passphrase
     ↓
Hidden Data Extraction
     ↓
audit_record.txt
     ↓
Attack-Time Information
```

The original POC used a command equivalent to:

```bash
cat audit_record.txt
```

to inspect the extracted information.

Competition-specific values have been removed from this public write-up.

---

# 12. Complete Attack Flow

The final attack chain can be summarized as:

```text
┌──────────────────────┐
│ Login Functionality  │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ SQL Injection        │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ Authentication       │
│ Bypass               │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ Account Enumeration  │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ Dashboard Access     │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ Investigate HR-017   │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ Identify london.jpeg │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ Burp Suite Analysis  │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ Hidden HTML Comment  │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ Base64 Decoding      │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ Recover Passphrase   │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ steghide Extraction  │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ Audit Record         │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ Challenge Objective  │
└──────────────────────┘
```

---

# 13. Tools Used

| Tool           | Purpose                                 |
| -------------- | --------------------------------------- |
| Burp Suite     | HTTP interception and response analysis |
| Browser        | Application interaction                 |
| Linux CLI      | Security investigation                  |
| SQL Injection  | Authentication/query manipulation       |
| Base64 utility | Encoded-data decoding                   |
| steghide       | Hidden-data extraction                  |

---

# 14. Security Impact

This challenge demonstrated how multiple weaknesses can be chained into a larger compromise.

### SQL Injection

Potential consequences include:

* Authentication bypass
* Account enumeration
* Unauthorized database access
* Data disclosure
* Query manipulation

### Sensitive Information in HTML

Potential consequences include:

* Credential disclosure
* Recovery-key exposure
* Internal information leakage
* Further compromise through exposed secrets

### Steganographic Data

Sensitive information hidden inside files should not be treated as secure merely because it is not visually apparent.

---

# 15. Recommended Remediation

## SQL Injection

Use:

* Parameterized queries
* Prepared statements
* Proper input handling
* Least-privilege database accounts

Never construct SQL queries by directly concatenating untrusted input.

---

## Authentication

Authentication should:

* Use secure server-side validation
* Avoid exposing account-enumeration behavior
* Implement consistent authentication responses
* Apply appropriate rate limiting

---

## Sensitive Information Disclosure

Never place credentials, passwords, recovery keys, API keys, or other secrets inside:

```text
HTML comments
JavaScript
Client-side configuration
Source code
Browser-accessible responses
```

Secrets should remain server-side.

---

## File Security

Applications handling uploaded or retained evidence should:

* Validate file types
* Analyze suspicious file structures
* Apply appropriate access controls
* Avoid exposing sensitive artifacts unnecessarily
* Treat hidden data as potentially sensitive

---

# 16. Key Takeaways

The biggest lesson from this challenge was the importance of **attack chaining**.

An attacker does not necessarily need one catastrophic vulnerability.

Instead:

```text
SQL Injection
      +
Authentication Weakness
      +
Information Disclosure
      +
Exposed Secret
      +
Hidden File Data
      =
Complete Attack Chain
```

The challenge strengthened my practical understanding of moving from an initial web vulnerability toward deeper application and evidence analysis.

---

## Disclaimer

All testing documented here was performed within an authorized cybersecurity challenge environment.

The public version intentionally redacts:

* Challenge flags
* Credentials
* Recovery passphrases
* Live challenge infrastructure details
* Other competition-sensitive information

The purpose of this write-up is to demonstrate security methodology and technical learning without exposing sensitive competition data.
