---
title: "WebAuthn & FIDO2, Explained Without the Spec"
seoTitle: "WebAuthn and FIDO2 Explained: Passwordless Authentication Without Pass"
seoDescription: "A practical explanation of WebAuthn and FIDO2 without diving into the spec. Learn public-key credentials, challenges, authenticators, and origin binding"
datePublished: Tue Feb 10 2026 15:46:54 GMT+0000 (Coordinated Universal Time)
cuid: cmlgrxksa000402l4frs2gapq
slug: webauthn-and-fido2-explained-without-the-spec
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1770731509929/2052824b-999e-4981-b1fe-4b3c47f3c5e2.png
tags: biometrics, progressive-web-apps, web-security, application-security, webauthn, public-key-cryptgraphy, passwordless-authentication, fido2, authentication-architecture

---

If you read the WebAuthn specification end to end, you’ll come away with two thoughts:

1. This is extremely well designed.
    
2. No human should be expected to learn it this way.
    

WebAuthn didn’t appear to make logins prettier. It exists because the web needed a way to authenticate users **without shared secrets**, **without training users to detect phishing**, and **without centralizing catastrophic risk**.

This article explains what WebAuthn and FIDO2 solve, how they work at a conceptual level, and why their security properties emerge naturally from the design — not from UI tricks or user behavior.

No spec quotes. No diagrams full of acronyms. Just the moving parts that matter.

---

## The problems WebAuthn was designed to solve

WebAuthn doesn’t fix “bad passwords.”  
It eliminates the *need* for passwords entirely.

The core problems it targets are structural:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1770731657554/c2a859e1-2097-4db1-9e61-694542a59e18.png align="center")

### Shared secrets don’t scale

Passwords are secrets shared between user and server. That single fact creates:

* phishing,
    
* credential stuffing,
    
* password reuse,
    
* database breach fallout.
    

As long as the same secret can be typed and replayed, attackers will find ways to collect it.

### Servers shouldn’t need to keep login secrets

Even well-hashed passwords are still verifier secrets.  
If an attacker steals the database, they gain:

* offline attack capability,
    
* reusable material,
    
* leverage across systems.
    

WebAuthn removes this entire category of risk by design.

### Authentication should not rely on user judgment

Security systems that depend on users spotting fake URLs are already lost.

WebAuthn pushes phishing resistance into the browser and protocol layer, where it belongs. Users don’t need to “be careful.” The system refuses to cooperate with the attacker.

---

## The core idea: public-key credentials

WebAuthn is built on public-key cryptography, but you don’t need to think in equations.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1770733314294/9b8235cc-833e-40ef-8c1d-f274af420e41.png align="center")

Here’s the mental model:

* Each user registers a **credential** for a specific website.
    
* That credential is a **key pair**:
    
    * a **private key** stored securely on the user’s device,
        
    * a **public key** stored on the server.
        
* The private key **never leaves the device**.
    
* The public key is useless on its own.
    

This immediately changes the trust model:

* stealing the server database doesn’t let attackers log in,
    
* stealing one device doesn’t compromise other accounts,
    
* credentials can’t be replayed or reused elsewhere.
    

---

## Challenges and assertions: proving freshness

If the server never sees a secret, how does authentication work?

Through **challenge–response**.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1770733636644/afe20df9-909a-4dfd-b725-03dbc940309a.png align="center")

### The challenge

When a user wants to authenticate:

* the server generates a random, unpredictable challenge,
    
* the challenge is tied to the session and expires quickly,
    
* the server sends it to the client.
    

This ensures freshness.  
An old response will never work again.

### The assertion

The client asks the authenticator:

* “Sign this challenge for this site.”
    

The authenticator:

* verifies the user locally (if required),
    
* signs the challenge with the private key,
    
* returns a signed **assertion**.
    

The server:

* verifies the signature using the stored public key,
    
* confirms the challenge matches,
    
* checks counters to detect replay or cloning.
    

No secrets compared.  
No passwords transmitted.  
No reusable proof created.

---

## Platform vs roaming authenticators

Authenticators come in different forms, and WebAuthn treats them explicitly.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1770734078594/3b686595-56f7-4c60-ba2e-94030076c2ef.png align="center")

### Platform authenticators

These are built into devices:

* phone biometrics,
    
* laptop fingerprint readers,
    
* OS-level secure enclaves.
    

Characteristics:

* excellent UX,
    
* tightly integrated with the device,
    
* private keys stored in hardware-backed storage.
    

They are ideal for PWAs because:

* they feel native,
    
* they require no extra hardware,
    
* they encourage passwordless adoption.
    

### Roaming authenticators

These are external devices:

* USB security keys,
    
* NFC or Bluetooth tokens.
    

Characteristics:

* portable across devices,
    
* extremely strong isolation,
    
* ideal for high-assurance environments.
    

They solve a different problem: portability without centralization.

Well-designed systems allow **both**, because users have different constraints.

---

## User presence vs user verification

This distinction is subtle and often misunderstood.

### User presence (UP)

User presence means:

