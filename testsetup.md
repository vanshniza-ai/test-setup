# Sandbox Refresh Automation — Setup Guide

## High-level flow

The workflow runs in **GitHub Actions**. It installs the Salesforce CLI (`sf`), authenticates to the org using **JWT bearer flow**, then applies **retrieve → update → deploy** (and a few specialized paths) so the refreshed sandbox is safe and pointed at non-production endpoints.

## Post Refresh Configuration Flow

```mermaid
flowchart LR
  subgraph GitHub["GitHub repository"]
    W[".github/workflows/post-refresh-configuration.yml"]
    S["Scripts (updateNamed.js etc.)"]
    C["eca/server.crt (public certificate in repo)"]
  end

  subgraph Run["Actions runner"]
    SF["Salesforce CLI (sf command)"]
  end

  subgraph SFDC["Salesforce Sandbox Org"]
    ECA["External Client App (JWT login)"]
    ORG["Sandbox org (metadata & settings)"]
  end

  W --> SF
  S --> SF
  SF -->|"Uses JWT authentication"| ECA
  ECA -->|"Allows secure login"| SF
  SF -->|"Retrieve & deploy metadata + configurations"| ORG
  C -.->|"Uploaded/linked certificate"| ECA

**Authentication (JWT) in plain terms**

1. In Salesforce, an **External Client App** has **JWT Bearer Flow** enabled and the **public** certificate uploaded (see [ECA and `server.crt`](#eca-and-servercrt-external-client-app)).
2. In GitHub, the **private** key is stored as a secret (for example `SF_JWT_KEY`). The workflow writes it to a file (for example `server.key`) with restricted permissions.
3. The runner runs `sf org login jwt` (or equivalent) using the **consumer key** (from the ECA), **integration user username**, **private key file**, and **login/instance URL** (for sandboxes, typically `https://test.salesforce.com`).
4. Salesforce validates the JWT against the **public** cert on the app. The CLI then uses the authenticated session for **metadata retrieve/deploy**, **REST**, and **Tooling** calls.

---





```mermaid
%%{init: {'flowchart': {'nodeSpacing': 22, 'rankSpacing': 36}}}%%
flowchart TD
    A[GitHub Actions Trigger<br/>workflow_dispatch]
    C[Install Dependencies]

    subgraph auth [Authentication]
        D[JWT Authentication to Salesforce Sandbox]
    end

    E[Retrieve Metadata<br/>force-app]
    F{Update Steps}

    F1[Deactivate Communities]
    F2[Delete SAML SSO Configurations]
    F3[Enable Salesforce Credentials Login]
    F4[Set Max Login Attempts to 10]
    F5[Disable Outlook Integration]
    F6[Disable Identity Provider]
    F7[Enable Lightning Experience S1 Banner]
    F8[Deactivate Platform Events]
    F9[Update Trusted Domains for iFrame]
    F10[Disable Transaction Security Policies]
    F11[Deactivate Box Trigger]
    F12[Update Remote Site Settings]
    F13[Deactivate SCA_Exceptions Flow]
    F14[Update Email Relay Usernames to Non-Prod]
    F15[Update Named Credential URLs to Non-Prod]
    F16[Remove All Certificates]
    F17[Uninstall the smarsh managed package]

    G{Consolidate Metadata Deploy}
    H[Generate Summary<br/>with ✅/❌ status]
    I[Done - Sandbox is now safe & configured]

    A --> C --> D --> E --> F

    F --> F1 & F2 & F3 & F4 & F5 & F6 & F7 & F8 & F9 & F10 & F11 & F12 & F13 & F14 & F15 & F16 & F17
    F1 & F2 & F3 & F4 & F5 & F6 & F7 & F8 & F9 & F10 & F11 & F12 & F13 & F14 & F15 & F16 & F17 --> G

    G --> H --> I

    style auth fill:#FFF9C4,stroke:#F9A825,color:#000
```
