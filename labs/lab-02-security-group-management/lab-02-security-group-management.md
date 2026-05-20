# Lab 02 — Security Group Creation and Membership Management

**Platform:** Microsoft Entra ID
**Tenant:** Mitchell's Cloud Services
**License:** Microsoft Entra ID Free

## What This Lab Covers

Created and configured security groups in Microsoft Entra ID using both the portal and PowerShell. The lab covers three distinct group types used in real enterprise environments — Conditional Access target groups, app-based groups with separation of duties, and RBAC role assignment groups. The PowerShell portion covers group creation, bulk member assignment, and exporting a full group membership report.

## Why Group Structure Matters

Security groups are the foundation of access control in any Microsoft environment. Rather than assigning permissions directly to users, groups let you manage access at scale. Add a user to the right group and they inherit the appropriate access. Remove them and it's gone. This is how least privilege gets enforced in practice across HIPAA, SOC 2, and HITRUST environments.

## Environment

- **Tenant:** Mitchell's Cloud Services
- **Admin account:** Teradja Mitchell
- **Users provisioned:** 21 accounts
- **Starting state:** No groups configured

## Group Types Demonstrated

| Type | Purpose | Example |
|------|---------|---------|
| Conditional Access target | Used as the assignment target for CA policies | SG-CA-MFA |
| App-based with separation of duties | Controls access within an application by role | SG-App-PowerBI-Admins / SG-App-PowerBI-Users |
| RBAC role assignment | Container for Entra ID built-in role assignments | SG-RBAC-HelpDesk / SG-RBAC-Compliance / SG-RBAC-SecOps |

In production, these groups would also be candidates for dynamic membership rules using Entra ID P1, where membership is automatically maintained based on user attributes like department or job title. This lab uses assigned membership since the tenant is on the free tier, but the group structure and naming conventions reflect what you would build in a licensed environment.

## Part 1 — UI

### Step 1 — Confirm Baseline

Navigated to Groups in the Azure portal and confirmed no groups existed. This is the starting state before any access control structure is in place.

![Groups Baseline](screenshots/GitHub%20Labs%20Lab%2002%20-%20Group%20%26%20Membership%20Management/01-groups-baseline.png)
*Groups page showing 0 groups — clean starting state.*

### Step 2 — Create SG-CA-MFA (Conditional Access Target Group)

The first group created was SG-CA-MFA. In a P1 environment this group would be assigned directly to a Conditional Access policy requiring MFA. All users are members so the policy would apply tenant-wide.

- **Group type:** Security
- **Group name:** SG-CA-MFA
- **Description:** Conditional Access target group — MFA required for ALL users
- **Membership type:** Assigned
- **Members:** All 21 users

![SG-CA-MFA Creation](screenshots/GitHub%20Labs%20Lab%2002%20-%20Group%20%26%20Membership%20Management/02-new-group-sec-mfa-enforced.png)
*New group form for SG-CA-MFA with all 21 users selected.*

![SG-CA-MFA Created](screenshots/GitHub%20Labs%20Lab%2002%20-%20Group%20%26%20Membership%20Management/03-groups-sg-ca-mfa-created.png)
*SG-CA-MFA live in the tenant.*

### Step 3 — Create App-Based Groups with Separation of Duties

Created two groups for PowerBI access — one for admins and one for standard users. This demonstrates separation of duties at the application level. Admins get full publish and manage rights while standard users can only view and interact with reports.

Keeping these as separate groups means you can assign different application roles to each group rather than managing permissions per user. It also makes access reviews simpler — you review group membership rather than individual app assignments.

**SG-App-PowerBI-Admins**
- 3 members — small by design, admin access should always be restricted

![PowerBI Admins Group](screenshots/GitHub%20Labs%20Lab%2002%20-%20Group%20%26%20Membership%20Management/04-new-group-sg-app-powerbi-admins.png)
*SG-App-PowerBI-Admins with 3 members selected.*

**SG-App-PowerBI-Users**
- 7 members — broader access, view and interact only

![PowerBI Users Group](screenshots/GitHub%20Labs%20Lab%2002%20-%20Group%20%26%20Membership%20Management/05-new-group-sg-app-powerbi-users.png)
*SG-App-PowerBI-Users with 7 members selected.*

![Three Groups Created](screenshots/GitHub%20Labs%20Lab%2002%20-%20Group%20%26%20Membership%20Management/06-three-groups-created.png)
*Three groups live after creating the CA and PowerBI groups.*

### Step 4 — Create RBAC Role Assignment Groups

Created two groups designed to hold Entra ID built-in role assignments. Instead of assigning roles directly to users, roles get assigned to these groups. When a user is added to the group they inherit the role automatically. When they leave the group the role is revoked.

This approach scales better than direct role assignment and makes it easier to audit who has privileged access — you just look at group membership.

**SG-RBAC-HelpDesk**
- Maps to the Helpdesk Administrator role in Entra ID
- 4 members — support staff who need password reset and basic user management capabilities

![HelpDesk RBAC Group](screenshots/GitHub%20Labs%20Lab%2002%20-%20Group%20%26%20Membership%20Management/07-new-group-sg-rbac-helpdesk.png)
*SG-RBAC-HelpDesk configured for Helpdesk Administrator role assignment.*

**SG-RBAC-Compliance**
- Maps to the Global Reader role in Entra ID
- 4 members — audit and compliance staff who need read-only visibility across the tenant

![Compliance RBAC Group](screenshots/GitHub%20Labs%20Lab%2002%20-%20Group%20%26%20Membership%20Management/08-new-group-sg-rbac-compliance.png)
*SG-RBAC-Compliance configured for Global Reader role assignment.*

