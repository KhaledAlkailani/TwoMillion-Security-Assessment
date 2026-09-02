# TwoMillion — Vulnerability Assessment

> **Hack The Box | Linux | Web Application Security | API Security | Privilege Escalation**

![Platform](https://img.shields.io/badge/Platform-Hack%20The%20Box-9FEF00)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-blue)
![Assessment](https://img.shields.io/badge/Assessment-Vulnerability%20Assessment-red)
![Status](https://img.shields.io/badge/Status-Completed-success)

---

## Executive Summary

This assessment documents the security evaluation of **TwoMillion**, a Linux-based Hack The Box machine exposing both SSH and HTTP services.

The assessment demonstrated a complete compromise chain beginning with web application reconnaissance and ending in **root-level access**.

The primary attack path consisted of:

```text
External Reconnaissance
        │
        ▼
Web Enumeration
        │
        ▼
Hidden Invite Functionality
        │
        ▼
API Discovery
        │
        ▼
Authenticated User Access
        │
        ▼
Broken Authorization
        │
        ▼
Administrative Privilege
        │
        ▼
OS Command Injection
        │
        ▼
Remote Shell
        │
        ▼
Local Enumeration
        │
        ▼
Outdated Linux Kernel
        │
        ▼
CVE-2023-0386
        │
        ▼
ROOT
```

The assessment highlights how multiple weaknesses can be chained together to transform seemingly limited application functionality into complete system compromise.

---

## Assessment Information

| Field            | Details                                     |
| ---------------- | ------------------------------------------- |
| Target           | TwoMillion                                  |
| Platform         | Hack The Box                                |
| Operating System | Linux                                       |
| Difficulty       | Easy                                        |
| Assessment Type  | Vulnerability Assessment / Penetration Test |
| Assessment Date  | 2026-08-05                                  |
| Report Modified  | 2026-08-17                                  |
| Author           | Khaled Alkailani                            |
| Final Impact     | Full System Compromise                      |

---

# 1. Scope

The assessment focused on the externally exposed services and application functionality provided by the target.

### In Scope

* TCP/22 — SSH
* TCP/80 — HTTP
* Web application functionality
* Public and authenticated API endpoints
* Authorization controls
* Administrative functionality
* Local privilege escalation

### Objective

The primary objective was to identify vulnerabilities that could allow an attacker to:

1. Gain initial application access.
2. Escalate application privileges.
3. Execute arbitrary commands on the server.
4. Obtain a system shell.
5. Escalate privileges to root.

---

# 2. Methodology

The assessment followed a structured penetration-testing workflow:

```text
1. Reconnaissance
2. Service Enumeration
3. Web Application Enumeration
4. API Discovery
5. Authentication Testing
6. Authorization Testing
7. Input Validation Testing
8. Command Injection Testing
9. Local Enumeration
10. Vulnerability Identification
11. Privilege Escalation
12. Impact Assessment
```

The assessment emphasized identifying vulnerabilities based on their **security impact**, rather than simply collecting individual findings.

---

# 3. Timeline

| Time  | Activity                                   |
| ----- | ------------------------------------------ |
| 13:02 | Nmap reconnaissance                        |
| 13:10 | Web directory enumeration                  |
| 13:22 | API endpoint discovery                     |
| 13:40 | Invite generation functionality identified |
| 13:42 | Invite code obtained                       |
| 14:27 | Web application account registered         |
| 15:06 | Authenticated API enumeration              |
| 17:40 | Administrative privileges obtained         |
| 17:42 | Administrative functionality enumerated    |
| 18:22 | Command injection identified               |
| 18:50 | Remote shell obtained                      |
| 20:xx | Local privilege escalation analysis        |
| 21:xx | Kernel vulnerability identified            |
| 21:xx | Root access achieved                       |

---

# 4. Attack Surface Discovery

## 4.1 Nmap

Initial reconnaissance identified two exposed TCP services:

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

### Attack Surface

| Port | Service | Security Relevance                        |
| ---- | ------- | ----------------------------------------- |
| 22   | SSH     | Potential remote administration interface |
| 80   | HTTP    | Primary attack surface                    |

The HTTP service was therefore selected as the primary target for further enumeration.

---

# 5. Web Application Enumeration

Directory enumeration initially produced false positives because the application returned a wildcard response for nonexistent paths.

After filtering the common response characteristics, an interesting endpoint was identified:

```text
/invite
```

This endpoint provided functionality related to account registration.

The page source contained JavaScript responsible for communicating with the application's backend API.

The JavaScript was obfuscated, but deobfuscation revealed an API endpoint responsible for generating invite instructions:

```text
/api/v1/invite/how/to/generate
```

### Security Observation

The client-side JavaScript exposed application functionality and API routes that were not immediately visible through the primary interface.

This demonstrates the importance of treating:

* JavaScript
* API calls
* frontend source
* hidden routes

as part of the application's attack surface.

---

# 6. Invite Code Generation

The discovered API returned instructions encoded using ROT13.

The decoded instruction indicated that a POST request should be sent to:

```text
/api/v1/invite/generate
```

The endpoint returned an encoded invite code.

After decoding the Base64 value, a valid invitation code was obtained.

The invite code allowed registration of a new application account.

### Security Impact

The invite mechanism was not functioning as a meaningful access-control barrier.

Instead, the application exposed the functionality required to generate a valid invitation through publicly reachable API endpoints.

### Finding

**Weak Access Control / Predictable Invite Workflow**

**Severity:** Medium

---

# 7. Authenticated API Enumeration

After registration, the application exposed additional API functionality to authenticated users.

Enumeration revealed endpoints including:

```text
/api/v1/user/auth
/api/v1/user/vpn/generate
/api/v1/user/vpn/regenerate
/api/v1/user/vpn/download
```

More importantly, administrative functionality was also exposed:

```text
/api/v1/admin/auth
/api/v1/admin/vpn/generate
/api/v1/admin/settings/update
```

This significantly expanded the attack surface.

The next phase focused on determining whether administrative endpoints correctly enforced authorization.

---

# 8. Finding — Broken Authorization

## Finding ID

```text
TWO-001
```

## Title

**Broken Access Control Allows Privilege Escalation to Administrator**

### Severity

**High**

### Description

The administrative settings functionality failed to properly restrict modification of security-sensitive account attributes.

The application accepted user-controlled parameters including:

```json
{
  "email": "<USER>",
  "is_admin": 1
}
```

The `is_admin` attribute directly affected the privilege level of the account.

The application failed to enforce a strong server-side authorization boundary around this operation.

### Security Impact

An authenticated low-privileged user could manipulate their account state and obtain administrative privileges.

This transformed a normal authenticated account into an administrative account.

### Attack Chain

```text
Normal User
    │
    ▼
Authenticated API
    │
    ▼
/api/v1/admin/settings/update
    │
    ▼
Privilege Attribute Manipulation
    │
    ▼
is_admin = 1
    │
    ▼
Administrative Access
```

### Root Cause

The underlying issue is not simply that an `is_admin` parameter exists.

The fundamental problem is:

> **The server trusted a security-sensitive privilege attribute supplied through an operation that should have been restricted to authorized administrators.**

### Remediation

* Enforce server-side authorization on every administrative endpoint.
* Never allow ordinary users to modify authorization attributes.
* Do not accept `is_admin` from untrusted clients.
* Derive authorization state from trusted server-side data.
* Implement centralized authorization middleware.
* Apply least-privilege principles.
* Log and alert on privilege changes.

### Recommended Design

Instead of:

```text
User → update_settings(is_admin)
```

use:

```text
User
 │
 └── update own profile
       │
       └── allowed fields only

Administrator
 │
 └── privileged account-management API
       │
       └── authorization required
```

---

# 9. Finding — OS Command Injection

## Finding ID

```text
TWO-002
```

## Title

**OS Command Injection in Administrative VPN Generation**

### Severity

**Critical**

### Affected Functionality

```text
POST /api/v1/admin/vpn/generate
```

### Description

The administrative VPN generation functionality accepted a user-controlled `username` value.

The application subsequently passed this value into a shell command:

```php
if ($user != null) {
    exec("/bin/bash /var/www/html/VPN/gen.sh $user", $output, $return_var);
}
```

The critical security issue is the direct interpolation of untrusted input into a shell command.

The intended operation is conceptually:

```text
gen.sh <username>
```

However, because the input is interpreted by a shell, attacker-controlled shell metacharacters can alter the command structure.

### Root Cause

**Improper Neutralization of Special Elements used in an OS Command**

The application does not safely separate data from command syntax.

### Security Impact

Successful exploitation allows an attacker with access to the vulnerable administrative functionality to execute arbitrary operating-system commands under the privileges of the web application process.

Potential consequences include:

* Remote code execution
* Remote shell access
* Credential theft
* Application compromise
* Data disclosure
* Lateral movement
* Full host compromise

### Attack Chain

```text
Authenticated User
        │
        ▼
Broken Authorization
        │
        ▼
Administrator
        │
        ▼
VPN Generation Endpoint
        │
        ▼
Unsanitized username
        │
        ▼
Shell command execution
        │
        ▼
OS Command Injection
        │
        ▼
Remote Code Execution
```

### Evidence

The vulnerable code demonstrates the security boundary failure:

```php
exec("/bin/bash /var/www/html/VPN/gen.sh $user", ...);
```

The `$user` variable should be treated as untrusted input.

### Remediation

The preferred solution is to **avoid shell execution entirely** where possible.

If an external executable is unavoidable:

* Use a process execution API that does not invoke a shell.
* Pass arguments separately from the executable.
* Validate the username against a strict allowlist.
* Reject unexpected characters.
* Never rely solely on blacklist filtering.
* Execute the process using the lowest possible OS privileges.
* Apply filesystem and process isolation.

For example, conceptually:

```text
Executable:
    /var/www/html/VPN/gen.sh

Argument:
    validated_username
```

rather than constructing:

```text
/bin/bash /path/gen.sh <untrusted input>
```

---

# 10. Finding — Outdated Linux Kernel

## Finding ID

```text
TWO-003
```

## Title

**Vulnerable Linux Kernel Enables Local Privilege Escalation**

### Severity

**High**

### Identified Kernel

```text
Linux 5.15.70-051570-generic
```

The host reported:

```text
Ubuntu 22.04.2 LTS
```

The running kernel was sufficiently old to warrant investigation for known local privilege-escalation vulnerabilities.

---

# 11. CVE-2023-0386

The application compromise provided a local foothold on the host.

During local enumeration, an internal system message referenced an OverlayFS/FUSE kernel issue.

This led to investigation of:

```text
CVE-2023-0386
```

CVE-2023-0386 affects the Linux kernel OverlayFS subsystem and can allow a local attacker to escalate privileges.

Ubuntu rates the vulnerability **High**, with a CVSS 3 score of **7.8**. Ubuntu's security advisory indicates that Ubuntu 22.04 was fixed in kernel version `5.15.0-70.77`.

### Security Impact

A low-privileged local user can potentially escalate privileges to root on an affected system.

### Attack Chain

```text
Web Application
      │
      ▼
Command Injection
      │
      ▼
Application Shell
      │
      ▼
Local User
      │
      ▼
Vulnerable OverlayFS
      │
      ▼
CVE-2023-0386
      │
      ▼
ROOT
```

---

# 12. Final Impact

The vulnerabilities were successfully chained to achieve complete compromise of the target.

### Initial Access

```text
Public Web Application
```

### Privilege Escalation

```text
User
 ↓
Administrator
```

### Code Execution

```text
Administrator
 ↓
OS Command Injection
 ↓
System Shell
```

### Local Privilege Escalation

```text
System User
 ↓
CVE-2023-0386
 ↓
root
```

### Overall Impact

**Full system compromise**

An attacker capable of chaining these vulnerabilities could potentially gain:

* Application-level access
* Administrative application privileges
* Arbitrary command execution
* Operating-system access
* Root privileges
* Full confidentiality, integrity, and availability impact

---

# 13. Risk Summary

| ID      | Finding                         | Severity | Impact                           |
| ------- | ------------------------------- | -------- | -------------------------------- |
| TWO-001 | Broken Access Control           | High     | Application privilege escalation |
| TWO-002 | OS Command Injection            | Critical | Remote code execution            |
| TWO-003 | Outdated Kernel / CVE-2023-0386 | High     | Local privilege escalation       |

### Overall Risk

# CRITICAL

The individual weaknesses become significantly more dangerous when chained together.

A vulnerability that appears limited in isolation can become a critical security issue when it provides the next stage of an attack.

---

# 14. Remediation Priority

## Priority 1 — Eliminate Command Injection

Replace shell-based execution with safe process execution.

**Priority:** Immediate

---

## Priority 2 — Fix Authorization Controls

Administrative endpoints must perform authorization checks server-side.

Sensitive properties such as:

```text
is_admin
role
permissions
```

must never be controlled directly by ordinary users.

**Priority:** Immediate

---

## Priority 3 — Patch the Operating System

Update the Linux kernel to a supported security-patched version.

Maintain an ongoing vulnerability-management process covering:

* Kernel
* Operating system packages
* Web server
* Application dependencies
* Third-party components

**Priority:** High

---

## Priority 4 — Improve API Security

Implement:

* Centralized authorization
* Strict input validation
* Rate limiting
* Security logging
* Consistent authentication checks
* API inventory management
* Negative authorization testing

---

# 15. Lessons Learned

The most important lesson from TwoMillion was that successful compromise did not depend on a single vulnerability.

The attack was the result of **multiple security failures interacting with each other**.

```text
Hidden Functionality
        +
Weak Access Control
        +
Unsafe Command Execution
        +
Unpatched Kernel
        =
Full System Compromise
```

The assessment also demonstrated the importance of following evidence rather than assumptions.

A seemingly insignificant API endpoint exposed additional application functionality.

An application privilege parameter exposed an authorization weakness.

A server-side command execution flaw transformed application access into operating-system access.

Finally, local enumeration revealed that the underlying operating system provided another escalation path.

---

# 16. Assessment Takeaway

> **Security is rarely about finding one big vulnerability. It is about understanding how small weaknesses connect.**

TwoMillion is a strong example of an attack chain where:

```text
Reconnaissance
      ↓
Enumeration
      ↓
Discovery
      ↓
Authorization Testing
      ↓
Command Injection
      ↓
Local Enumeration
      ↓
Privilege Escalation
```

Each stage provided the information or access required for the next.

---

# 17. References

* Hack The Box — TwoMillion
* Ubuntu Security — CVE-2023-0386
* MITRE CVE Database
* Linux Kernel Security Advisories

---

## Disclaimer

This assessment was performed against an intentionally vulnerable **Hack The Box** environment for educational and authorized security-testing purposes.

The techniques and evidence presented in this repository should not be used against systems without explicit authorization.
