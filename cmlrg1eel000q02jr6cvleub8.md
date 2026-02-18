---
title: "Integrating OIDC (Feide) as Fallback and Recovery"
seoTitle: "Integrating Feide OIDC with WebAuthn in ASP.NET Core Passwordless PWA"
seoDescription: "Learn how to integrate Feide OpenID Connect as fallback and recovery in a passwordless PWA using ASP.NET Core, VueJS, SQL Server, and WebAuthn (FIDO2)."
datePublished: Wed Feb 18 2026 02:59:25 GMT+0000 (Coordinated Universal Time)
cuid: cmlrg1eel000q02jr6cvleub8
slug: integrating-oidc-feide-as-fallback-and-recovery
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1771383125974/0ea35be2-6c8d-465b-94b8-72ef335d3470.png
tags: vuejs, aspnet-core, progressive-web-apps, oidc, openid-connect, webauthn, passwordless-authentication, account-linking, fido2, federated-identity, authentication-architecture, feide

---

WebAuthn gave us phishing-resistant, device-bound authentication.  
But devices get lost. Browsers reset. Users switch laptops. Institutions manage identities centrally.

That’s where **OIDC (Feide)** enters — not as a competitor to passwordless, but as structural support.

This article walks through my real implementation:

* **Frontend:** VueJS PWA
    
* **Backend:** ASP.NET Core
    
* **Database:** SQL Server
    
* **Passwordless:** fido2-net-lib
    
* **Federation:** OpenID Connect (Feide)
    
* **Session:** HTTP-only cookie
    

And we’ll focus on four things:

1. What can Feide bring to the table
    
2. How OIDC fits without undermining WebAuthn
    
3. Security boundaries between IdP and my system
    
4. Account linking in practice
    

### Disclaimer

*This article describes architectural patterns and technical approaches based on a real-world implementation. All examples, code snippets, and flow descriptions have been generalized and simplified for educational purposes. No proprietary business logic, confidential configurations, credentials, or organization-specific details are disclosed. The focus is strictly on publicly documented standards (WebAuthn, OIDC) and implementation patterns within a standard VueJS + ASP.NET Core + SQL Server stack.*

---

# What can Feide bring to the table

[Feide](https://docs.feide.no/general/feide_overview.html) is widely used in Norwegian education and research sectors. That matters for three reasons:

### 1️⃣ Institutional Identity Already Exists

Users already have:

* A managed identity
    
* Centralized credential lifecycle
    
* Organizational trust
    

Recreating identity inside my PWA would be redundant and weaker.

### 2️⃣ Compliance & Governance

Institutional IdPs typically enforce:

* MFA policies
    
* Password strength
    
* Account revocation
    
* Auditing
    

By integrating Feide, my system inherits that upstream assurance without storing passwords.

### 3️⃣ Recovery and Bootstrap

WebAuthn is device-bound.

Feide provides:

* Cross-device identity continuity
    
* Secure account recovery
    
* Bootstrap trust for new devices
    

---

# How OIDC Fits Without Undermining Passwordless

The common fear:

> “If I add OIDC fallback, doesn’t that weaken passwordless?”

Only if fallback is careless.

My architecture enforces this model:

* WebAuthn = primary authentication
    
* Feide OIDC = bootstrap + recovery
    
* HTTP-only cookie = session integrity
    
* SQL Server = credential persistence
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1771382401976/436634c8-3781-4665-98b2-a3fe4b13ec9c.png align="center")

Feide does not authenticate users inside my system directly.

Feide asserts identity.

WebAuthn proves device possession.

Those are different trust layers.

---

# Real OIDC Integration (ASP.NET Core)

My implemented Authorization Code flow with PKCE.

### OIDC Configuration

```csharp
services.AddAuthentication(options =>
{
    options.DefaultScheme = "Cookies";
    options.DefaultChallengeScheme = "oidc";
})
.AddCookie("Cookies", options =>
{
    options.Cookie.HttpOnly = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.SameSite = SameSiteMode.Lax;
})
.AddOpenIdConnect("oidc", options =>
{
    options.Authority = "https://auth.feide.no";
    options.ClientId = Configuration["Feide:ClientId"];
    options.ClientSecret = Configuration["Feide:ClientSecret"];
    options.ResponseType = "code";
    options.SaveTokens = false;
    options.GetClaimsFromUserInfoEndpoint = true;

    options.Scope.Add("openid");
    options.Scope.Add("profile");
    options.Scope.Add("email");

    options.TokenValidationParameters.NameClaimType = "name";
});
```

Important detail:

```csharp
options.SaveTokens = false;
```

You do not store IdP tokens in the browser.

You convert identity into a server-controlled session.

---

# OIDC Callback Flow

```csharp
[HttpGet("callback")]
public async Task<IActionResult> Callback()
{
    var authenticateResult = await HttpContext.AuthenticateAsync("oidc");

    if (!authenticateResult.Succeeded)
        return Unauthorized();

    var externalUserId = authenticateResult.Principal.FindFirst("sub")?.Value;

    var user = await FindOrCreateUser(externalUserId);

    SignInUser(user.Id);

    if (!user.WebAuthnCredentials.Any())
        return Redirect("/enable-passwordless");

    return Redirect("/dashboard");
}
```

This is critical:

* Feide proves identity.
    
* The system maps that identity to internal user record.
    
