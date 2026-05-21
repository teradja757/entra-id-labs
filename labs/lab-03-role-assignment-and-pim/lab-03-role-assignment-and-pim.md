# Lab 03 — Entra ID Role Assignment and Privileged Access Management

**Platform:** Microsoft Entra ID
**Tenant:** Mitchell's Cloud Services
**License:** Microsoft Entra ID Free

## What This Lab Covers

Assigned Entra ID built-in roles to users through the portal and verified all assignments using Microsoft Graph PowerShell. Two roles were configured — Helpdesk Administrator and Global Reader — using the same users from the RBAC groups built in Lab 02. The PowerShell section pulls a full role assignment report formatted for audit use. The final section covers how this setup would run in production with PIM, including eligible assignments, JIT activation, and Zero Trust alignment.

## Environment

- **Tenant:** Mitchell's Cloud Services
- **Admin account:** Teradja Mitchell
- **Users provisioned:** 21 accounts
- **Starting state:** No role assignments configured

## Roles Assigned

| Role | Assigned Users | What It Can Do |
|------|---------------|----------------|
| Helpdesk Administrator | Ciara Walker, Brianna Davis, Andre Thompson, Aaliyah Jackson | Reset passwords for non-administrators and Helpdesk Administrators |
| Global Reader | Elijah Hall, Destiny Johnson, DeShawn Scott, Darius Martin | Read everything a Global Administrator can see but cannot change anything |

## Part 1 — Portal

### Step 1 — Confirm Baseline

Navigated to Entra ID > Roles and administrators and confirmed no assignments existed on either target role before starting.

![All Roles Baseline](screenshots/01-all-roles-baseline.png)
*Full list of built-in roles with zero assignments on Helpdesk Administrator and Global Reader.*

### Step 2 — Assign Helpdesk Administrator

Opened Helpdesk Administrator and confirmed it was empty.

![Helpdesk Admin Baseline](screenshots/02-helpdesk-admin-role-baseline.png)
*Helpdesk Administrator with no assignments.*

Clicked Add assignments and selected the four users from the SG-RBAC-HelpDesk group in Lab 02. On the free tier, role assignments go to individual users directly. In a P1 environment this would be a single assignment to the SG-RBAC-HelpDesk group, which is the cleaner and more scalable approach.

![Helpdesk Admin Assigned](screenshots/03-helpdesk-admin-assigned.png)
*Ciara Walker, Brianna Davis, Andre Thompson, and Aaliyah Jackson assigned to Helpdesk Administrator.*

### Step 3 — Assign Global Reader

Same process for Global Reader. Confirmed empty state, then added the four users from SG-RBAC-Compliance.

![Global Reader Baseline](screenshots/04-global-reader-role-baseline.png)
*Global Reader with no assignments.*

![Global Reader Assigned](screenshots/05-global-reader-assigned.png)
*Elijah Hall, Destiny Johnson, DeShawn Scott, and Darius Martin assigned to Global Reader.*

## Part 2 — PowerShell

Connected to Microsoft Graph with the scope required to read role assignment data:

```powershell
Connect-MgGraph -Scopes "RoleManagement.Read.Directory" `
    -TenantId "b0275963-f730-47b9-90da-7ef6ea7da0f9" -UseDeviceCode
```

![PowerShell Connected](screenshots/06-powershell-connected.png)
*Connected to Microsoft Graph via device code authentication.*

### Export All Role Assignments

Pulled every active role assignment in the tenant, resolved each to a readable role name and user, and exported to CSV. This is the type of report used during SOC 2 access reviews to document who holds privileged access and what they are authorized to do.

```powershell
$assignments = Get-MgRoleManagementDirectoryRoleAssignment -All
$report = @()

foreach ($assignment in $assignments) {
    $roleDef = Get-MgRoleManagementDirectoryRoleDefinition -UnifiedRoleDefinitionId $assignment.RoleDefinitionId
    $user = Get-MgUser -UserId $assignment.PrincipalId -ErrorAction SilentlyContinue

    $report += [PSCustomObject]@{
        RoleName  = $roleDef.DisplayName
        UserName  = $user.DisplayName
        UserUPN   = $user.UserPrincipalName
        Scope     = $assignment.DirectoryScopeId
    }
}

$report | Export-Csv -Path "C:\MCS_Role_Assignments.csv" -NoTypeInformation
Write-Host "Report exported to C:\MCS_Role_Assignments.csv"
```

![Role Assignments Export](screenshots/07-powershell-role-assignments-export.png)
*All assignments queried and exported to C:\MCS_Role_Assignments.csv.*

![Role Assignments Excel](screenshots/08-role-assignments-excel.png)
*Full assignment report showing role, user, UPN, and scope for every principal in the tenant.*

### Query Role Descriptions

Pulled the official description for each role directly from the directory for documentation purposes.

```powershell
$roles = @("Helpdesk Administrator", "Global Reader")