### Step 5 — Verify All Groups

After creating all groups, confirmed the full list in the portal. Five security groups covering three distinct use cases.

![All Groups Final](screenshots/GitHub%20Labs%20Lab%2002%20-%20Group%20%26%20Membership%20Management/09-all-groups-final.png)
*All 5 groups live in the tenant — CA target, app-based separation of duties, and RBAC containers.*

## Part 2 — PowerShell

Connected to Microsoft Graph with the scopes needed for group management:

```powershell
Connect-MgGraph -Scopes "Group.ReadWrite.All", "GroupMember.ReadWrite.All" `
    -TenantId "b0275963-f730-47b9-90da-7ef6ea7da0f9" -UseDeviceCode
```

![PowerShell Connected](screenshots/GitHub%20Labs%20Lab%2002%20-%20Group%20%26%20Membership%20Management/10-powershell-connected.png)
*Connected to Microsoft Graph via device code authentication.*

### Create a Group via PowerShell

Created SG-RBAC-SecOps programmatically. Same operation as clicking through the portal but scriptable and repeatable at scale.

```powershell
$group = New-MgGroup -DisplayName "SG-RBAC-SecOps" `
    -Description "Security operations team — Security Administrator role assignment" `
    -MailEnabled:$false `
    -MailNickname "SG-RBAC-SecOps" `
    -SecurityEnabled:$true

Write-Host "Created group: $($group.DisplayName) — ID: $($group.Id)"
```

![Create Group PowerShell](screenshots/GitHub%20Labs%20Lab%2002%20-%20Group%20%26%20Membership%20Management/11-powershell-create-group.png)
*SG-RBAC-SecOps created via PowerShell with Group ID returned.*

### Add Members to a Group via PowerShell

Added three members to the newly created group by UPN. The script resolves each UPN to a user object and adds them as a member using the group ID captured from the previous step.

```powershell
$groupId = $group.Id

$members = @(
    "jordan.white@teradja757gmail.onmicrosoft.com",
    "malik.lewis@teradja757gmail.onmicrosoft.com",
    "marcus.williams@teradja757gmail.onmicrosoft.com"
)

foreach ($upn in $members) {
    $user = Get-MgUser -Filter "userPrincipalName eq '$upn'"
    New-MgGroupMember -GroupId $groupId -DirectoryObjectId $user.Id
    Write-Host "Added: $($user.DisplayName)"
}
```

![Add Members PowerShell](screenshots/GitHub%20Labs%20Lab%2002%20-%20Group%20%26%20Membership%20Management/12-powershell-add-members.png)
*Jordan White, Malik Lewis, and Marcus Williams added to SG-RBAC-SecOps.*

### Export All Groups and Membership to CSV

Pulled a full report of every group and all members, exported to CSV. This is the kind of report you would produce during a SOC 2 or HIPAA access review to prove who has access to what.

```powershell
$groups = Get-MgGroup -All
$report = @()

foreach ($group in $groups) {
    $members = Get-MgGroupMember -GroupId $group.Id -All

    if ($members.Count -eq 0) {
        $report += [PSCustomObject]@{
            GroupName = $group.DisplayName
            Member    = "No members"
            MemberUPN = ""
        }
    } else {
        foreach ($member in $members) {
            $user = Get-MgUser -UserId $member.Id -ErrorAction SilentlyContinue
            $report += [PSCustomObject]@{
                GroupName = $group.DisplayName
                Member    = $user.DisplayName
                MemberUPN = $user.UserPrincipalName
            }
        }
    }
    Write-Host "Processed: $($group.DisplayName)"
}

$report | Export-Csv -Path "C:\MCS_Groups_Report.csv" -NoTypeInformation
Write-Host "Report exported to C:\MCS_Groups_Report.csv"
```

![Groups Export PowerShell](screenshots/GitHub%20Labs%20Lab%2002%20-%20Group%20%26%20Membership%20Management/13-powershell-groups-export.png)
*All 6 groups processed and exported to C:\MCS_Groups_Report.csv.*

![Groups Report Excel](screenshots/GitHub%20Labs%20Lab%2002%20-%20Group%20%26%20Membership%20Management/14-groups-report-excel.png)
*Full group membership report — every group, every member, every UPN.*

## Compliance Mapping

| Framework | Control | How This Addresses It |
|-----------|---------|----------------------|
| HIPAA | 45 CFR 164.312(a)(1) | Role-based access control implemented through security groups |
| SOC 2 | CC6.1, CC6.2 | Logical access and group-based permission management |
| HITRUST CSF | 01.a, 01.b | Formal access control structure with separation of duties |
| NIST 800-53 | AC-2, AC-3 | Account management and access enforcement via groups |

## What I Would Do Differently in Production

- Use dynamic membership rules for department and role-based groups with Entra ID P1
- Assign Entra ID built-in roles directly to the RBAC groups rather than managing per-user
- Enforce a naming convention policy so group naming stays consistent as the tenant scales
- Schedule the PowerShell membership report to run automatically for access review cycles
- Nest groups where it makes sense — for example pulling the RBAC groups into the CA target group

## References

- [Microsoft Docs — Manage groups in Entra ID](https://learn.microsoft.com/en-us/entra/fundamentals/how-to-manage-groups)
- [Microsoft Docs — Group-based role assignment](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/groups-concept)
- [Microsoft Graph PowerShell — New-MgGroup](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.groups/new-mggroup)

*Teradja Mitchell | [LinkedIn](https://www.linkedin.com/in/teradja-mitchell-0494b9208) | [GitHub](https://github.com/teradja757)*
