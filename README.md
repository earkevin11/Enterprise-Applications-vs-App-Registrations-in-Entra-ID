# Enterprise Applications vs. App Registrations in Entra ID

## Overview

When working with Microsoft Entra ID (formerly Azure AD), understanding the difference between **Enterprise Applications** and **App Registrations** is critical for proper application management, security, and access control. This repository provides comprehensive guidance on when and why to use each.

---

## Quick Comparison Table

| Aspect | App Registration | Enterprise Application |
|--------|------------------|----------------------|
| **What It Is** | Identity platform registration | Service principal instance (instantiation of an app) |
| **Location** | App registrations blade | Enterprise applications blade |
| **Primary Purpose** | Configure app's identity & authentication | Manage app's access & permissions in your tenant |
| **Created By** | Developer/admin during app development | Automatically created when app is registered OR manually added via gallery |
| **Who Uses It** | Developers building apps | IT admins managing access & security |
| **Scope** | Organization-wide identity definition | Tenant-specific access and permissions |
| **Number in Tenant** | One per app | Can have multiple (if multi-tenant or added via gallery) |
| **Configuration Focus** | Authentication flows, credentials, URIs | Permissions, role assignments, user assignments |
| **Manages** | App credentials, certificates, API permissions | Who can access the app, what roles they have |
| **Stores Credentials?** | YES - Client Secrets, Certificates | NO - References App Registration |
| **Authenticates?** | YES - Uses credentials to prove identity | NO - Relies on App Registration authentication |

---

## CRITICAL: Credentials & Authentication Explained

### Where Are Credentials Stored?

**Answer: Credentials are stored in the APP REGISTRATION, NOT the Enterprise Application**

```
┌─────────────────────────────────────────────────────┐
│         App Registration (Developer Side)           │
│  ┌────────────────────────────────────────────┐    │
│  │ CREDENTIALS STORED HERE:                   │    │
│  │ - Client ID (Application ID)               │    │
│  │ - Client Secret(s)                         │    │
│  │ - Certificates                             │    │
│  │ - Thumbprints                              │    │
│  └────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
                           ↓
                    (Used by app to prove its identity)
                           ↓
┌─────────────────────────────────────────────────────┐
│  Enterprise Application (Admin/Tenant Side)         │
│  ┌────────────────────────────────────────────┐    │
│  │ NO CREDENTIALS HERE:                       │    │
│  │ - User/Group Assignments                   │    │
│  │ - Roles & Permissions                      │    │
│  │ - Access Policies                          │    │
│  │ - Configuration (not secrets)              │    │
│  └────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

---

## Why Does App Registration Have Client Secrets?

### The Authentication Flow Explained

**Client Secrets (and Certificates) exist because:**

An application needs to **prove its identity** to Entra ID before it can:
1. Request access tokens
2. Call protected APIs (Microsoft Graph, SharePoint, etc.)
3. Access resources on behalf of users or itself

### How App Authentication Works

#### **Scenario: PowerShell Script Needs to Manage Azure Resources**

```
┌──────────────────────────────────────────────────────────┐
│  1. App Registration Created                             │
│     - Client ID: abc123def456                            │
│     - Client Secret: SecurePassword@123                  │
│     - Permissions: Directory.ReadWrite.All               │
└──────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────┐
│  2. PowerShell Script Uses Credentials to Authenticate   │
│                                                          │
│  $clientId = "abc123def456"                              │
│  $clientSecret = "SecurePassword@123"                    │
│  $tenantId = "your-tenant-id"                            │
│                                                          │
│  Connect-MgGraph -ClientId $clientId \                   │
│                   -ClientSecret $clientSecret \          │
│                   -TenantId $tenantId                    │
└──────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────┐
│  3. Entra ID Validates the Credentials                   │
│                                                          │
│  "Does this Client ID + Secret match what's stored       │
│   in the App Registration?"                              │
│   → YES ✓ → Grant access token                           │
│   → NO ✗ → Deny access                                   │
└──────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────┐
│  4. Script Receives Access Token                         │
│     Token proves the app is who it claims to be          │
└──────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────┐
│  5. Script Uses Token to Call Microsoft Graph API        │
│     "I have this token from that app, let me manage      │
│      users and groups now"                              │
└──────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────┐
│  6. Enterprise Application Authorization Kicks In        │
│                                                          │
│  "Does this service principal (from App Registration)    │
│   have permission to call this API?"                     │
│   → Checks roles assigned to Enterprise App              │
│   → Checks Directory.ReadWrite.All permission            │
│   → YES ✓ → Allow API call                               │
│   → NO ✗ → Deny API call                                 │
└──────────────────────────────────────────────────────────┘
```

---

## Do Enterprise Applications Have Credentials?

### Answer: NO - Not in the Traditional Sense

**Enterprise Applications do NOT store or manage credentials directly.**

However, they are **linked to credentials** through the App Registration:

```
┌──────────────────────────────────────────────────────────┐
│  App Registration                                        │
│  ├─ Stores: Client ID, Client Secret, Certificates      │
│  └─ Purpose: Authentication (proving identity)          │
│                                                          │
│      ↓ (Uses same identity)                              │
│                                                          │
│  Enterprise Application (Service Principal)              │
│  ├─ Uses: The identity from App Registration             │
│  ├─ Stores: Permissions, role assignments, policies     │
│  └─ Purpose: Authorization (what it can do)             │
└──────────────────────────────────────────────────────────┘
```

**Think of it like:**
- **App Registration's Client Secret** = Your house key
- **Enterprise Application** = Your house's alarm code

You authenticate with the key. Once inside, the alarm code determines what you can access.

---

## How Do They Authenticate?

### Authentication vs. Authorization

| Step | Component | What Happens |
|------|-----------|--------------|
| **1. Authentication** | **App Registration** | App proves its identity using Client Secret/Certificate |
| **2. Get Token** | **App Registration** | Entra ID returns an access token |
| **3. Authorization** | **Enterprise Application** | System checks if the app (service principal) has permission |
| **4. Execute** | **Enterprise Application** | If authorized, app can perform the action |

### Authentication Flow (App Registration)
```
App: "Hello, I'm Client ID 550e8400-e29b-41d4-a716-446655440000"
App: "My password is abc.123~XyZ_DefGhij-KlmnopQrst"

