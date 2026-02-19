---
title: "OpenID Connect as the Glue"
seoTitle: "OpenID Connect Explained: Why PWAs Still Need Federated Identity"
seoDescription: "Learn what OpenID Connect actually provides, how Authorization Code + PKCE works conceptually, and why passwordless PWAs still need federated identity"
datePublished: Thu Feb 12 2026 15:22:41 GMT+0000 (Coordinated Universal Time)
cuid: cmljly59k000002i83h4pe2bf
slug: openid-connect-as-the-glue
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1770820703641/6a05fe05-ac02-4de6-9d7e-64786373456f.png
tags: progressive-web-apps, web-security, oidc, openid-connect, pkce, webauthn, passwordless-authentication, federated-identity, authorization-code-flow, identity-architecture

---

If WebAuthn answers the question:

> “Can this device prove control of a credential right now?”

OpenID Connect answers a different question entirely:

> “Who is this person across systems, time, and organizations?”

Modern authentication systems fail when they confuse those two layers.

WebAuthn is about **proof of possession**.  
OpenID Connect (OIDC) is about **portable identity assertions**.

When building a Progressive Web App, especially one meant to survive device loss, multi-device usage, and organizational boundaries, OIDC becomes the connective tissue that passwordless authentication alone cannot provide.

This article explains what OIDC actually does, why PWAs still need federated identity, and how flows like Authorization Code + PKCE fit into a passwordless-first architecture.

---

## What OpenID Connect actually provides

<mark>OpenID Connect is an identity layer built on OAuth 2.0.</mark>

OAuth by itself is about *delegated authorization* — letting an app access resources on your behalf.

OIDC extends OAuth to answer:

> “Who is the authenticated user?”

It does this by issuing an **ID token**, typically a signed JSON Web Token (JWT), containing claims such as:

* subject identifier,
    
* issuer,
    
* audience,
    
* authentication time,
    
* optional profile attributes.
    

In plain terms, OIDC allows one trusted system (an Identity Provider, or IdP) to tell another system:

> “I have authenticated this user, and here is cryptographic proof.”

That’s it.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1770907391672/7a2f0320-031a-49de-a91b-6e7933fd20c1.png align="center")

OIDC does not:

* define how users authenticate internally,
    
* guarantee phishing resistance,
    
* manage device credentials,
    
* handle authorization inside your app.
    

<mark>It provides </mark> **<mark>identity assertions</mark>**<mark>, not session logic and not passwordless mechanics.</mark>

This distinction is critical.

---

## What OIDC does not provide

Because OIDC is often treated as a complete solution, it’s worth being explicit about what it does not do.

It does not:

* eliminate passwords (many IdPs still use them internally),
    
* replace device-bound authentication,
    
* prevent phishing unless combined with phishing-resistant mechanisms,
    
* manage account lifecycle inside your application.
    

<mark>OIDC is transport for identity claims.<br>It is not an authentication method in itself.</mark>

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1770907597223/8f36d01d-2164-4148-9720-2d9fc7754979.png align="center")

Think of it as a passport issued by another authority.  
It tells you who someone is — it does not determine how they proved it.

---

## Why PWAs still need federated identity

At first glance, a passwordless PWA using WebAuthn might seem self-contained.

User registers a credential.  
User signs in with a biometric.  
Done.

But real systems rarely live in isolation.

PWAs face several realities:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1770907793518/bd7dbef2-86fa-4788-8313-594c03198fd8.png align="center")

### Device loss

If a user loses their only registered device, how do they recover?

WebAuthn is intentionally device-bound. That is a strength for security — but it requires a recovery path.

Federated identity allows:

* bootstrap access,
    
* credential re-registration,
    
* account restoration without weak recovery questions.
    

### Multi-device usage

Users expect to log in from:

* phone,
    
* laptop,
    
* tablet,
    
* shared workstation.
    

WebAuthn supports multiple credentials per account — but how does the user prove ownership of the account to register a new device?

Federated identity provides a portable proof of identity across devices.

### Organizational trust

In enterprise or education environments, users already have identities managed by:

* corporate directories,
    
* institutional identity providers,
    
* centralized account governance.
    

Federated identity allows your PWA to integrate into that trust fabric instead of inventing its own.

### Lifecycle management

Identity providers often handle:

* account provisioning,
    
* deactivation,
    
* role updates,
    
* compliance requirements.
    

A passwordless-only system must reimplement these concerns or delegate them.

In most real-world architectures, delegation wins.

---

## Authorization Code + PKCE (conceptual overview)

Modern browser-based applications — including PWAs — should use the **Authorization Code flow with PKCE** when integrating with OIDC.

You do not need to memorize the spec to understand the reasoning.

The flow exists to solve a simple problem:

> How can a public client (like a browser app) securely obtain tokens without exposing secrets?

Here’s the high-level sequence:

1. The app initiates login with the IdP.
    
2. The browser redirects the user to the IdP’s authentication page.
    
3. The user authenticates there.
    
4. The IdP redirects back with an authorization code.
    
5. The app exchanges that code for tokens.
    

PKCE (Proof Key for Code Exchange) adds protection by:

* generating a one-time verifier,
    
* sending a hashed version during the initial request,
    
* proving possession of the original verifier during token exchange.
    

This prevents intercepted authorization codes from being reused.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1770908321799/bfe23842-644a-4a0a-a20b-127172c8c51e.png align="center")

The key idea is this:  
<mark>Authorization Code + PKCE prevents token theft in public clients.</mark>

