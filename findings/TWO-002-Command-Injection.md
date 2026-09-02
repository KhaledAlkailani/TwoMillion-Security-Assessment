# TWO-002 — OS Command Injection

## Finding Overview

| Field                  | Details                           |
| ---------------------- | --------------------------------- |
| **Finding ID**         | TWO-002                           |
| **Title**              | OS Command Injection              |
| **Severity**           | Critical                          |
| **CWE**                | CWE-78                            |
| **Affected Component** | Administrative VPN Generation     |
| **Endpoint**           | `POST /api/v1/admin/vpn/generate` |
| **Authentication**     | Administrative access required    |
| **Impact**             | Remote Code Execution             |
| **Status**             | Confirmed                         |

---

## Description

The administrative VPN generation functionality accepts a user-controlled
`username` parameter and passes the value into a shell command.

The relevant application logic was identified as:

```php
if ($user != null) {
    exec("/bin/bash /var/www/html/VPN/gen.sh $user", $output, $return_var);
}
```

The application constructs a shell command by directly concatenating
untrusted input into the command string.

This violates the separation between **data** and **command syntax** and
creates an OS command injection vulnerability.

MITRE classifies OS Command Injection as CWE-78, where externally
controlled input can influence the commands executed by the operating
system.

---

## Affected Endpoint

```text
POST /api/v1/admin/vpn/generate
```

The endpoint accepts a parameter conceptually represented as:

```json
{
    "username": "<USER_CONTROLLED_VALUE>"
}
```

---

## Vulnerable Code

```php
if ($user != null) {
    exec("/bin/bash /var/www/html/VPN/gen.sh $user", $output, $return_var);
}
```

The `$user` variable is incorporated directly into a command executed
through `/bin/bash`.

---

## Technical Analysis

The intended execution flow is:

```text
gen.sh <username>
```

However, the application does not safely treat `<username>` as a data
value.

Instead, the value becomes part of a shell command.

This means shell metacharacters and command syntax can potentially alter
the intended execution flow.

The core vulnerability is therefore not simply "bad input validation."

The actual root cause is:

> **Untrusted input is incorporated into a shell command without safely
> separating the input from command syntax.**

---

## Attack Flow

```text
┌──────────────────────┐
│ Authenticated User   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Administrative Access│
└──────────┬───────────┘
           │
           ▼
┌────────────────────────────┐
│ /admin/vpn/generate        │
└──────────┬─────────────────┘
           │
           ▼
┌────────────────────────────┐
│ User-Controlled username   │
└──────────┬─────────────────┘
           │
           ▼
┌────────────────────────────┐
│ Shell Command Construction │
└──────────┬─────────────────┘
           │
           ▼
┌────────────────────────────┐
│ OS Command Injection       │
└──────────┬─────────────────┘
           │
           ▼
┌────────────────────────────┐
│ Remote Code Execution      │
└────────────────────────────┘
```

---

## Security Impact

Successful exploitation can result in arbitrary operating-system command
execution with the privileges of the web application process.

Potential consequences include:

* Remote Code Execution
* Remote shell access
* Sensitive file access
* Credential exposure
* Data modification
* Application compromise
* Lateral movement
* Full host compromise

In the assessed attack chain, this vulnerability was used to transition
from administrative application access to operating-system access.

---

## Root Cause

The root cause is unsafe construction of an OS command using untrusted
input.

The application invokes:

```text
/bin/bash
```

and constructs the command as a string.

This creates a shell interpretation boundary that the supplied input can
potentially influence.

---

## Evidence

Supporting evidence:

```text
evidence/
└── 05-command-injection/
    ├── vulnerable-code.png
    └── command-execution.png
```

Exploit payloads, IP addresses and session information have been omitted
or redacted from the public report where appropriate.

---

## Remediation

### 1. Avoid Shell Invocation

The preferred solution is to remove shell-based execution whenever
possible.

### 2. Separate Arguments From Commands

If an external process must be executed, use an execution mechanism that
passes the executable and arguments separately rather than constructing a
shell command string.

### 3. Strict Input Validation

If the expected value is a username, define exactly what a valid username
looks like.

For example:

```text
Allowed:
    letters
    numbers
    underscore
    hyphen
```

Reject values outside the expected format.

Input validation should be considered defense-in-depth rather than the
primary protection against command injection.

### 4. Least Privilege

The web application should run with the minimum operating-system
privileges required to perform its function.

### 5. Process Isolation

Consider additional containment mechanisms such as:

* AppArmor
* Container isolation
* Restricted service accounts
* Filesystem permissions
* Network segmentation

---

## Recommended Architecture

Instead of:

```text
User Input
    ↓
String Concatenation
    ↓
/bin/bash -c "..."
```

use:

```text
Validated Input
      ↓
Explicit Argument
      ↓
Process Execution API
      ↓
VPN Generator
```

---

## Verification

After remediation, attempts to inject command syntax through the
`username` parameter should be treated as invalid input and must not
result in additional operating-system commands being executed.

Expected behavior:

```text
Malicious Input
      ↓
Input Rejected
      ↓
No Command Execution
```

---

## References

* MITRE CWE-78 — OS Command Injection