Entra ID: *Checks App Registration database*
Entra ID: "That Client ID and secret match! You're authenticated ✓"
Entra ID: "Here's your access token"
```

### Authorization Flow (Enterprise Application)
```
App: "I have this access token. Can I create a new user?"

Entra ID: *Checks Enterprise Application configuration*
Entra ID: "Let me see what roles/permissions this service principal has..."
Entra ID: "This app has Directory.ReadWrite.All role? YES ✓"
Entra ID: "You're authorized to create users. Go ahead."
```

---

## Real-World Example: Automated User Creation Script

**Scenario:** Your company wants a PowerShell script to automatically create new users in Entra ID every morning

```
┌─────────────────────────────────────────────────────┐
│ STEP 1: Developer Creates App Registration         │
│                                                     │
│ Portal → App Registrations → New Registration      │
│ Name: "User Provisioning Automation"               │
│ Result: Gets Client ID (identifier)                │
└─────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────┐
│ STEP 2: Developer Creates Client Secret            │
│                                                     │
│ App Registration → Certificates & Secrets          │
│ → New Client Secret                                │
│ → Save value securely (this is the PASSWORD)       │
│                                                     │
│ WHY? So the script can prove "I am this app"      │
└─────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────┐
│ STEP 3: Add API Permissions                        │
│                                                     │
│ App Registration → API Permissions                 │
│ → Add: User.ReadWrite.All                          │
│ → Add: Directory.ReadWrite.All                     │
│                                                     │
│ This tells Entra ID: "This app *might* need these  │
│ permissions" (not granting yet)                    │
└─────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────┐
│ STEP 4: Admin Grants Admin Consent                 │
│                                                     │
│ Enterprise Application is automatically created    │
│ Admin grants the permissions via "Grant admin      │
│ consent" button                                    │
│                                                     │
│ WHY? So the app actually HAS permission to do those│
│ things in the tenant                               │
└─────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────┐
│ STEP 5: Script Authenticates (Uses App Registration)
│                                                     │
│ $credential = New-Object System.Management.        │
│   Automation.PSCredential(                          │
│     $clientId,                                      │
│     (ConvertTo-SecureString $clientSecret          │
│       -AsPlainText)                                 │
│   )                                                 │
│                                                     │
│ Connect-MgGraph -Credential $credential \          │
│                  -TenantId $tenantId               │
│                                                     │
│ ACTION: Script sends Client ID + Secret to Entra ID│
│ ENTRA ID CHECKS: Does this secret match this      │
│                  Client ID?                        │
│ RESPONSE: YES → Here's an access token             │
└─────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────┐
│ STEP 6: Script Uses Token (Enterprise App Authorizes)
│                                                     │
│ New-MgUser -DisplayName "John Doe" \              │
│            -UserPrincipalName "john@company.com"  │
│            -PasswordProfile $passwordProfile \     │
│            -AccountEnabled                         │
│                                                     │
│ ACTION: Script sends access token + API request    │
│ ENTRA ID CHECKS: Does the service principal        │
│                  (Enterprise App) have permission  │
│                  to create users?                  │
│ RESPONSE: YES (Directory.ReadWrite.All was granted)
│           User created successfully ✓              │
└─────────────────────────────────────────────────────┘
```

---

## Types of Credentials in App Registration

### 1. **Client Secret** (Password-based)

```
Where: App Registration → Certificates & secrets → Client secrets
Type: Plaintext password
How it works: App sends Client ID + Client Secret to get token
Security: Less secure (can be exposed in code/logs)
Best for: Development/testing, scripts without cert access
Expiry: Can set expiration dates (1 month, 2 years, etc.)

