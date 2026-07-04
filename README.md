
LAB SUMMARY: SUCCESSFUL CONDITIONAL ACCESS PERIMETER CONFIGURATION

1. USER DIRECTORY ATTRIBUTE CONFIGURATION
--------------------------------------------------------------------------------
To establish attribute-based access controls, specific text strings were mapped 
directly to the user account profiles inside Microsoft Entra ID (MEID):

* Alpha Engineer (Source):
  - Job Title Attribute: Tier3
  - Department Attribute: ProjectQuantum
* Bravo Engineer (Source):
  - Job Title Attribute: Tier1
  - Department Attribute: ProjectLegacy

2. SECURITY GROUP ARCHITECTURE
--------------------------------------------------------------------------------
Two separate group containers were constructed to manage policy targeting.

* The Macro Static Group (ABAC-Engineers-Tier3-Quantum):
  - Group Type: Static Security Group.
  - Membership: Alpha Engineer (Source) and Bravo Engineer (Source) were both 
    manually added as direct members. This group establishes the broad net 
    for the policy scope.
    
* The Micro Dynamic Group (ABAC-Authorized-Tier3-Quantum):
  - Group Type: Dynamic Security Group.
  - Membership Rule Syntax: 
    (user.jobTitle -eq "Tier3") -and (user.department -eq "ProjectQuantum")
  - Operational Behavior: The MEID identity engine automatically reads user 
    attributes. Because Alpha Engineer matches the rule, Alpha is automatically 
    added to this group. Because Bravo Engineer fails the rule, Bravo is left out.

3. CONDITIONAL ACCESS POLICY CONFIGURATION
--------------------------------------------------------------------------------
The Conditional Access (CA) policy was configured under the specific name 
`ABAC-Office365-Enforcement-Gate`. To make the policy execute successfully, 
the structural assignments must be exactly as follows:

* Users and Groups Assignment:
  - INCLUDE Tab: Choose "Select users and groups" and check the box for the 
    Macro Static Group (ABAC-Engineers-Tier3-Quantum).
  - EXCLUDE Tab: Choose "Select users and groups" and check the box for the 
    Micro Dynamic Group (ABAC-Authorized-Tier3-Quantum).
* Target Resources Assignment:
  - INCLUDE Tab: Set the radio button to "All resources" (formerly "All cloud apps").
  - EXCLUDE Tab: None.
* Access Controls (Grant):
  - Select the radio button for "Block access".
* Policy State:
  - Set the toggle switch to "On".

4. WHY THIS CONFIGURATION WORKS
--------------------------------------------------------------------------------
The identity system requires explicit user assignments to trigger an evaluation. 
When the policy includes all resources and targets the defined groups, access 
tokens are governed by a strict logical symmetry:

* The Failure State (Bravo Blocked): When Bravo Engineer (Source) logs into the 
  tenant, the CA engine evaluates the traffic. Bravo is included via the macro 
  static group. Because Bravo's profile attributes do not match the dynamic group 
  rule, Bravo is not excluded. The policy applies a hard block, and MEID completely 
  refuses to issue an Access Token (AT) or a Refresh Token (RT), stopping Bravo 
  at the door with an explicit access denied error screen.

* The Success State (Alpha Allowed): When Alpha Engineer (Source) logs into the 
  tenant, the CA engine evaluates the traffic. Alpha is included via the macro 
  static group, but Alpha's matching profile attributes successfully place them 
  inside the micro dynamic group. In MEID logic, an explicit exclusion always 
  overrides an inclusion. The policy evaluates as "Not Applied" for Alpha, the 
  block gate lifts, and Alpha is successfully issued an Access Token (AT) and a 
  Refresh Token (RT) to log into the workspace.
================================================================================
