# Lab 05 — SSO Token Configuration and OIDC Claim Mapping

**Platform:** Microsoft Entra ID
**Tenant:** Mitchell's Cloud Services
**License:** Microsoft Entra ID Free

## Important Disclaimer

This lab covers the Entra ID side of SSO setup only. SSO requires two things working together -- an identity provider, which is Entra ID, and a real application that can receive and use the tokens Entra sends. The PowerBI app from Lab 04 is a registration only. There is no actual application behind it. Because of that, a full SSO flow cannot be completed here. What this lab documents is the Entra ID configuration that has to be done before a real application can use SSO. That includes confirming the sign-on protocol, reviewing the claims setup, and configuring group claims so that a user's group memberships are included in the token when they log in.

## What This Lab Covers

Reviewed the OIDC-based sign-on configuration on the PowerBI Enterprise Application, documented why Entra ID defaulted the app to OpenID Connect, walked through the claims configuration and documented where group claims actually get configured for an OIDC app, set up a security group claim through Token configuration on the App Registration, and validated the result via PowerShell.

## Why SSO Configuration Matters

SSO means a user logs in once and gets access to multiple applications without having to log in again for each one. Every sign-in goes through Entra ID, which means you have one place to enforce MFA, review sign-in activity, and cut off access when needed. Without SSO, each application manages its own login, which makes access harder to monitor and harder to revoke quickly.

For environments that have to meet compliance requirements like HIPAA or SOC 2, having all authentication flow through a central identity provider is important. It creates a single audit trail showing who logged in, when, and from where.

## OIDC vs SAML

OIDC and SAML are both protocols that handle SSO. They do the same job but in different ways.

SAML is older. It requires you to manually configure both sides -- the identity provider and the application -- with certificates, metadata files, and endpoint URLs. It works but it takes more setup.

OIDC is newer and handles a lot of that setup automatically. When the app was registered in Lab 04 using the standard flow, Entra ID defaulted it to OIDC because the app uses Microsoft's own APIs. That is the right protocol for this app. Switching it to SAML would just add extra configuration steps with no benefit.

## Environment

- **Tenant:** Mitchell's Cloud Services
- **Admin account:** Teradja Mitchell
- **App registration:** PowerBI (from Lab 04)
- **Enterprise Application:** PowerBI (service principal)

## Part 1 — Portal

### Step 1 — Review OIDC Sign-On Baseline

Navigated to Enterprise applications > All applications > PowerBI > Single sign-on. The blade opened directly on the OIDC-based Sign-on screen. This happened because the app was registered using the standard App Registration flow in Lab 04, which defaults to OpenID Connect. Entra ID recognized the registration type and set the protocol automatically.

The screen showed two things worth noting. The app is on OIDC, which means the core login flow is handled by the protocol without manual endpoint setup. And no custom claims had been added yet -- the Attributes and Claims section was empty.

![OIDC Sign-On Baseline](screenshots/01-enterprise-app-sso-oidc-baseline.png)
*PowerBI Enterprise Application SSO blade showing OIDC-based Sign-on. The banner at the top confirms that because the app uses OpenID Connect, most of the SSO setup is already handled. No custom claims are configured yet.*

### Step 2 — Review Attributes and Claims

Clicked Edit on the Attributes and Claims section. The panel opened showing no claims configured under either Required claim or Additional claims.

This is expected. With OIDC, the basic identity information -- things like the user's name, email, and a unique identifier -- are included in the token automatically by the protocol. You do not have to define those manually. SAML works differently and requires you to map every attribute explicitly, but OIDC handles the standard ones by default.

![Attributes and Claims Baseline](screenshots/02-attributes-claims-baseline.png)
*Attributes and Claims panel showing no claims configured. With OIDC, standard identity claims are included in the token automatically without manual setup.*

### Step 3 — Group Claims Limitation on Enterprise Application Side

Clicked Add a group claim from the Attributes and Claims panel. The Group Claims panel opened with options to choose which group types to include in the token. A warning appeared immediately saying that group claims cannot be customized from this location and that the correct place to configure them is Token configuration on the App Registration side.

This is how the platform works for OIDC apps, not a licensing limitation. The Enterprise Application side handles claim customization for SAML apps. For OIDC, group claims are configured through the App Registration. The warning is the portal directing you to the right place.

![Group Claims Panel](screenshots/03-group-claims-panel.png)
*Group Claims panel on the Enterprise Application side showing the warning. Group claims for OIDC apps have to be configured through Token configuration on the App Registration, not here.*

### Step 4 — Token Configuration Baseline

Navigated to App registrations > PowerBI > Token configuration using the link in the warning. This is the correct place to configure group claims for an OIDC app. The page showed no claims configured yet.

![Token Configuration Baseline](screenshots/04-token-configuration-baseline.png)
*App Registration Token configuration blade with no claims configured. This is the right location for group claim setup on an OIDC application.*

### Step 5 — Configure Security Group Claim

Clicked Add groups claim. The panel opened with four options for which types of groups to include in the token: Security groups, Directory roles, All groups, and Groups assigned to the application.

Selected Security groups only. The security groups from Lab 02 are what control access in this tenant. There is no reason to include Directory roles or distribution lists in the token -- the application does not need that information and including it would make the token larger than it needs to be. Keeping it to Security groups means only the relevant group memberships travel in the token.

