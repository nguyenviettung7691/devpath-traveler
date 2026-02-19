---
title: "Implementing WebAuthn in Practice"
seoTitle: "Implementing WebAuthn in ASP.NET Core with VueJS, SQL Server, FIDO2"
seoDescription: "A practical guide to implementing WebAuthn (FIDO2) using ASP.NET Core, VueJS, SQL Server, and fido2-net-lib."
datePublished: Tue Feb 17 2026 08:38:24 GMT+0000 (Coordinated Universal Time)
cuid: cmlqcphf7000t02l7gr2abre7
slug: implementing-webauthn-in-practice
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1771316072943/5198e555-e9f7-469e-a948-0182afe23005.png
tags: sql-server, vuejs, aspnet-core, progressive-web-apps, application-security, webauthn, public-key-cryptgraphy, passwordless-authentication, fido2, authentication-architecture, fido2-net-lib

---

WebAuthn looks deceptively simple at a high level:

* Generate challenge
    
* Call browser API
    
* Verify signature
    
* Done
    

In practice, it is not that simple.

WebAuthn is cryptographically elegant but operationally unforgiving.  
Small mistakes create subtle security gaps or inexplicable failures.

This article walks through:

* The tooling used
    
* The data model design
    
* Real code from ASP.NET Core + VueJS
    
* Common pitfalls
    
* And what surprised me during implementation
    

### Disclaimer

*This article describes architectural patterns and technical approaches based on a real-world implementation. All examples, code snippets, and flow descriptions have been generalized and simplified for educational purposes. No proprietary business logic, confidential configurations, credentials, or organization-specific details are disclosed. The focus is strictly on publicly documented standards (WebAuthn, OIDC) and implementation patterns within a standard VueJS + ASP.NET Core + SQL Server stack.*

---

# Tooling Used

## Backend: `fido2-net-lib`

