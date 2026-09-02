# TWO-001 — Broken Access Control

## Finding Overview

| Field                  | Details                                       |
| ---------------------- | --------------------------------------------- |
| **Finding ID**         | TWO-001                                       |
| **Title**              | Broken Access Control / Missing Authorization |
| **Severity**           | High                                          |
| **CWE**                | CWE-862                                       |
| **Affected Component** | Administrative Settings API                   |
| **Endpoint**           | `PUT /api/v1/admin/settings/update`           |
| **Authentication**     | Required                                      |
| **Impact**             | Application Privilege Escalation              |
| **Status**             | Confirmed                                     |

---

## Description

The application exposes an administrative settings endpoint that modifies
security-sensitive account attributes.

During testing, an authenticated low-privileged user was able to modify
the `is_admin` attribute of an account through the API.

The application did not enforce an adequate server-side authorization
boundary before processing this security-sensitive modification.

This allowed a normal application user to elevate their privileges to
administrator.

According to MITRE, CWE-862 occurs when a product does not perform an
appropriate authorization check before allowing an actor to perform an
action or access a resource.

---

## Affected Endpoint

```text
PUT /api/v1/admin/settings/update
```

The relevant request structure observed during testing was:

```json
{
    "email": "<USER_EMAIL>",
    "is_admin": 1
}
```

Sensitive values have been redacted.

---

## Technical Analysis

The security-sensitive parameter was:

```text
is_admin
```

This parameter determines whether the application treats the account as
an administrator.

A normal user should not be able to directly define or modify their own
authorization level.

The security boundary should instead be enforced by the server based on
the authenticated user's existing privileges.

---

## Attack Flow

```text
┌──────────────────────┐
│ Low-Privileged User  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Authenticated API    │
└──────────┬───────────┘
           │
           ▼
┌─────────────────────────────┐
│ /admin/settings/update      │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│ Modify security-sensitive   │
│ authorization attribute     │
└──────────┬──────────────────┘
           │
           ▼
┌──────────────────────┐
│ is_admin = 1         │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Administrative Access│
└──────────────────────┘
```

---

## Security Impact

Successful exploitation allows an authenticated attacker to elevate
their application privileges.

Administrative access subsequently exposes additional privileged
functionality, including the VPN generation functionality.

This vulnerability therefore acts as a critical link in the overall
attack chain.

### Potential Impact

* Unauthorized privilege escalation
* Access to administrative functionality
* Modification of account settings
* Access to privileged API endpoints
* Increased attack surface
* Potential transition to remote code execution

---

## Root Cause

The root cause is insufficient server-side authorization around a
security-sensitive operation.

The application effectively trusts a client-controlled value to influence
the user's authorization state.

Authorization decisions must be made by trusted server-side logic and
must not depend on arbitrary client-supplied privilege attributes.

---

## Evidence

Supporting evidence:

```text
evidence/
└── 04-authorization/
    └── admin-access.png
```

Sensitive session identifiers and credentials have been removed from
published evidence.

---

## Remediation

### 1. Enforce Server-Side Authorization

Administrative operations must verify that the authenticated account has
the required administrative privileges before processing the request.

### 2. Protect Security-Sensitive Attributes

Users must not be permitted to modify:

```text
is_admin
role
permissions
privileges
```

through ordinary profile-management functionality.

### 3. Implement Role-Based Access Control

Use a centralized RBAC mechanism to define which roles can access
administrative functionality.

### 4. Apply Allowlisting

Only explicitly approved fields should be accepted by user profile
update endpoints.

For example:

```text
Allowed:
    display_name
    timezone
    profile_image

Restricted:
    is_admin
    role
    permissions
```

### 5. Security Logging

Record security-sensitive events such as:

* Role changes
* Permission changes
* Administrative account creation
* Failed authorization attempts

---

## Recommended Security Model

```text
Normal User
    │
    └── Profile API
          │
          └── User-controlled profile fields only


Administrator
    │
    └── Administrative API
          │
          └── Explicit server-side authorization
                │
                └── Privileged operations
```

---

## Verification

After remediation, the following test should fail for a normal user:

```text
Normal User
     │
     ▼
Administrative Endpoint
     │
     ▼
403 Forbidden
```

The application should never rely on the client to determine whether
the requesting account is an administrator.

---

## References

* MITRE CWE-862 — Missing Authorization
