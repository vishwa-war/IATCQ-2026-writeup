# Spectral_Override — Prototype Pollution

## Overview

**Challenge:** Spectral_Override
**Category:** Web Application Security
**Vulnerability:** Prototype Pollution
**Impact:** Authorization / privilege-state manipulation
**Environment:** Authorized TCQ 2026 challenge environment

---

## 1. Objective

The objective was to analyze the application's configuration update functionality, identify a security weakness, exploit it, and reach the protected functionality provided by the challenge.

During testing, the configuration mechanism was found to accept a user-controlled property path and value.

This behavior allowed manipulation of the JavaScript prototype chain.

---

# 2. Initial Analysis

The application exposed functionality that accepted two important pieces of information:

```text
Property path
Value
```

The property path was processed dynamically by the application.

This immediately made prototype-chain traversal an interesting attack surface because JavaScript objects inherit properties through their prototypes.

The security-relevant prototype chain was:

```text
Object
  ↓
prototype
```

The testing focused on whether attacker-controlled input could traverse into this prototype.

---

# 3. Vulnerability Identification

The critical behavior was discovered by testing whether special JavaScript object properties could be supplied as path components.

The successful traversal was:

```text
constructor → prototype
```

This provided access to the relevant prototype object.

The next step was determining whether a security-sensitive property could be introduced through that prototype.

The property selected by the challenge logic was:

```text
isAdmin
```

---

# 4. Exploitation

The successful challenge payload used an array-based property path.

### Payload

```json
{
  "path": ["constructor", "prototype", "isAdmin"],
  "value": true
}
```

This caused the relevant prototype property to be modified:

```text
isAdmin = true
```

The original POC confirms that this modification changed the application's administrative state.

---

# 5. Attack Process

The complete exploitation sequence was:

```text
┌─────────────────────────────┐
│ Access Challenge Interface  │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ Identify Configuration      │
│ Update Functionality        │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ Analyze User-Controlled     │
│ Property Path               │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ Test Prototype Traversal    │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ constructor → prototype     │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ Set isAdmin = true          │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ Authorization State Changed │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ Query Vault                 │
└──────────────┬──────────────┘
               ↓
        Challenge Objective
```

No terminal commands were required for this exploitation path; the payload was submitted directly through the challenge interface.

---

# 6. Why the Exploit Worked

The underlying issue was unsafe handling of attacker-controlled property paths.

Instead of restricting property traversal to legitimate application-owned properties, the application allowed traversal through special JavaScript object properties.

Conceptually:

```text
Attacker Input
      ↓
Dynamic Property Traversal
      ↓
constructor
      ↓
prototype
      ↓
Shared Prototype
      ↓
isAdmin = true
```

Because prototype properties can be inherited by objects, modifying a prototype can influence how those objects behave.

In this challenge, the application relied on the `isAdmin` property for authorization-related behavior, making the prototype modification security-sensitive.

---

# 7. Impact

Prototype pollution can have different impacts depending on how polluted properties are subsequently used.

Potential impacts include:

* Authorization bypass
* Privilege escalation
* Application logic manipulation
* Denial of service
* Security-control bypass
* Chaining with other vulnerabilities

In this challenge, the vulnerability resulted in modification of the application's administrative state and enabled access to the protected Query Vault functionality.

---

# 8. Recommended Remediation

Applications accepting user-controlled property paths should avoid blindly traversing or assigning arbitrary object properties.

Recommended controls include:

### Input Allowlisting

Only permit explicitly expected property names.

### Block Prototype-Related Properties

Reject dangerous traversal keys such as:

```text
__proto__
constructor
prototype
```

### Safe Object Handling

Use safer object-manipulation patterns and avoid dynamically assigning properties based directly on untrusted paths.

### Authorization Validation

Do not rely solely on inherited JavaScript properties for security decisions.

Security-sensitive authorization state should be explicitly validated.

---

# 9. Key Learning

This challenge demonstrated how a seemingly simple configuration-update feature can become an authorization vulnerability when it allows uncontrolled object-property traversal.

The important lesson was:

```text
Untrusted Property Path
        +
Dynamic Object Assignment
        +
Prototype Chain
        =
Potential Prototype Pollution
```

---

## Disclaimer

This exploitation was performed only against an authorized cybersecurity challenge environment.

The public version intentionally excludes challenge flags and other sensitive competition information.