* the user performed a conscious action,
    
* such as touching a button or tapping a key.
    

It prevents silent, background authentication.

### User verification (UV)

User verification means:

* the authenticator verified *who* is using it,
    
* via biometrics or a PIN.
    

UV answers: *is this the legitimate user of this device?*  
UP answers: *did someone physically interact with the device?*

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1770737256212/d6533844-c965-4aea-ac31-4ecab832dbb8.png align="center")

WebAuthn supports both:

* some flows require UV,
    
* others accept UP only,
    
* policy decides what’s acceptable.
    

This flexibility allows systems to balance:

* security,
    
* accessibility,
    
* device capability.
    

---

## Why WebAuthn is phishing-resistant by design

This is the most important property — and it’s not accidental.

WebAuthn credentials are **bound to origin**.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1770737587402/3898f1a6-3995-4a60-9e92-07d86fa2001d.png align="center")

That means:

* a credential created for `example.com`
    
* will not work for `examp1e.com` (notice the difference)
    
* or inside an iframe on another site
    
* or on a cloned login page.
    

The browser enforces this binding.  
The user never sees the decision.

A phishing site can:

* perfectly mimic your UI,
    
* use the same text,
    
* even embed your real site visually.
    

But when it asks the browser to authenticate:

* no matching credential exists,
    
* the authenticator refuses,
    
* the attack fails silently.
    

No warning dialogs.  
No user training.  
No race against social engineering.

This is what “security by design” looks like.

---

## What WebAuthn does *not* do

Clarity here prevents bad architecture later.

WebAuthn does **not**:

* manage identities across systems,
    
* recover lost accounts,
    
* replace authorization logic,
    
* eliminate the need for backend validation,
    
* solve UX by itself.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1770738078728/8bc09df4-2769-499e-b323-fb2cc6e9f8c6.png align="center")

WebAuthn solves **one problem extremely well**:  
*How can a user prove control of a credential securely, without shared secrets, and without being phishable?*

Everything else still requires system design.

---

## WebAuthn as infrastructure, not magic

When WebAuthn works well, it feels invisible:

* a prompt,
    
* a touch,
    
* and you’re in.
    

That simplicity hides real complexity:

* cryptography,
    
* device trust,
    
* browser enforcement,
    
* careful server validation.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1770738240788/360af9ed-5cac-4783-a10a-07bcf23b9a57.png align="center")

But that complexity exists whether you manage it or not.  
WebAuthn simply exposes it in a form that can be reasoned about and secured.

In the next article, we’ll step back from the protocol and look at **how WebAuthn fits into a full PWA architecture** — including sessions, fallback paths, and real-world failure modes.

Because passwordless authentication doesn’t live in a vacuum.  
It lives inside systems built by humans, for humans, on devices that get lost.

---

## ☰ Series Navigation

### Core Series

* [Introduction](https://devpath-traveler.nguyenviettung.id.vn/introduction-to-passwordless-modern-authentication-patterns-for-pwas)
    
* [**Article 1 — Authentication is Not Login**](https://devpath-traveler.nguyenviettung.id.vn/authentication-is-not-login)
    
* [**Article 2 — What “Passwordless” Actually Means**](https://devpath-traveler.nguyenviettung.id.vn/what-passwordless-actually-means)
    
* → **Article 3 — WebAuthn & FIDO2, Explained Without the Spec**
    
* [**Article 4 — OpenID Connect as the Glue**](https://devpath-traveler.nguyenviettung.id.vn/openid-connect-as-the-glue)
    
* [**Article 5 — Designing a Passwordless-First PWA Architecture**](https://devpath-traveler.nguyenviettung.id.vn/designing-a-passwordless-first-pwa-architecture)
    
* [**Article 6 — UX and Failure Are Part of the Security Model**](https://devpath-traveler.nguyenviettung.id.vn/ux-and-failure-are-part-of-the-security-model)
    
* [**Article 7 — A Real Passwordless PWA Flow (Architecture Walkthrough)**](https://devpath-traveler.nguyenviettung.id.vn/passwordless-pwa-flow-architecture-walkthrough)
    
* [**Article 8 — Implementing WebAuthn in Practice**](https://devpath-traveler.nguyenviettung.id.vn/implementing-webauthn-in-practice)
    
* [**Article 9 — Integrating OIDC (Feide) as Fallback and Recovery**](https://devpath-traveler.nguyenviettung.id.vn/integrating-oidc-feide-as-fallback-and-recovery)
    
* [**Article 10 — What Worked, What Didn’t, What I’d Change**](https://devpath-traveler.nguyenviettung.id.vn/passwordless-what-worked-what-didnt-what-id-change)
    

### Optional Extras

* [**Why Passwordless Alone Is Not an Identity Strategy**](https://devpath-traveler.nguyenviettung.id.vn/why-passwordless-alone-is-not-an-identity-strategy)
    
* [**How Browser UX Shapes Security More Than Cryptography**](https://devpath-traveler.nguyenviettung.id.vn/how-browser-ux-shapes-security-more-than-cryptography)