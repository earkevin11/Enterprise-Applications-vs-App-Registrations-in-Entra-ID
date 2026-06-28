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
| **Manages** | <b>App credentials, certificates, API permissions </b> | Who can access the app, what roles they have |

---

## Detailed Explanation

### App Registration

#### **Purpose**
An **App Registration** is the first step in integrating an application with Entra ID. It registers your application with the identity platform, allowing it to authenticate users and call protected APIs.

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
│  │  - Credentials                              │    │
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
| Forgetting to create App Registration before requesting permissions | App has no identity; cannot authenticate | Always start with App Registration |
| Creating Enterprise App without setting user assignments | Users can't access the app even though it's registered | Assign users/groups to Enterprise App |
| Granting excessive permissions in App Registration | Security risk; app has more access than needed | Use least privilege principle |
| Confusing where to configure SSO (App Registration vs Enterprise App) | SSO doesn't work properly | Configure in Enterprise App for user-facing settings |
| Not granting admin consent to API permissions | App doesn't have permission to perform actions | Grant admin consent in App Registration or Enterprise App |
| Using hardcoded credentials instead of managed identities | Security vulnerability; credentials can leak | Use Azure Managed Identity when possible |

---

## Quick Reference: What to Use When

### For Developers

- **Need to build an app with Entra ID auth?** → App Registration
- **Need app credentials to call APIs?** → App Registration (add secrets/certificates)
- **Need to define what resources my app can access?** → App Registration (API Permissions)

### For IT/Security Admins

- **Need to install a third-party app for users?** → Enterprise Application (from gallery)
- **Need to control who can access an app?** → Enterprise Application (User/Group Assignments)
- **Need to set up MFA for an app?** → Enterprise Application (Conditional Access)
- **Need to automate user provisioning?** → Enterprise Application (Provisioning settings)

---

## Additional Resources

- [Microsoft Learn: Implement app registration strategy](https://learn.microsoft.com/en-us/training/modules/implement-app-registration/2-plan-your-line-business-registration-strategy)
- [Microsoft Entra ID App Registration Documentation](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app)
- [Service Principal in Entra ID](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals)
- [Single Sign-On in Enterprise Applications](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/add-application-portal)

---

## Summary

| | App Registration | Enterprise Application |
|---|---|---|
| **Think of it as...** | The app's passport/identity card | The app's visa/entry permit in your country |
| **Created by** | Developer (during development) | Admin (when deploying) |
| **Main question it answers** | "Who is this application?" | "Who can use this application?" |
| **Focus** | **Authentication** | **Authorization & Access Control** |
| **Manages** | App credentials, API permissions | User access, roles, policies |

---

## Contributing

If you have additional scenarios or clarifications to add, please feel free to contribute!