Example:
  Client ID: 550e8400-e29b-41d4-a716-446655440000
  Client Secret: abc.123~XyZ_DefGhij-KlmnopQrst
```

### 2. **Certificate** (Public-private key)

```
Where: App Registration → Certificates & secrets → Certificates
Type: X.509 certificate (.pfx, .cer, .pem)
How it works: App uses private key to sign requests (like digital signature)
Security: More secure (private key never sent to Entra ID)
Best for: Production, Azure VMs, CI/CD pipelines, high security
Expiry: Tied to certificate expiration date

Example:
  Thumbprint: 1234567890ABCDEF1234567890ABCDEF12345678
  Certificate file: app-cert.pfx (contains private key)
```

### 3. **Federated Credentials** (Workload Identity)

```
Where: App Registration → Certificates & secrets → Federated credentials
Type: Trust relationship (no secrets stored)
How it works: GitHub Actions, GitLab CI, etc. issue tokens that Entra ID trusts
Security: Most secure (no long-lived secrets)
Best for: CI/CD pipelines (GitHub Actions, GitLab)
Expiry: None (token-based)

Example:
  GitHub Actions → Issues OIDC token → Entra ID trusts GitHub
  Result: No secrets needed in GitHub
```

---

## Enterprise Application Does NOT Authenticate, It Authorizes

**Important Distinction:**

| Term | Meaning | Who Does It | Component |
|------|---------|------------|-----------|
| **Authentication** | "Are you really who you say you are?" | Entra ID | **App Registration** |
| **Authorization** | "Are you allowed to do this?" | Entra ID | **Enterprise Application** |

---

## Detailed Explanation

### App Registration

#### **Purpose**
An **App Registration** is the first step in integrating an application with Entra ID. It registers your application with the identity platform, allowing it to authenticate users and call protected APIs. Each application you want the Microsoft identity platform to perform identity and access management (IAM) for must be registered. Register an app in the Azure portal so the Microsoft identity platform can provide authentication and authorization services for your application and its users. Whether it's a client application, like a web or mobile app, or a web API that backs a client app, registering it <b> establishes a trust relationship between your application and the identity provider, the Microsoft identity platform. </b>

**Key Points:**
- It's a **developer-focused** artifact
- Represents the **application itself** in Entra ID
- Provides the application with an identity (Client ID)
- Used for configuring how the app authenticates

#### **When to Create**
- Building a custom application that needs Entra ID authentication
- Developing an app that will call Microsoft Graph API or other protected resources
- Creating a service/daemon application that needs to authenticate as itself
- Building a multi-tenant application to be distributed

#### **Configuration Elements**
- **Client ID** (Application ID): Unique identifier for your app
- **Redirect URIs**: Where Entra ID sends the user after authentication
- **API Permissions**: What resources your app can access
- **Credentials**: Client secrets or certificates for authentication
- **Authentication flows**: OAuth 2.0, OpenID Connect, SAML

#### **Use Cases & Scenarios**

| Scenario | Why Use App Registration |
|----------|------------------------|
| Building a web app with Entra ID login | Register to get Client ID and configure redirect URIs |
| Creating a daemon/background service | Register to establish service principal credentials |
| Calling Microsoft Graph API | Register to define API permissions needed |
| Building an API for others to consume | Register to expose your API and manage permissions |
| Multi-tenant SaaS application | Register once, used across multiple tenants |

---

### Enterprise Application

#### **Purpose**
An **Enterprise Application** (also called **Service Principal**) is the **instance** of the registered application **in your specific tenant**. It represents how your app behaves and what permissions it has within your organization.

**Key Points:**
- It's an **admin/IT-focused** artifact
- Represents the **app's presence in your tenant**
- Controls who can access the app and what roles they have
- Manages access policies and conditional access rules

#### **When to Create/Configure**
- Installing a third-party SaaS app (like Salesforce, Slack)
- Adding permissions and assigning users/groups to an app
- Setting up single sign-on (SSO) for an application
- Configuring access policies and conditional access
- Managing which users can access an enterprise app

#### **Configuration Elements**
- **User and Group Assignments**: Who can access the app
- **Roles**: What permissions users have within the app
- **Permissions and Consent**: What the app can do on behalf of users
- **Single Sign-On (SSO) Settings**: SAML, OpenID Connect configuration
- **Provisioning**: Automated user/group provisioning
- **Conditional Access Policies**: Requirements for access (MFA, location, etc.)
- **Application Properties**: Display name, logo, branding

#### **Use Cases & Scenarios**

| Scenario | Why Use Enterprise Application |
|----------|-------------------------------|
| Installing third-party SaaS (Salesforce, Slack) | Add app from gallery to enable SSO in your tenant |
| Restricting app access to specific users | Assign users/groups to control who can access |
| Setting up MFA requirement for an app | Configure conditional access policies |
| Automated user provisioning | Set up SCIM provisioning to sync user accounts |
| Delegating app management to team | Grant application admin roles to team members |

---

## Real-World Scenarios & Decisions

### Scenario 1: Building an Internal Automation Tool

**Request:** "We need to create an app that automates our daily data processing tasks."

**Decision Path:**
1. **Create App Registration** → This gives your automation app an identity and credentials
2. **Use Service Principal** → The app needs to authenticate to Azure resources
3. **Configure Permissions** → Grant the app access to required Microsoft Graph endpoints
4. **Enterprise App Not Needed** → If only the app itself runs (no human users), you may not need user assignments

**Implementation:**
```
App Registration → Define credentials → Enterprise App → Grant permissions to service principal
```

---

### Scenario 2: Deploying Third-Party SaaS (e.g., Salesforce)

**Request:** "We want our users to access Salesforce with their Entra ID credentials."

**Decision Path:**
1. **Find in Gallery** → Search for Salesforce in Entra ID app gallery
2. **Create Enterprise Application** → Add Salesforce to your tenant
3. **No App Registration Needed** → Salesforce already has a pre-configured registration
4. **Configure SSO** → Set up SAML authentication
5. **Assign Users/Groups** → Grant employees access to Salesforce

**Implementation:**
```
Enterprise Applications → Add from gallery → Configure SSO → Assign users
```

---

### Scenario 3: Building a Multi-Tenant SaaS Product

**Request:** "We're building an app that different organizations will use. How do we set this up?"

**Implementation:**
```
STEP 1: Create App Registration (in development/publisher tenant)
        ↓
