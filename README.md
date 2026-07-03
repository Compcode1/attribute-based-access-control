# Identity Architecture Lab: Attribute-Based Access Control (ABAC) Blueprint

This deployment manual serves as the definitive technical reference and architecture blueprint for implementing Attribute-Based Access Control (ABAC) frameworks within a Microsoft Entra ID (MEID) directory tenant workspace. It records completed configuration states, outlines platform boundaries, and establishes a clear verification path for identity-plane asset protection.

---

## Phase 1: Administrative Elevation (The Identity Bootstrap)

### 1. Technical Implementation Summary
Two highly specialized, data-plane administrative roles were explicitly assigned to the global administrator identity within the Microsoft Entra ID (MEID) directory tenant:
* **Attribute Definition Administrator (ADA)**
* **Attribute Assignment Administrator (AAA)**

This assignment was executed and verified within an isolated, private browser session (Incognito Window) to force an immediate session-state termination, requiring Microsoft Entra ID (MEID) to perform a new authentication handshake and issue a fresh Access Token (AT) containing these newly assigned directory role claims.

### 2. Architectural Rationale
By default, the Global Administrator role possesses no inherent read or write privileges over the Custom Security Attributes (CSA) database plane. This strict separation of duties ensures that even a compromised high-level identity cannot view or manipulate sensitive security tags without explicit, granular authorization. 

* **Failure State:** Attempting to view or modify custom attributes without active ADA or AAA directory roles causes an immediate `Access Denied` trigger due to the absence of data-plane claims within the active Access Token (AT).
* **Success State (The Inverse Logic):** Active ADA and AAA directory assignments ensure that the newly generated Access Token (AT) carries explicit schema-write claims, successfully unlocking the attribute definition and object assignment menus.

### 3. Downstream Policy Leverage
These elevated administrative privileges establish the root authority required to act as the schema architect. This identity capability is leveraged throughout the lifecycle to manage, audit, and append metadata parameters to directory objects.

---

## Phase 2: Metadata Schema Definition

### 1. Technical Implementation Summary
A custom metadata container called an Attribute Set was provisioned and named exactly `EngineeringMetadata`. Inside this structural container, two distinct security keys were defined:
1. `ClearanceLevel`: A single-value string schema constrained by pre-defined dropdown parameters (`Tier1`, `Tier2`, `Tier3`).
2. `ProjectCode`: A multi-value string array engineered to accept freeform text string inputs.

### 2. Architectural Rationale
Before user identities can be tagged with security metadata, the structural blueprint must exist within the tenant schema. A pre-defined dropdown pattern was selected for `ClearanceLevel` to eliminate typographical human errors that would break downstream authorization logic. A multi-value string array was selected for `ProjectCode` to allow a single engineering identity to hold multiple project tags concurrently.

* **Failure State:** If an administrator attempts to input an unauthorized string into a pre-defined attribute like `ClearanceLevel`, the validation engine rejects the write event, throwing a schema validation error.
* **Success State (The Inverse Logic):** Inputting an authorized, pre-defined string like `Tier3` allows the database engine to validate the string against the container rules and successfully commit the data write to the object.

### 3. Downstream Policy Leverage
These two custom attributes function as cryptographic access tokens. In downstream validation phases, access policies evaluate whether the requesting user object possesses these exact attribute values, eliminating the need to write complex, hardcoded rules tracking explicit user identities.

---

## Phase 3: Identity Profile Stamping

### 1. Technical Implementation Summary
Two distinct user identities were provisioned within the directory and stamped with explicit custom metadata pairings within the `EngineeringMetadata` attribute set:
* **Alpha Engineer (Source/Subject Identity):** Configured with `ClearanceLevel` = `Tier3` and `ProjectCode` = `ProjectQuantum`.
* **Bravo Engineer (Source/Subject Identity):** Configured with `ClearanceLevel` = `Tier1` and `ProjectCode` = `ProjectLegacy`.

### 2. Architectural Rationale
Validating an access control gate requires two completely opposing identity profiles: an authorized test subject (Alpha Engineer) and an unauthorized fallback control profile (Bravo Engineer). This deliberate segregation of profile attributes allows for systematic verification that the policy functions as an absolute binary gate.

* **Failure State:** If a user object lacks the exact required metadata combinations, the evaluation engine automatically categorizes that identity as out-of-scope for high-security resource access.
* **Success State (The Inverse Logic):** When Alpha Engineer's profile contains the exact matches (`Tier3` + `ProjectQuantum`), the authorization engine registers a positive boolean evaluation, treating those attributes as a legitimate authorization key.