Left the token type settings at their defaults. The groups claim will be included in ID tokens, Access tokens, and SAML tokens.

![Edit Groups Claim Panel](screenshots/05-edit-groups-claim-panel.png)
*Edit groups claim panel with Security groups selected. All other group types left unchecked. Token type settings left at default.*

Clicked Add.

### Step 6 — Confirm Groups Claim Configured

The Token configuration page updated and now shows the groups claim in the table. Claim name is `groups`, Token type is `ID, Access, SAML`, and Optional settings shows `Default`.

Default means the token will include the object IDs of the user's security groups. Object IDs are unique identifiers for each group in Entra ID. In production with an Entra ID P1 license you could switch this to include the group's display name instead, which is easier for an application to work with. For this lab, object IDs are fine.

![Groups Claim Configured](screenshots/06-groups-claim-configured.png)
*Token configuration showing the groups claim added. Token type confirms the claim will be in all token types. Default under Optional settings means group object IDs are included in the claim.*

## Part 2 — PowerShell

Connected to Microsoft Graph with the scope needed to read app registration data:

```powershell
Connect-MgGraph -Scopes "Application.Read.All" `
    -TenantId "b0275963-f730-47b9-90da-7ef6ea7da0f9" -UseDeviceCode
```

![PowerShell Connected](screenshots/07-powershell-connected.png)
*Connected to Microsoft Graph via device code authentication.*

### Validate Token Configuration

Pulled the app registration and confirmed the group claim configuration is reflected on the object.

```powershell
$app = Get-MgApplication -Filter "DisplayName eq 'PowerBI'"

[PSCustomObject]@{
    DisplayName           = $app.DisplayName
    AppId                 = $app.AppId
    GroupMembershipClaims = $app.GroupMembershipClaims
    SignInAudience        = $app.SignInAudience
} | Format-List
```

![PowerShell SSO Validation](screenshots/08-powershell-sso-validation.png)
*GroupMembershipClaims returning SecurityGroup confirms the group claim is active on the app registration. SignInAudience returning AzureADMyOrg confirms the app is locked to this tenant only.*

| Property | Value | What It Confirms |
|---|---|---|
| DisplayName | PowerBI | Correct app registration pulled |
| AppId | 72e1afe4-1e61-454e-be25-ff0847aac78a | Matches the App Registration from Lab 04 |
| GroupMembershipClaims | SecurityGroup | Security group claim is configured and active |
| SignInAudience | AzureADMyOrg | App is single-tenant, scoped to Mitchell's Cloud Services only |

## How This Fits Into a Complete SSO Implementation

The Entra ID side of SSO for this app is set up. The app is using the right protocol, the group claim is configured so a user's security group memberships are included in the token, and the app roles from Lab 04 are included in the token automatically. If a real application were built against this registration, the identity provider side would be ready to go.

What is missing is the application side. A complete SSO flow works like this:

1. A user tries to access the application and gets redirected to Entra ID to log in
2. Entra ID checks who they are, enforces MFA through the SG-CA-MFA group, and issues a token
3. That token includes the user's identity, their security group memberships, and their app role (PowerBI.Admin or PowerBI.User)
4. The application receives the token and checks that it is valid
5. The application reads the claims in the token and gives the user access based on their role

That application-side work cannot be done here because there is no real application behind this registration. The portal and PowerShell output confirm the identity provider configuration is correct. The rest would be built by whoever develops the actual application.

## Compliance Mapping

| Framework | Control | How This Addresses It |
|---|---|---|
| HIPAA | 45 CFR 164.312(d) | Authentication goes through Entra ID with group memberships included in every token |
| SOC 2 | CC6.1, CC6.3 | Access controls enforced at the identity provider with sign-in logs available for review |
| HITRUST CSF | 01.a, 01.d | Authentication configuration documented with defined claim scope |
| NIST 800-53 | AC-2, IA-2 | Account management and user authentication controls |

## What I Would Do Differently in Production

- Build the application against this registration so the full SSO flow can be tested end to end
- Switch GroupMembershipClaims from Default (object IDs) to sAMAccountName with Entra ID P1 so the application gets readable group names instead of GUIDs
- Scope the groups claim to Groups assigned to the application instead of all Security groups to keep the token lean and follow least privilege
- Set up Conditional Access policies targeting SG-CA-MFA to enforce MFA on every sign-in for this app
- Review sign-in logs on the Enterprise Application regularly to catch anything unusual

## References

- [Microsoft Docs -- Configure group claims for applications](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-fed-group-claims)
- [Microsoft Docs -- OpenID Connect on the Microsoft identity platform](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols-oidc)
- [Microsoft Docs -- Optional claims reference](https://learn.microsoft.com/en-us/entra/identity-platform/optional-claims-reference)
- [Microsoft Docs -- Token configuration in app registrations](https://learn.microsoft.com/en-us/entra/identity-platform/optional-claims)
- [Microsoft Graph PowerShell -- Get-MgApplication](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.applications/get-mgapplication)

*Teradja Mitchell | [LinkedIn](https://www.linkedin.com/in/teradja-mitchell-0494b9208) | [GitHub](https://github.com/teradja757)*
