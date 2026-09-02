# Assessment Methodology

## Overview

This assessment followed a structured Vulnerability Assessment and Penetration Testing (VAPT) methodology.

The objective was to identify vulnerabilities, validate their security impact, determine whether individual findings could be chained together, and assess the resulting level of compromise.

The methodology was divided into the following phases:

```text
Reconnaissance
     ↓
Service Enumeration
     ↓
Web Application Enumeration
     ↓
API Discovery
     ↓
Authentication & Authorization Testing
     ↓
Exploitation
     ↓
Post-Exploitation
     ↓
Privilege Escalation
     ↓
Impact Assessment
     ↓
Remediation Recommendations
```

---

## 1. Reconnaissance

The initial phase focused on identifying exposed network services and understanding the target's attack surface.

Activities included:

* TCP port scanning
* Service detection
* Version enumeration
* Identification of externally accessible services

The objective was to determine which services could provide an entry point into the target.

---

## 2. Service Enumeration

After identifying open ports, each exposed service was examined to determine its purpose and potential attack surface.

The assessment identified:

* SSH
* HTTP

The web service was prioritized because it exposed an interactive application and additional functionality.

---

## 3. Web Application Enumeration

The web application was enumerated for:

* Accessible directories
* Hidden resources
* Application functionality
* Client-side JavaScript
* API endpoints
* Authentication mechanisms

Directory enumeration identified the `/invite` functionality.

The application initially appeared to return wildcard redirects for unknown paths. Response characteristics were therefore analyzed to distinguish genuine resources from false positives.

---

## 4. Application Logic Analysis

The `/invite` functionality was examined to understand how invitation codes were generated.

Client-side JavaScript contained obfuscated application logic.

After deobfuscation, an internal API endpoint was identified:

```http
/api/v1/invite/how/to/generate
```

The endpoint disclosed information about the application's invitation-generation process.

Further testing identified the generation endpoint:

```http
/api/v1/invite/generate
```

The resulting invite code was decoded and used to create an application account.

---

## 5. Authentication and API Enumeration

Once an authenticated account was obtained, the application's API surface was enumerated.

Identified functionality included:

```text
/api/v1
/api/v1/invite/how/to/generate
/api/v1/invite/generate
/api/v1/invite/verify
/api/v1/user/auth
/api/v1/user/vpn/generate
/api/v1/user/vpn/regenerate
/api/v1/user/vpn/download
/api/v1/user/register
/api/v1/user/login
/api/v1/admin/auth
/api/v1/admin/vpn/generate
/api/v1/admin/settings/update
```

The objective was to determine whether authorization controls were consistently enforced across authenticated and administrative functionality.

---

## 6. Authorization Testing

Administrative API endpoints were tested using the privileges of a normal authenticated user.

The following endpoint was identified as vulnerable:

```http
PUT /api/v1/admin/settings/update
```

The endpoint accepted a security-sensitive privilege attribute:

```json
{
  "is_admin": 1
}
```

The server failed to properly enforce authorization for this operation.

This resulted in administrative privilege escalation.

The finding was classified as:

**Broken Access Control / Missing Authorization — CWE-862**

---

## 7. Command Execution Testing

After obtaining administrative privileges, administrative functionality was examined for unsafe handling of user-controlled input.

The VPN generation functionality was identified as executing input through a shell command.

The relevant application logic was:

```php
exec("/bin/bash /var/www/html/VPN/gen.sh $user", $output, $return_var);
```

Because untrusted input was incorporated into a shell command, the functionality was vulnerable to OS Command Injection.

The finding was classified as:

**CWE-78 — Improper Neutralization of Special Elements used in an OS Command**

The vulnerability was validated by demonstrating command execution and obtaining a shell on the target.

---

## 8. Post-Exploitation Enumeration

Following successful command execution, limited post-exploitation enumeration was performed to understand the security impact.

Activities included:

* Identifying the current user
* Enumerating operating-system information
* Reviewing local files
* Inspecting potentially sensitive application data
* Identifying the running kernel version
* Searching for potential privilege-escalation paths

The host was identified as:

```text
Ubuntu 22.04.2 LTS
Kernel: 5.15.70-051570-generic
```

---

## 9. Privilege Escalation Assessment

The host's kernel version was researched against publicly known vulnerabilities.

The assessment identified:

**CVE-2023-0386**

The vulnerability affects OverlayFS functionality and can allow a local unprivileged user to escalate privileges.

The vulnerability was validated in the controlled assessment environment, resulting in root-level access.

---

## 10. Impact Assessment

The complete attack chain demonstrated the following progression:

```text
External Access
      ↓
Web Enumeration
      ↓
Invite Code Generation
      ↓
Account Registration
      ↓
API Enumeration
      ↓
Broken Access Control
      ↓
Administrative Privileges
      ↓
OS Command Injection
      ↓
Remote Shell
      ↓
Kernel Exploitation
      ↓
Root Access
```

This demonstrated that individually scoped weaknesses could be chained together to achieve complete compromise of the target host.

---

## 11. Evidence Collection

Evidence was collected throughout the assessment to support each finding.

Evidence was organized according to the assessment phase:

```text
evidence/
├── 01-reconnaissance/
├── 02-web-enumeration/
├── 03-api-enumeration/
├── 04-authorization/
├── 05-command-injection/
└── 06-privilege-escalation/
```

Only relevant evidence was retained to demonstrate exploitability and impact.

Sensitive information such as credentials, session tokens, private keys, and real-world identifiers should be removed or redacted before publication.

---

## 12. Reporting

Each confirmed vulnerability was documented independently with:

* Finding ID
* Severity
* CWE/CVE classification
* Affected component
* Description
* Technical details
* Impact
* Evidence
* Remediation
* References

This structure allows individual findings to be reviewed independently while preserving the complete attack-chain narrative in the main assessment report.

---

## Assessment Outcome

The assessment successfully demonstrated a complete compromise path from externally accessible application functionality to root-level access.

The primary security lesson is that effective security requires controls across multiple layers:

**Application → Authorization → Input Handling → Host Security → Patch Management**