It does not replace WebAuthn.  
It complements it.

---

## Using IdPs for bootstrap

One of the most elegant uses of OIDC in a passwordless-first architecture is during initial account creation.

A user may:

* authenticate via an external IdP,
    
* receive a verified identity,
    
* then register a WebAuthn credential tied to that account.
    

After that:

* WebAuthn becomes the default authentication method,
    
* OIDC remains available as fallback.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1770909202936/accc826f-be48-4e30-aa79-6ee0c76ad313.png align="center")

In this model:

* OIDC establishes identity,
    
* WebAuthn establishes device-bound proof,
    
* the two reinforce each other.
    

---

## Using IdPs for recovery

Recovery is where pure passwordless systems often reveal their fragility.

If:

* the only credential is lost,
    
* and no alternative exists,
    
* the account becomes inaccessible.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1770909337893/ab1d83ae-1675-411c-94a5-28a2fa6a7e08.png align="center")

An IdP provides a recovery path that does not require:

* weak security questions,
    
* email-only resets,
    
* or administrative overrides.
    

The system can:

1. Require OIDC login.
    
2. Validate identity via trusted external authority.
    
3. Allow new credential registration.
    
4. Invalidate old credentials.
    

This turns recovery into a structured process instead of an improvised exception.

---

## Using IdPs for portability

Federated identity also enables portability across systems.

If multiple applications trust the same IdP:

* users authenticate once,
    
* identity claims propagate,
    
* account linking becomes consistent.
    

WebAuthn credentials are bound to origins.  
OIDC identities are portable across origins.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1770909503536/f1fb8ed6-86a9-4600-929b-c8dea848628b.png align="center")

The combination gives you:

* local phishing-resistant authentication,
    
* global identity continuity.
    

That pairing is powerful.

---

## WebAuthn and OIDC are not competitors

There is a persistent misconception that passwordless authentication replaces federated identity.

It doesn’t.

They solve orthogonal problems:

WebAuthn:

* secure, phishing-resistant proof of possession,
    
* bound to origin,
    
* device-specific.
    

OIDC:

* portable identity assertion,
    
* cross-system trust,
    
* organizational integration.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1770909649937/9ffdce4e-4663-44db-bda9-4423dcf3af74.png align="center")

When combined thoughtfully:

* WebAuthn becomes the fast path,
    
* OIDC becomes the resilient path,
    
* the user experience remains smooth,
    
* the architecture remains robust.
    

---

## OIDC as architectural glue

<mark>If WebAuthn is the lock on the door, OIDC is the passport system.</mark>

WebAuthn ensures:

* the right key opens the right lock.
    

OIDC ensures:

* the person behind the key is recognized across contexts.
    

PWAs that attempt to eliminate federation entirely often rediscover its necessity the hard way — during device loss, enterprise integration, or compliance review.

The mature posture is not choosing between passwordless and federation.

It is designing a system where:

* passwordless handles everyday authentication,
    
* federation handles identity continuity,
    
* and neither is forced to solve the other’s problems.
    

In the next article, we move from protocols to architecture:  
how WebAuthn, OIDC, sessions, storage, and fallback flows combine inside a real PWA authentication system.

Because theory becomes meaningful only when it survives implementation.

---

## ☰ Series Navigation

### Core Series

* [Introduction](https://devpath-traveler.nguyenviettung.id.vn/introduction-to-passwordless-modern-authentication-patterns-for-pwas)
    
* [**Article 1 — Authentication is Not Login**](https://devpath-traveler.nguyenviettung.id.vn/authentication-is-not-login)
    
* [**Article 2 — What “Passwordless” Actually Means**](https://devpath-traveler.nguyenviettung.id.vn/what-passwordless-actually-means)
    
* [**Article 3 — WebAuthn & FIDO2, Explained Without the Spec**](https://devpath-traveler.nguyenviettung.id.vn/webauthn-and-fido2-explained-without-the-spec)
    
* → **Article 4 — OpenID Connect as the Glue**
    
* [**Article 5 — Designing a Passwordless-First PWA Architecture**](https://devpath-traveler.nguyenviettung.id.vn/designing-a-passwordless-first-pwa-architecture)
    
* [**Article 6 — UX and Failure Are Part of the Security Model**](https://devpath-traveler.nguyenviettung.id.vn/ux-and-failure-are-part-of-the-security-model)
    
* [**Article 7 — A Real Passwordless PWA Flow (Architecture Walkthrough)**](https://devpath-traveler.nguyenviettung.id.vn/passwordless-pwa-flow-architecture-walkthrough)
    
* [**Article 8 — Implementing WebAuthn in Practice**](https://devpath-traveler.nguyenviettung.id.vn/implementing-webauthn-in-practice)
    
* [**Article 9 — Integrating OIDC (Feide) as Fallback and Recovery**](https://devpath-traveler.nguyenviettung.id.vn/integrating-oidc-feide-as-fallback-and-recovery)
    
* [**Article 10 — What Worked, What Didn’t, What I’d Change**](https://devpath-traveler.nguyenviettung.id.vn/passwordless-what-worked-what-didnt-what-id-change)
    

### Optional Extras

* [**Why Passwordless Alone Is Not an Identity Strategy**](https://devpath-traveler.nguyenviettung.id.vn/why-passwordless-alone-is-not-an-identity-strategy)
    
* [**How Browser UX Shapes Security More Than Cryptography**](https://devpath-traveler.nguyenviettung.id.vn/how-browser-ux-shapes-security-more-than-cryptography)