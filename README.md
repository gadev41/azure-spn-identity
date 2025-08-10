# azure-spn-identity
spn-study
Identity and Access Management (IAM)
What it is:
A broad concept (not a specific Microsoft product) — covers the processes, policies, and technologies for managing who can access what in an organization.

Scope:
IAM includes:

    Identity lifecycle (create, modify, delete users)

    Authentication & MFA

    Authorization & RBAC

    Auditing and compliance

In Microsoft’s world:
AD and Entra ID are both IAM systems, just for different environments (on-prem vs. cloud).

Active Directory (AD)

    What it is:
    On-premises, Windows Server-based directory service introduced in 1999.
    Manages users, groups, devices, computers, policies inside a corporate network.

    Where it runs:
    Inside your organization’s data center, typically in a domain or forest setup.

    Main use:
    Authentication & authorization for on-premises resources (file servers, printers, intranet apps).

    Protocols:
    Kerberos, NTLM, LDAP.

    Example use:
    You log into your office PC with CORP\username, and it pulls your profile and permissions from AD.


    Azure Active Directory (AAD)

    What it is:
    Cloud-based identity and access management from Microsoft, designed for Azure, Microsoft 365, and SaaS apps.
    It is not just "cloud AD" — it uses different protocols and architecture.

    Where it runs:
    Entirely in Microsoft’s cloud.

    Main use:
    Authentication & authorization for cloud apps and services.

    Protocols:
    OAuth 2.0, OpenID Connect, SAML 2.0, WS-Fed.

    Example use:
    You sign in to Office 365, Azure Portal, or any SSO-enabled SaaS app with your work email.

Microsoft Entra ID

    What it is:
    The new name for Azure Active Directory (AAD) — Microsoft rebranded AAD under the Entra product family in 2023.
    Functionally, it’s still the same service as AAD, just with a broader branding strategy that includes other identity products.

    Where it runs:
    In Microsoft’s cloud.

    Main use:
    Same as AAD — cloud identity & access management.

    Why rename?
    To unify Microsoft’s identity tools (Entra ID, Entra Permissions Management, Entra Verified ID, etc.).

    Example use:
    When you create a Service Principal, configure SSO, or manage conditional access in Azure — that’s Entra ID.

    Azure DevOps and the Azure Portal are related, but they are not the same thing — and that’s why we still use a Service Principal for CI/CD.

Here’s the breakdown:
1️⃣ Azure DevOps ≠ Azure Portal

    Azure Portal is for managing Azure resources (VMs, AKS, SQL, VNets, etc.) inside your Azure subscription.

    Azure DevOps is a separate SaaS product for project management, source control, build/release pipelines, etc. It’s not hosted in your Azure subscription.

    Even though both are “Microsoft Azure” branded services, Azure DevOps needs explicit permission to deploy into your Azure subscription — it doesn’t have it by default.

2️⃣ Why Service Principal is Needed

A Service Principal (SP) is essentially:

    A special identity in Azure AD for applications, scripts, or automation tools to authenticate and perform actions in Azure.

When you run a pipeline in Azure DevOps that deploys ARM/Bicep/Terraform, the pipeline is acting like a “robot user.”
That robot needs:

    Authentication → To prove to Azure “I’m allowed to log in”

    Authorization → To prove “I’m allowed to create/update/delete these resources”

The Service Principal provides:

    App ID (username for automation)

    Client Secret / Certificate (password/key for automation)

    RBAC Role Assignment (what the automation can do — e.g., Contributor)

3️⃣ Why Not Use My Own User Account?

    Using your personal Azure AD account in pipelines is insecure — if you leave the company, the automation breaks.

    Service Principals are non-human identities and can be scoped to the least privilege needed for the job.

    SP credentials can be stored securely in Azure DevOps Service Connections (or Key Vault).

    Azure DevOps Pipeline
      ↓
Service Connection in Azure DevOps (linked to Service Principal)
      ↓
Service Principal authenticates to Azure AD
      ↓
Azure Resource Manager (ARM/Bicep/Terraform) deploys resources


 Bottom line:
