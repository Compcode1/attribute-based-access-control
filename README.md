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

   Think of this as a two-dimensional lock designed to enforce strict "need-to-know" security: one axis is vertical rank, and the other is horizontal team assignment. The ClearanceLevel acts as your vertical ladder (Tier 1, 2, or 3)—you can only occupy one rung at a time, and we use a rigid dropdown so a simple typo can't accidentally grant someone executive access. The ProjectCode is your horizontal boundary; because an engineer can work on three different projects simultaneously, this attribute needs to hold a flexible list of text values that change as projects spin up and down. By combining them, you create a bulletproof gate: an engineer can only access a sensitive file if they have both the required rank and an active assignment to that specific project, completely preventing a high-clearance engineer from snooping on teams they don't belong to.



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
It boils down to a basic feature limitation: Entra’s automated sorting machine (the dynamic group engine) is completely blind to these new custom security tags. If you try to write an automated rule using them, the system just errors out because it doesn't look at that data. To bypass this roadblock, we skip the automation entirely and build a plain, manual bucket (a static group) to throw both engineers into. We aren't using this group to filter them; we are just using it as a net to bundle them together so we can push them both to our final security gate. That gate can read their individual tags, so it will successfully block Bravo while letting Alpha through.
### 3. Downstream Policy Leverage
This group acts as the macro-target for the Conditional Access (CA) policy. By targeting this group, both Alpha Engineer and Bravo Engineer are forced into the exact same security evaluation gate, allowing the policy engine to prove how it handles their differing user metadata profiles.

---

## The Forward Strategy: Identity-Plane Verification Pivot

## Phase 5: Identity-Plane Attribute Verification & Policy Enforcement

To build our "two-dimensional lock" (Rank + Project) entirely within the identity portal, we utilize two standard, highly visible profile fields natively tracked by the system: **Job Title** (representing Rank/Clearance Level) and **Department** (representing the active Project Code).

### Step 1: Map and Stamp the Identity Profile Tags
We manually update our test engineers' profiles within the Microsoft Entra admin center to embed their security clearance and project codes directly onto their identity records.

#### Execution Directions:
1. **Open the Users Directory:** In the left-hand menu, navigate to **Identity** > **Users** > **All users**.
2. **Configure Alpha Engineer:**
   * Find and click on **Alpha Engineer** from the list.
   * Click **Edit properties** in the top menu bar.
   * Scroll down to the **Job information** section.
   * In the **Job title** field, type exactly: `Tier3`
   * In the **Department** field, type exactly: `ProjectQuantum`
   * Click the blue **Save** button at the very bottom of the screen.
3. **Configure Bravo Engineer:**
   * Return to **All users**, then find and click on **Bravo Engineer**.
   * Click **Edit properties** in the top menu bar.
   * Scroll down to the **Job information** section.
   * In the **Job title** field, type exactly: `Tier1`
   * In the **Department** field, type exactly: `ProjectLegacy`
   * Click the blue **Save** button at the very bottom of the screen.

### Step 2: Build the Smart Filtering Gate
Because the Microsoft Entra ID (MEID) Conditional Access (CA) policy engine cannot parse raw string logic directly within its user assignment tab, we implement a production-grade **Include/Exclude Pattern**. We first build a dynamic security group to automatically discover and group authorized attributes, then we use a Conditional Access (CA) policy to block our general testing group while explicitly excluding the dynamically authorized group.

#### Part A: Provision the Dynamic Authorization Group
1. **Navigate to Groups:** In the left-hand menu of the Microsoft Entra admin center (MEAC), expand **Identity** > **Groups** > select **All groups**, and click **New group**.
2. **Configure Baseline Parameters:**
   * **Group type:** Select `Security`.
   * **Group name:** Type exactly: `ABAC-Authorized-Tier3-Quantum`
   * **Membership type:** Change the dropdown from *Assigned* to `Dynamic User`.
3. **Construct the Attribute Query:**
   * Under the **Dynamic user members** section, click the **Add dynamic query** hyperlink.
   * On the dynamic membership rules page, look at the top right of the rule builder and click the **Edit** button next to *Rule syntax*.
   * Clear any existing text in the flyout window and paste this exact engineering rule:
     `(user.jobTitle -eq "Tier3") -and (user.department -eq "ProjectQuantum")`
   * Click **OK** to close the syntax window, then click **Save** at the top of the query builder blade.
4. **Finalize Group:** Click the blue **Create** button at the bottom of the main page. (The directory engine will now background-scan your users; Alpha Engineer will automatically join this group, while Bravo Engineer will be skipped).

#### Part B: Configure the Conditional Access Policy Perimeter
1. **Navigate to Policies:** In the left menu of the Entra admin center, go to **Identity** > **Protection** > **Conditional Access**.
2. **Create New Container:** Click **+ New policy** from the top menu bar to open a fresh, clean-slate policy configuration workspace.
3. **Name the Policy:** In the **Name** text field at the very top of the blade, type exactly: `ABAC-Office365-Enforcement-Gate`
4. **Target the Resource Plane:**
   * Click on **Target resources** (formerly Cloud apps). Ensure the top dropdown is set to **Cloud apps**.
   * On the **Include** tab, select the radio button for **Select resources**.
   * Click the blue **Select** hyperlink that appears below. In the right-hand flyout search bar, type exactly: `Office 365`.
   * Check the box next to **Office 365** in the results list, and click the blue **Select** button at the bottom.
5. **Enforce the Block Control:**
   * Under the *Access controls* section at the bottom, click on **Grant**.
   * In the right-hand panel, select the radio button for **Block access**.
   * Click the green **Select** button at the bottom of the panel.
6. **Activate the Policy:**
   * Locate the **Enable policy** toggle at the very bottom of the main screen.
   * Flip the toggle from *Report-only* to **On**.
   * Click the blue **Create** (or **Save**) button.

### Step 3: Validate the Perimeter (The Live Test)
We execute separate private browser sessions to witness the identity evaluation in real-time:
* **Bravo Engineer Test:** Bravo attempts a sign-in. The engine reads their fields (`Tier1` / `ProjectLegacy`), flags the mismatch against the required parameters, and triggers an immediate **Access Denied** block screen.
* **Alpha Engineer Test:** Alpha attempts a sign-in. The engine reads their fields (`Tier3` / `ProjectQuantum`), confirms the match, bypasses the block rule, and successfully **Authorizes** the login.