For .NET Core, [`fido2-net-lib`](https://github.com/passwordless-lib/fido2-net-lib) is one of the most mature and spec-compliant WebAuthn libraries available.

It handles:

* Challenge generation
    
* Attestation verification
    
* Assertion verification
    
* Counter validation
    
* Origin validation
    
* Credential parsing
    

Initialization:

```csharp
var fido2 = new Fido2(new Fido2Configuration
{
    ServerDomain = "yourdomain.com",
    ServerName = "Your App",
    Origin = "https://yourdomain.com"
});
```

The important realization:

The library handles cryptography —  
You must handle state.

## Frontend: Native WebAuthn API

In VueJS, no heavy library was required.  
The browser already implements [WebAuthn](https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API).

Registration:

```javascript
const credential = await navigator.credentials.create({
  publicKey: options
});
```

Authentication:

```javascript
const assertion = await navigator.credentials.get({
  publicKey: options
});
```

However:

<mark>You must convert Base64URL fields correctly between server and client.</mark>

This is one of the first places things break.

---

# Data Model Design (SQL Server)

This is where real decisions matter.

A WebAuthn credential is not just an ID.

Here’s the simplified SQL model:

```sql
CREATE TABLE WebAuthnCredentials (
    Id UNIQUEIDENTIFIER PRIMARY KEY,
    UserId UNIQUEIDENTIFIER NOT NULL,
    CredentialId VARBINARY(MAX) NOT NULL,
    PublicKey VARBINARY(MAX) NOT NULL,
    SignatureCounter BIGINT NOT NULL,
    CreatedAt DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);
```

### Why VARBINARY?

Because:

* Credential IDs are binary.
    
* Public keys are binary (COSE format).
    
* Storing them as strings introduces encoding risk.
    

### Why store SignatureCounter?

The counter protects against cloned authenticators.

If the new counter ≤ stored counter, something is wrong.

WebAuthn security is incomplete without counter tracking.

---

# Registration Flow (Real Implementation)

## Step 1: Generate Options

```csharp
[HttpPost("register-options")]
public IActionResult RegisterOptions()
{
    var user = GetCurrentUser();

    var options = _fido2.RequestNewCredential(
        new Fido2User
        {
            Id = Encoding.UTF8.GetBytes(user.Id.ToString()),
            Name = user.Email,
            DisplayName = user.Email
        },
        new List<PublicKeyCredentialDescriptor>(),
        AuthenticatorSelection.Default,
        AttestationConveyancePreference.None
    );

    HttpContext.Session.SetString("fido2.attestationChallenge", options.Challenge);

    return Ok(options);
}
```

Notice:

* Challenge is stored server-side.
    
* Attestation preference set to `None` (privacy-friendly).
    
* No credentials excluded in this example.
    

## Step 2: Verify Attestation

```csharp
[HttpPost("verify-registration")]
public async Task<IActionResult> VerifyRegistration([FromBody] AuthenticatorAttestationRawResponse attestation)
{
    var challenge = HttpContext.Session.GetString("fido2.attestationChallenge");

    var result = await _fido2.MakeNewCredentialAsync(
        attestation,
        new List<PublicKeyCredentialDescriptor>(),
        (args) => args.Challenge == challenge
    );

    var credential = new WebAuthnCredential
    {
        UserId = GetCurrentUserId(),
        CredentialId = result.Result.CredentialId,
        PublicKey = result.Result.PublicKey,
        SignatureCounter = result.Result.Counter
    };

    _db.WebAuthnCredentials.Add(credential);
    await _db.SaveChangesAsync();

    return Ok();
}
```

Key insight:

<mark>The challenge validator delegate must explicitly check equality.</mark>

Do not assume the library does that for you.

---

# Authentication Flow (Assertion)

## Generate Assertion Options

```csharp
var options = _fido2.GetAssertionOptions(
    storedCredentials,
    UserVerificationRequirement.Preferred
);

HttpContext.Session.SetString("fido2.challenge", options.Challenge);
```

## Verify Assertion

```csharp
var result = await _fido2.MakeAssertionAsync(
    clientResponse,
    storedCredential.PublicKey,
    storedCredential.SignatureCounter,
    args => args.Challenge == challenge
);

storedCredential.SignatureCounter = result.Counter;
await _db.SaveChangesAsync();
```

The counter update is not optional.

It is part of replay protection.

---

# Common Implementation Pitfalls

## 1\. Base64URL encoding mismatches

Browser returns ArrayBuffers.  
ASP.NET expects byte arrays.

If encoding conversion is inconsistent, verification fails silently.

Solution: Use consistent Base64URL encoding utilities.

### Example

```javascript
const assertion = await navigator.credentials.get({ publicKey: options });

await fetch("/api/auth/verify-webauthn", {
  method: "POST",
  body: JSON.stringify(assertion)
});
```

Problem: `assertion.rawId` is an ArrayBuffer — not Base64URL.

Explicit conversion helpers:

```javascript
function bufferToBase64Url(buffer) {
  return btoa(String.fromCharCode(...new Uint8Array(buffer)))
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=/g, '');
}
```

## 2\. Forgetting challenge persistence

If the challenge:

* is not stored,
    
* or stored per user incorrectly,
    
* or overwritten in concurrent requests,
    

verification fails.

Challenge must be:

* short-lived,
    
* per session,
    
* non-reusable.
    

### Example

```csharp
HttpContext.Session.SetString("challenge", options.Challenge);
```

Then later:

```csharp
var challenge = HttpContext.Session.GetString("fido2.challenge");
```

By using 2 different keys would introduce this bug:

```csharp
Fido2VerificationException: Challenge mismatch
```

Or:

```csharp
Fido2VerificationException: Invalid challenge.
```

## 3\. Not validating origin

Origin mismatch is a common deployment issue.

If your production URL differs from development configuration, authentication breaks.

### Example:

Your production:

```csharp
https://app.yourdomain.com
```

But config says:

```csharp
Origin = "https://yourdomain.com"
```

Subdomain mismatch would lead to this error:

```csharp
Fido2VerificationException: Invalid origin
```

Or:

```csharp
Origin https://app.yourdomain.com does not match expected https://yourdomain.com
```

## 4\. Counter mishandling

Some authenticators:

* return 0 initially.
    
* do not increment as expected.
    

Your logic must handle legitimate zero counters.

Rejecting zero blindly causes user lockout.

### Example

Authenticator returns:

```javascript
counter = 0
```

Stored counter also:

```sql
0
```

Your logic:

```csharp
if (result.Counter <= storedCounter)
{
    throw new SecurityException("Possible cloned authenticator");
}
```

Immediate lockout and return error:

```csharp
Fido2VerificationException: Signature counter did not increase.
```

Or your own thrown exception:

```csharp
Possible cloned authenticator detected.
```

Correct logic: Only enforce monotonicity when counter &gt; 0.

## 5\. Misunderstanding attestation

Attestation verifies device manufacturer.

Most applications do not need this.

Setting `AttestationConveyancePreference.None`:

* avoids privacy concerns,
    
* reduces complexity,
    
* avoids metadata verification headaches.
    

### Example:

You enable:

```csharp
AttestationConveyancePreference.Direct
```

Now browser returns full attestation.

But you don’t validate metadata, which would returns:

```csharp
Fido2VerificationException: Attestation format not supported
```

Or:

```csharp
Fido2VerificationException: No metadata service configured
```

## Bonus: Browser-Side Errors

### User Cancels

```javascript
DOMException: The operation was aborted.
```

### Not Allowed

```javascript
DOMException: The user aborted a request.
```

### Unsupported Platform

```javascript
NotSupportedError: The operation is not supported.
```

These are not backend problems — but your UX must handle them gracefully.

---

# What Surprised Me During Implementation

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1771317391660/baa24c64-7482-4f3e-aea6-d4291af80a6b.png align="center")