### 3. Downstream Policy Leverage
These two identities act as the live test subjects for the policy gate. When they present their credentials to a targeted resource, the evaluation engine reads these hidden database strings to determine whether to authorize the session or trigger an immediate block event.

---

## Phase 4: Static Cryptographic Container Segments

### 1. Technical Implementation Summary
A traditional Microsoft Entra ID (MEID) Security Group was created with an **Assigned** membership type, named exactly `ABAC-Engineers-Tier3-Quantum`. Both **Alpha Engineer** and **Bravo Engineer** were manually assigned as active member objects within this single container.

### 2. Architectural Rationale
This step was implemented to pivot around a native platform constraint: the Microsoft Entra ID (MEID) dynamic group rule syntax engine strictly cannot read Custom Security Attributes (CSA) at the group-calculation plane. By utilizing a standard assigned group and placing both engineers inside it, a unified testing boundary was established.

* **Failure State:** Attempting to build an automated dynamic group query referencing `user.CustomSecurityAttributes` results in an immediate syntax validation failure, leaving the group deployment incomplete.
* **Success State (The Inverse Logic):** Utilizing a standard assigned group container allows the group object to successfully save and hold both engineers simultaneously, shifting the filtering logic down to the policy enforcement point.

### 3. Downstream Policy Leverage
This group acts as the macro-target for the Conditional Access (CA) policy. By targeting this group, both Alpha Engineer and Bravo Engineer are forced into the exact same security evaluation gate, allowing the policy engine to prove how it handles their differing user metadata profiles.

---

## The Forward Strategy: Identity-Plane Verification Pivot

Because a pristine test tenant without a paid, active Azure infrastructure subscription does not instantiate the underlying Azure management service principal database rows, attempting to target the Azure Service Management API inside Conditional Access will result in zero search results.

**Next Deployment Phase:** The laboratory policy will target an enterprise application guaranteed to exist natively in the sandbox environment: **Office 365** (or **Microsoft Graph Explorer**). The Conditional Access (CA) policy will target the `ABAC-Engineers-Tier3-Quantum` group, map the clearance tiers to standard directory **Extension Attributes (EA)**, and demonstrate a fully operational user attribute-filtering gate without requiring external Azure infrastructure.

---

## System Validator: Constraints & Edge Cases

### 1. Access Denied Triggers
* **Trigger 01:** An administrator possessing the Global Administrator role but lacking the Attribute Assignment Administrator (AAA) role attempts to update Alpha Engineer's attributes. This action triggers a hard `Access Denied` error at the UI plane because the active Access Token (AT) lacks the specific data-plane assignment scope.

### 2. Negative Constraints
* **Constraint 01:** Microsoft Entra ID (MEID) Conditional Access (CA) policies cannot read *User-level* Custom Security Attributes (CSA) natively during the initial user sign-in phase. User-level Custom Security Attributes (CSA) are structurally engineered for downstream Azure Data-Plane Role-Based Access Control (RBAC) conditions.

### 3. The Inverse Logic
* To evaluate user attributes at the initial identity-plane login boundary using Conditional Access (CA), an organization must utilize standard directory **Extension Attributes (EA)** (such as `extensionAttribute1`). While Custom Security Attributes (CSA) fail to evaluate at the Conditional Access (CA) login boundary, Extension Attributes (EA) successfully evaluate through the native **Filter for users** condition engine.

### 4. Hard Booleans
* `User.Membership -in Group` ➔ **TRUE** for both Alpha Engineer and Bravo Engineer.
* `User.ExtensionAttribute1 -eq "Tier3"` ➔ **TRUE** for Alpha Engineer, **FALSE** for Bravo Engineer.

---

## Section 11: Clinical Case Studies

### Case Study 01: The Token Stale-State Authorization Failure
* **Configuration:** 
    * Policy: Tenant Directory Role Assignment.
    * Identity: Administrator account.
    * Device State: Unmanaged workstation.
* **Action:** The user account is assigned the Attribute Definition Administrator (ADA) role via the portal interface. The administrator immediately attempts to create a new Attribute Set inside the same browser window session without logging out or utilizing an Incognito Window.
* **Outcome:** `Hard Boolean: Failure`
* **Technical Logic:** The user's current session relies on an existing Access Token (AT) and Refresh Token (RT) issued *prior* to the role assignment event. Because Access Tokens (AT) are immutable objects that cannot dynamically ingest directory role modifications mid-lifecycle, the current Access Token (AT) contains no claims for the Attribute Definition Administrator (ADA) role. The directory target reads the token schema, notes the missing claim token, and blocks the write command. The root cause is remedied only when a new authentication handshake occurs via an Incognito Window, forcing the token issuing service to generate a fresh Access Token (AT) embedded with the active role claims.