STEP 2: Configure as multi-tenant (allow "Any Azure AD organization" sign-ins)
        ↓
STEP 3: Add API permissions (Microsoft Graph, custom APIs)
        ↓
STEP 4: Create credentials (client secret/certificate)
        ↓
STEP 5: When customer signs up, Enterprise App is created in their tenant
        ↓
STEP 6: Customer admin assigns users/groups to the Enterprise App in their tenant
```

---

### Scenario 4: Automation Task - Graph API Integration

**Request:** "We need a PowerShell script to automatically manage users and groups in Entra ID."

**Decision Path:**
1. **Create App Registration** → Script needs an identity
2. **Add Microsoft Graph Permissions** → Grant access to user/group management APIs
3. **Create Client Secret** → Script uses this to authenticate
4. **Create Enterprise Application** → Needed to grant actual permissions
5. **Grant Permissions** → Assign Directory.ReadWrite.All or specific roles
6. **No User Assignment Needed** → Service principal authenticates as itself, not users

**Implementation:**
```
App Registration (with credentials)
    ↓
Add Graph Permissions (Directory.ReadWrite.All)
    ↓
Grant Admin Consent
    ↓
Enterprise App shows up automatically
    ↓
PowerShell script uses: Connect-MgGraph -ClientId $clientId -TenantId $tenantId -CertificateThumbprint $cert
```

---

## Decision Tree

```
┌─ Do you need to build a custom application?
│  ├─ YES → Create App Registration
│  │         └─ Does it need to access protected resources?
│  │            ├─ YES → Configure API Permissions in App Registration
│  │            └─ NO → Just register for authentication
│  │
│  └─ NO → Are you installing a third-party SaaS app?
│     ├─ YES → Create Enterprise Application (from gallery or manually)
│     │         └─ Configure SSO and assign users
│     │
│     └─ NO → Are you automating tasks with service credentials?
│        ├─ YES → Create App Registration + Enterprise Application
│        │         └─ Grant permissions to the Enterprise App
│        │
│        └─ NO → Do you need to manage user/group access to an app?
│           └─ YES → Use Enterprise Application to assign users/groups
```

---

## Key Relationships

### The Life Cycle

```
1. Developer → Creates APP REGISTRATION
                (defines the app's identity)
                        ↓
2. System → Automatically creates ENTERPRISE APPLICATION
                (service principal in tenant)
                        ↓
3. Admin → Configures ENTERPRISE APPLICATION
                (assigns users, sets permissions)
                        ↓
4. Users → Access the app through Enterprise Application
                (with assigned permissions)
```

### The Complete Authentication & Authorization Flow

```
┌────────────────────────────────────────────────────────────┐
│ 1. Developer Creates App Registration (Dev-time)          │
│                                                            │
│   Portal → App Registrations → New                         │
│   Result: Client ID created                               │
└────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────┐
│ 2. Developer Adds Credentials (Dev-time)                  │
│                                                            │
│   App Registration → Certificates & Secrets               │
│   → Add Client Secret                                     │
│   → App now has: Client ID + Secret                       │
│                                                            │
│   PURPOSE: Script/app can prove its identity              │
└────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────┐
│ 3. Developer Adds API Permissions (Dev-time)              │
│                                                            │
│   App Registration → API Permissions                      │
│   → Add User.ReadWrite.All                                │
│   → Add Directory.ReadWrite.All                           │
│                                                            │
│   PURPOSE: Declare what the app needs access to           │
└────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────┐
│ 4. Admin Deploys & Grants Consent (Deploy-time)           │
│                                                            │
│   Enterprise Application is automatically created         │
│   Admin → Grants admin consent                            │
│   Result: Service principal now has permissions in tenant │
│                                                            │
│   PURPOSE: App actually gets permissions to act           │
└────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────┐
│ 5. App/Script Runs & Authenticates (Runtime)              │
│                                                            │
│   Script sends: Client ID + Client Secret                 │
│   Entra ID: Validates against App Registration            │
│   Response: "Authenticated! Here's your access token"     │
│                                                            │
│   PURPOSE: Prove identity                                 │
└────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────┐
│ 6. App/Script Makes API Call & Gets Authorized (Runtime)  │
│                                                            │
│   Script sends: Access token + API request                │
│   Entra ID: Checks Enterprise Application permissions     │
│   Response: "Authorized! Here's your data"                │
│                                                            │
│   PURPOSE: Verify authority                               │
└────────────────────────────────────────────────────────────┘
```

### Visual Relationship

```
┌──────────────────────────────────────────────────────┐
│                   Your Tenant                         │
├──────────────────────────────────────────────────────┤
│                                                       │
│  ┌─────────────────────────────────────────────┐    │
│  │  App Registration                           │    │
│  │  (App's Identity Definition)                │    │
│  │  - Client ID                                │    │
│  │  - Credentials (Secrets, Certs)             │    │
│  │  - API Permissions                          │    │
│  │  - Redirect URIs                            │    │
│  └─────────────────────────────────────────────┘    │
│                        ↓                              │
│  ┌─────────────────────────────────────────────┐    │
│  │  Enterprise Application (Service Principal) │    │
│  │  (App's Presence in Your Tenant)            │    │
│  │  - User/Group Assignments                   │    │
│  │  - Roles & Permissions                      │    │
│  │  - Conditional Access Policies              │    │
│  │  - SSO Configuration                        │    │
│  └─────────────────────────────────────────────┘    │
│                        ↓                              │
│                   Users Access App                   │
│                                                       │
└──────────────────────────────────────────────────────┘
```

---

## Common Mistakes & How to Avoid Them

| Mistake | Impact | Solution |
|---------|--------|----------|
| Storing Client Secret in source code | Security breach; credentials exposed | Use Azure Key Vault or managed identities |
| Forgetting to create App Registration before requesting permissions | App has no identity; cannot authenticate | Always start with App Registration |
| Creating Enterprise App without setting user assignments | Users can't access the app even though it's registered | Assign users/groups to Enterprise App |
| Granting excessive permissions in App Registration | Security risk; app has more access than needed | Use least privilege principle |
| Confusing where to configure SSO (App Registration vs Enterprise App) | SSO doesn't work properly | Configure in Enterprise App for user-facing settings |
| Not granting admin consent to API permissions | App doesn't have permission to perform actions | Grant admin consent in App Registration or Enterprise App |
| Using hardcoded credentials instead of managed identities | Security vulnerability; credentials can leak | Use Azure Managed Identity when possible |
| Thinking Enterprise App stores credentials | Looking in wrong place for secrets | Always look in App Registration for credentials |

---

## Quick Reference: What to Use When

### For Developers

- **Need to build an app with Entra ID auth?** → App Registration
- **Need app credentials to call APIs?** → App Registration (add secrets/certificates)
- **Need to define what resources my app can access?** → App Registration (API Permissions)
- **Where are my secrets?** → App Registration → Certificates & Secrets

### For IT/Security Admins

- **Need to install a third-party app for users?** → Enterprise Application (from gallery)
- **Need to control who can access an app?** → Enterprise Application (User/Group Assignments)
- **Need to set up MFA for an app?** → Enterprise Application (Conditional Access)
- **Need to automate user provisioning?** → Enterprise Application (Provisioning settings)
- **Looking for where secrets are managed?** → NOT here - that's in App Registration

---

## Additional Resources

- [Microsoft Learn: Implement app registration strategy](https://learn.microsoft.com/en-us/training/modules/implement-app-registration/2-plan-your-line-business-registration-strategy)
- [Microsoft Entra ID App Registration Documentation](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app)
- [Service Principal in Entra ID](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals)
- [Single Sign-On in Enterprise Applications](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/add-application-portal)
- [Credentials and Certificate Management](https://learn.microsoft.com/en-us/entra/identity-platform/howto-create-service-principal-portal#option-2-create-a-new-application-secret)

---

## Summary

| | App Registration | Enterprise Application |
|---|---|---|
| **Think of it as...** | The app's passport/identity card | The app's visa/entry permit in your country |
| **Created by** | Developer (during development) | Admin (when deploying) |
| **Main question it answers** | "Who is this application?" | "Who can use this application?" |
| **Focus** | **Authentication** | **Authorization & Access Control** |
| **Manages** | App credentials, API permissions | User access, roles, policies |
| **Stores Credentials?** | YES - Client Secrets, Certificates | NO - References App Registration |
| **Authenticates?** | YES - Uses credentials to prove identity | NO - Relies on App Registration authentication |

---

## Key Differences
- Apps added through Microsoft Entra ID - App registrations are by default OIDC-based apps, while apps added through Microsoft Entra ID - Enterprise applications might use any SSO standard.
- App Registrations is where you register your applications, while Enterprise Applications is where you manage access to these applications.
- [Enterprise Applications](https://learn.microsoft.com/en-us/training/modules/plan-design-integration-of-enterprise-apps-for-sso/7-configure-pre-integrated-gallery-saas-apps)


## Contributing

If you have additional scenarios or clarifications to add, please feel free to contribute!