Even though Azure DevOps is part of Microsoft’s ecosystem, it’s not inherently trusted by your Azure subscription. A Service Principal is the secure, auditable way to grant it just enough permission to perform CI/CD deployments.

             ┌──────────────────────────┐
         │ Azure DevOps Pipeline    │
         │ (Build / Release)        │
         └───────────┬──────────────┘
                     │
                     │ Uses
                     │ Service Connection
                     ▼
         ┌──────────────────────────┐
         │ Service Principal (SP)    │
         │ - App ID                   │
         │ - Secret / Cert            │
         │ - RBAC Role (Contributor)  │
         └───────────┬──────────────┘
                     │
                     │ Authenticates
                     ▼
         ┌──────────────────────────┐
         │ Azure Active Directory    │
         │ (Identity Management)     │
         └───────────┬──────────────┘
                     │ Issues token
                     ▼
         ┌──────────────────────────┐
         │ Azure Resource Manager    │
         │ (Deploy ARM/Bicep/Terraform)│
         └───────────┬──────────────┘
                     │
                     ▼
        ┌───────────────────────────┐
        │ Azure Resources            │
        │ (VMs, AKS, SQL, etc.)      │
        └───────────────────────────┘

Key Points in the Diagram

    Azure DevOps lives outside your subscription — it’s a separate service.

    Service Principal acts like a robot account with permissions in your Azure subscription.

    Azure AD verifies the SP and issues a token for deployment.

    ARM takes the token and provisions resources.

    “Azure DevOps may be Microsoft, but it’s still an outsider to your subscription. A Service Principal is like a controlled visitor badge that lets it in, with exactly the right permissions for CI/CD.”

1. Service Principal (SPN) – What & How it Works

A Service Principal is an Azure AD identity (application/service account) used by applications, automation, or scripts to authenticate and access Azure resources.

Think of it as a non-human Azure user with its own credentials, permissions, and role assignments.

Key parts of a Service Principal:

    Application ID (Client ID) → The unique identifier of the SPN in Azure AD.

    Tenant ID → The Azure AD directory the SPN belongs to.

    Client Secret or Certificate → The credential the SPN uses to log in.

    Role Assignments → Define what the SPN can do (e.g., Contributor on a Resource Group).

How it works:

    Your automation tool (like Azure DevOps, Terraform, GitHub Actions) sends the SPN’s Client ID, Tenant ID, and Secret/Cert to Azure AD.

    Azure AD verifies them and returns an access token.

    The token is used to call Azure Resource Manager (ARM) or other Azure services with the assigned permissions.

2. Service Connection – What & How it Works

A Service Connection is an Azure DevOps concept that stores connection details (credentials, subscription, etc.) for external services, like Azure.

When you create an Azure Resource Manager Service Connection in Azure DevOps, you’re linking it to a Service Principal under the hood.

Key facts:

    In Azure DevOps, a Service Connection is a credential vault entry + metadata about the target system.

    For Azure, it usually contains:

        Subscription ID

        Tenant ID

        Service Principal ID (Client ID)

        Authentication method (Service Principal key or certificate)

    Used in pipelines to authenticate automatically without prompting for credentials.

How it works:

    You set up a Service Connection in Azure DevOps.

    It either creates a new Service Principal automatically (if you use "Automatic" mode) or you provide an existing SPN’s details.

    In a pipeline, when a task (like az cli, ARM template deployment, Terraform, etc.) needs to connect to Azure, Azure DevOps fetches the SPN credentials from the Service Connection and logs in.

3. How They Relate

    Service Principal = the identity in Azure AD.

    Service Connection = the way Azure DevOps knows how to use that identity.

    You can think of it as:

   Service Connection = SPN credentials + target subscription details + DevOps-friendly wrapper
Without a Service Principal → no Azure Service Connection can be made (unless using Managed Identity).
4. Best Practices
Security

    Least Privilege:
    Assign the SPN only the exact roles needed.
    Example: If your pipeline only deploys to one Resource Group, give Contributor role on that RG, not the entire subscription.

    Secrets Rotation:
    If using a Client Secret, rotate it every 3–6 months. Use Key Vault for storage.

    Certificate over Secret:
    Prefer certificate-based authentication for longer-lived, more secure credentials.

    Disable Unused Connections:
    Periodically audit Azure DevOps Service Connections and delete old/unused ones.