* The system issues session cookie.
    

The IdP does not create sessions in my system.

---

# Security Boundaries Between OIDC and My System

Understanding boundaries prevents architectural confusion.

## What OIDC Is Responsible For

* Authenticating the user upstream
    
* Issuing ID tokens
    
* Managing institutional identity lifecycle
    
* Enforcing upstream MFA policies
    

## What My System Is Responsible For

* Mapping external identity (`sub`) to internal user
    
* Managing WebAuthn credentials
    
* Verifying FIDO2 assertions
    
* Issuing and invalidating session cookies
    
* Authorization within my application
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1771382763546/5a5fa933-6a2e-48b3-a819-28f7dc85eb4b.png align="center")

OIDC is not trusted to:

* Authorize application actions
    
* Manage WebAuthn devices
    
* Maintain the session integrity
    

Trust is layered, not delegated.

---

# Account Linking Considerations

This is where real complexity lives.

OIDC provides:

```javascript
{
  "sub": "abcd1234",
  "email": "user@example.edu"
}
```

But what if:

* Email changes?
    
* User logs in with different institutional account?
    
* Duplicate local account exists?
    

You must choose a stable linking strategy.

## Recommended Linking Model

Use `sub` as the primary external identifier.

Database model:

```sql
CREATE TABLE ExternalLogins (
    Id UNIQUEIDENTIFIER PRIMARY KEY,
    UserId UNIQUEIDENTIFIER NOT NULL,
    Provider NVARCHAR(50) NOT NULL,
    ExternalSubject NVARCHAR(255) NOT NULL
);
```

Mapping logic:

```csharp
var externalLogin = await _db.ExternalLogins
    .FirstOrDefaultAsync(x =>
        x.Provider == "Feide" &&
        x.ExternalSubject == externalUserId);

if (externalLogin == null)
{
    // First login → create link
    var user = CreateNewUser();
    _db.ExternalLogins.Add(new ExternalLogin {
        UserId = user.Id,
        Provider = "Feide",
        ExternalSubject = externalUserId
    });
}
```

Never rely solely on email for linking.

Emails change. `sub` should not.

---

# Recovery Flow Using Feide

Lost device scenario:

1. User clicks “Login with Feide”
    
2. OIDC completes
    
3. Identity verified
    
4. System invalidates old WebAuthn credentials
    
5. User registers new credential
    

Example revocation:

```csharp
_db.WebAuthnCredentials.RemoveRange(user.WebAuthnCredentials);
await _db.SaveChangesAsync();
```

Then redirect to registration.

Recovery is structured. Not improvised.

---

# Why This Does Not Undermine Passwordless

Weak fallback undermines security when:

* It bypasses verification
    
* It skips policy
    
* It exists only as emergency shortcut
    

My implementation ensures:

* OIDC must complete successfully
    
* Session is server-issued
    
* WebAuthn remains primary method
    
* Registration after OIDC is explicit
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1771383053155/e3ce863b-1f1d-4178-96bf-a0e6de19693f.png align="center")

This maintains assurance.

---

# VueJS PWA Integration

From frontend:

```javascript
function loginWithFeide() {
  window.location.href = "/api/auth/feide-login";
}
```

No tokens stored client-side.  
No JWT in localStorage.  
No client-managed identity state.

The PWA only reacts to session cookie.

This keeps attack surface small.

---

# What This Architecture Achieves

By combining:

* WebAuthn (device-bound proof)
    
* Feide OIDC (identity continuity)
    
* SQL Server (credential persistence)
    
* HTTP-only cookies (session security)
    

You achieve:

* Phishing resistance
    
* Device lifecycle resilience
    
* Institutional identity integration
    
* Controlled fallback
    
* Clear trust boundaries
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1771383362636/e4304380-fee7-4b23-b891-0582eb1f9de5.png align="center")

Most importantly:

You avoid false dichotomy.

This is not:

“Passwordless vs Federation.”

It is:

“Passwordless for authentication. Federation for identity continuity.”

---

# Final Reflection

Integrating OIDC did not weaken the system.

It completed it.

WebAuthn without federation is brittle.  
Federation without WebAuthn is phishable.

Together, they form a layered trust architecture.

In the next article, we’ll examine operational lessons learned after deploying this combined system — including monitoring, auditing, and real-world behavioral patterns that only surface after production traffic begins.

Because authentication design doesn’t end at implementation.

It evolves under pressure.

---

## ☰ Series Navigation

### Core Series

* Introduction
    
* **Article 1 — Authentication is Not Login**
    
* **Article 2 — What “Passwordless” Actually Means**
    
* **Article 3 — WebAuthn & FIDO2, Explained Without the Spec**
    
* **Article 4 — OpenID Connect as the Glue**
    
* **Article 5 — Designing a Passwordless-First PWA Architecture**
    
* **Article 6 — UX and Failure Are Part of the Security Model**
    
* **Article 7 — A Real Passwordless PWA Flow (Architecture Walkthrough)**
    
* **Article 8 — Implementing WebAuthn in Practice**
    
* → **Article 9 — Integrating OIDC (Feide) as Fallback and Recovery**
    
* **Article 10 — What Worked, What Didn’t, What I’d Change**
    

### Optional Extras

* **Why Passwordless Alone Is Not an Identity Strategy**
    
* **How Browser UX Shapes Security More Than Cryptography**