foreach ($roleName in $roles) {
    $role = Get-MgRoleManagementDirectoryRoleDefinition -Filter "DisplayName eq '$roleName'"
    Write-Host "Role: $($role.DisplayName)"
    Write-Host "Description: $($role.Description)"
    Write-Host "---"
}
```

![Role Descriptions](screenshots/09-role-descriptions-powershell.png)
*Role descriptions returned directly from Entra ID.*

**Helpdesk Administrator:** Can reset passwords for non-administrators and Helpdesk Administrators.

**Global Reader:** Can read everything that a Global Administrator can, but not update anything.

## Part 3 — Production Design with PIM

The assignments in this lab are permanent and active. The user holds the role continuously from the moment it is assigned. That works for a lab environment but it is a problem in production.

Permanent standing privilege means that if an account is compromised, the attacker inherits the role immediately with no additional steps required. There is no time limit, no approval gate, nothing to slow them down. The access just exists.

Privileged Identity Management addresses this by separating authorization from access. Instead of holding a role permanently, a user is marked as eligible. They do not actually have the role until they request it, provide a justification, and activate it. Once the activation window closes, the role is removed automatically.

### Eligible vs Active

Active assignment means the user holds the role right now, continuously. This is what was configured in Part 1.

Eligible assignment means the user is authorized to request the role but does not hold it. They activate it when they need it, use it within the configured window, and then it is automatically revoked.

### JIT Activation Flow

1. User requests activation of their eligible role through PIM or My Access
2. User provides a business justification for why they need the access
3. If approval is required, a designated approver accepts or denies the request
4. Once approved, the role activates for the configured time window
5. All steps are logged including the request, justification, approver decision, activation time, and expiration
6. When the window closes the role is removed automatically

### PIM Configuration for These Two Roles

**Helpdesk Administrator**
- Assignment type: Eligible
- Activation window: 4 hours
- Justification required: Yes
- MFA required on activation: Yes
- Approval required: Optional at this privilege level
- Review cycle: Every 90 days

**Global Reader**
- Assignment type: Eligible
- Activation window: 2 hours
- Justification required: Yes
- MFA required on activation: Yes
- Approval required: Yes. Read access across the entire tenant warrants a second approval before activation
- Review cycle: Every 90 days

### Zero Trust Alignment

PIM maps directly to the three Zero Trust principles as they apply to identity.

**Verify explicitly.** Every activation requires MFA and a documented justification. Access is not assumed based on a previous assignment. It is verified at the point of use every time.

**Use least privilege.** No standing access exists. The role is only active during the window needed to complete a specific task. Once that window closes the privilege is gone.

**Assume breach.** An eligible assignment gives a compromised account nothing to work with. The attacker still has to complete an MFA-protected activation that generates real-time alerts. That is a fundamentally different exposure profile than a permanently active privileged account.

The PIM audit log, covering who activated what role, when, with what justification, who approved it, and when it expired, is exactly the privileged access documentation that HIPAA and SOC 2 auditors look for during a control review.

## Compliance Mapping

| Framework | Control | How This Addresses It |
|-----------|---------|----------------------|
| HIPAA | 45 CFR 164.312(a)(1) | Role-based access control with defined privilege boundaries per role |
| SOC 2 | CC6.1, CC6.3 | Logical access controls and privileged access management |
| HITRUST CSF | 01.a, 01.b, 07.a | Formal access control with defined roles and least privilege enforcement |
| NIST 800-53 | AC-2, AC-3, AC-6 | Account management, access enforcement, and least privilege |

## What I Would Do Differently in Production

- Use Entra ID P1 to assign roles to SG-RBAC-HelpDesk and SG-RBAC-Compliance directly instead of individual users
- Use Entra ID P2 to configure all assignments as eligible through PIM with JIT activation
- Set activation windows based on role risk level, shorter for higher privilege
- Require approval for any role with broad read or write access across the tenant
- Run quarterly access reviews on all privileged role assignments
- Configure PIM alerts for activations outside business hours or from unusual locations

## References

- [Microsoft Docs — Entra ID Built-in Roles](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference)
- [Microsoft Docs — What is Privileged Identity Management](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)
- [Microsoft Docs — Assign Entra ID Roles in PIM](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-how-to-add-role-to-user)
- [Microsoft Graph PowerShell — Get-MgRoleManagementDirectoryRoleAssignment](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.governance/get-mgrolemanagementdirectoryroleassignment)

*Teradja Mitchell | [LinkedIn](https://www.linkedin.com/in/teradja-mitchell-0494b9208) | [GitHub](https://github.com/teradja757)*