Governance

    Name Convention:
    spn-<project>-<env>
svccon-<project>-<env>
    Example: spn-aks-prod, svccon-aks-prod.

    Separate per Environment:
    Don’t share the same SPN/Service Connection for dev, stage, and prod — avoid blast radius.

    Centralized Creation:
    Let the cloud/DevOps admin create and control SPNs so permissions are consistent.

    Enable Approvals (ADO Feature):
    For sensitive connections, require approval before pipeline can use the Service Connection.

Pipeline Hygiene

    Don’t hardcode credentials — always use the Service Connection in pipeline YAML.

    Use Managed Identity if possible — avoids storing any secret entirely.
    Context

When you create an Azure Resource Manager Service Connection in Azure DevOps, it needs a Service Principal in Azure AD to authenticate to Azure.

Azure DevOps offers two modes for setting that up:

    Automatic app registration

    Manual app registration

The “app” here means an App Registration in Azure AD (which is basically a Service Principal + Application object).
2. Automatic App Registration

What happens:

    Azure DevOps itself will create the App Registration and Service Principal for you in your Azure AD tenant.

    It will automatically assign the necessary permissions (usually Contributor) to the target subscription/resource group.

    The client secret is generated and stored inside Azure DevOps’ Service Connection securely.

Requirements:

    You (or whoever is creating the Service Connection) must have Owner or User Access Administrator role at the subscription level in Azure.

    You must also have Application Administrator or Global Administrator role in Azure AD (or be allowed to create enterprise apps).

    Your account must be connected via Azure CLI login or Azure portal authentication pop-up during Service Connection creation.

Pros:

    Quick and easy — no manual SPN creation.

    Permissions are handled automatically.

    Less chance of human error in SPN configuration.

Cons:

    Azure DevOps creates a generic name for the app (e.g., AzureDevOps-spn-xxxx).

    Less control over naming, scoping, or credential settings.

    You might end up with multiple scattered SPNs over time if every project team uses auto mode.

    Requires high privileges for the person creating it.

3. Manual App Registration

What happens:

    You manually create the App Registration in Azure AD yourself (or ask your Azure AD admin to do it).

    You configure the Service Principal (assign roles, set secrets or certificates, control scope).

    You then copy Tenant ID, Client ID, and Client Secret/Certificate into Azure DevOps Service Connection setup.

Steps:

    Go to Azure AD → App registrations → New registration.

    Create the app, note down the Application (client) ID and Directory (tenant) ID.

    Go to Certificates & secrets, generate a client secret or upload a certificate.

    Go to Azure Role Assignments (IAM), assign the SPN the correct role (e.g., Contributor) at the right scope.

    In Azure DevOps → Service Connections → Azure Resource Manager → choose Service Principal (manual) → fill in IDs and secret.

Pros:

    Full control over naming, permissions, secret lifetime, and scope.

    Matches your naming conventions and governance policies.

    Can be pre-created and reused across DevOps orgs or projects.

    Allows separation of duties — cloud admins create SPNs, DevOps teams just use them.

Cons:

    More steps to set up.

    Easy to misconfigure roles or forget a required permission.

    Requires coordination with Azure AD administrators.

4. When to Use Which
Situation	Recommended Approach
Small team, low governance, need quick setup	Automatic app registration
Enterprise setup with strict naming, scoping, RBAC, and secret policies	Manual app registration
Admin wants to reuse SPNs across projects	Manual app registration
You lack Azure AD admin privileges	Manual (with help from AD admin)


Automatic App Registration:
Azure DevOps → Calls Azure AD API → Creates App Registration (SPN) → Assigns role → Stores secret

Manual App Registration:
Admin → Creates App Registration in Azure AD → Assigns role → Gives credentials to DevOps → DevOps stores in Service Connection



    