## 1\. How much state management matters

The cryptography is handled by the library.

The complexity lives in:

* challenge storage,
    
* session lifecycle,
    
* device registration state,
    
* error branching.
    

WebAuthn is less about math and more about disciplined state handling.

## 2\. Browser inconsistencies

Different browsers:

* format errors differently,
    
* handle cancellation differently,
    
* vary in UI timing.
    

Your retry UX must account for that.

## 3\. The importance of fallback

The first time a device:

* failed biometric recognition,
    
* or returned unexpected counter values,
    

I realized:

Passwordless-only systems are fragile.

Fallback is not optional.

## 4\. Offline expectations vs reality

Because this is a PWA, users assume:

“It’s installed. It should just work.”

But WebAuthn requires:

* live challenge from server,
    
* real-time verification.
    

Offline login is not true authentication.

Designing expectations around that was essential.

## 5\. The psychological difference

Once implemented properly:

Users stopped typing passwords.

They trusted the system more.

That was not because of UI polish.

It was because:

* no secrets were transmitted,
    
* no reset emails were needed,
    
* no password rules existed.
    

Security felt natural.

That is rare.

---

# Final Reflection

Implementing WebAuthn is not:

* copying code from documentation,
    
* adding biometric login,
    
* or flipping a feature flag.
    

It is:

* modeling credentials correctly,
    
* handling state carefully,
    
* validating challenges strictly,
    
* updating counters reliably,
    
* integrating session management securely.
    

It is architecture expressed through code.

In the next article, we’ll examine the integration of Feide OIDC in more depth — including account linking, token validation, and how federated identity interacts with my passwordless credential lifecycle.

Because WebAuthn proves possession.

Federation proves identity continuity.

Both are required for resilient systems.

---

## ☰ Series Navigation

### Core Series

* [Introduction](https://devpath-traveler.nguyenviettung.id.vn/introduction-to-passwordless-modern-authentication-patterns-for-pwas)
    
* [**Article 1 — Authentication is Not Login**](https://devpath-traveler.nguyenviettung.id.vn/authentication-is-not-login)
    
* [**Article 2 — What “Passwordless” Actually Means**](https://devpath-traveler.nguyenviettung.id.vn/what-passwordless-actually-means)
    
* [**Article 3 — WebAuthn & FIDO2, Explained Without the Spec**](https://devpath-traveler.nguyenviettung.id.vn/webauthn-and-fido2-explained-without-the-spec)
    
* [**Article 4 — OpenID Connect as the Glue**](https://devpath-traveler.nguyenviettung.id.vn/openid-connect-as-the-glue)
    
* [**Article 5 — Designing a Passwordless-First PWA Architecture**](https://devpath-traveler.nguyenviettung.id.vn/designing-a-passwordless-first-pwa-architecture)
    
* [**Article 6 — UX and Failure Are Part of the Security Model**](https://devpath-traveler.nguyenviettung.id.vn/ux-and-failure-are-part-of-the-security-model)
    
* [**Article 7 — A Real Passwordless PWA Flow (Architecture Walkthrough)**](https://devpath-traveler.nguyenviettung.id.vn/passwordless-pwa-flow-architecture-walkthrough)
    
* → **Article 8 — Implementing WebAuthn in Practice**
    
* [**Article 9 — Integrating OIDC (Feide) as Fallback and Recovery**](https://devpath-traveler.nguyenviettung.id.vn/integrating-oidc-feide-as-fallback-and-recovery)
    
* [**Article 10 — What Worked, What Didn’t, What I’d Change**](https://devpath-traveler.nguyenviettung.id.vn/passwordless-what-worked-what-didnt-what-id-change)
    

### Optional Extras

* [**Why Passwordless Alone Is Not an Identity Strategy**](https://devpath-traveler.nguyenviettung.id.vn/why-passwordless-alone-is-not-an-identity-strategy)
    
* [**How Browser UX Shapes Security More Than Cryptography**](https://devpath-traveler.nguyenviettung.id.vn/how-browser-ux-shapes-security-more-than-cryptography)