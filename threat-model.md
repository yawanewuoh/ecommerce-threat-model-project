# Threat Model: E-Commerce Application

## System Overview

The application is a Django e-commerce platform deployed on a single AWS EC2 instance behind Nginx. Supabase provides managed Postgres and authentication. Cloudflare R2 stores product images and uploaded content. A secrets manager holds database credentials and API keys. A third-party payment gateway (Stripe/Paystack) handles checkout and payment processing.

![Data flow diagram with trust boundaries](./dfd.png)

Five trust boundaries were identified, each representing a crossing between zones of differing trust.

---

## 1. What is at risk? (Assets)

- Customer PII — names, addresses, emails (Supabase)
- Credentials and session tokens (Supabase auth)
- Payment data and transaction records (payment gateway + DB references)
- Product images and uploaded content (Cloudflare R2)
- Secrets themselves — DB credentials, API keys, storage credentials (secrets manager)
- Application availability (Nginx/EC2)
- Audit trail integrity (logging/monitoring)

## 2. Who could exploit it?

- Opportunistic scanners and bots probing the public boundary with no specific targeting
- A registered customer attempting to escalate privileges — access another user's order, manipulate their own cart or price
- A credential-stuffing attacker reusing leaked passwords against the auth system
- An insider or an attacker with a compromised CI credential, reaching the secrets manager or EC2 directly
- An attacker specifically targeting the payment integration, where the direct financial payoff is highest

## 3. Why would they do it?

- **Financial gain** — carding, price manipulation, theft of stored payment references
- **Data resale** — PII and credentials carry market value independent of payment access
- **Lateral access** — storage or database credentials reused as a pivot point if IAM scoping is loose
- **Low-effort opportunism** — many attacks on small e-commerce sites are untargeted; the app simply surfaces in an automated scan

## 4. How could it realistically happen?

**Boundary 1 — Internet → App (Nginx/Django)**
- Missing rate limiting enabling credential stuffing or checkout abuse
- Weak input validation enabling injection or template injection
- Session or auth logic bugs enabling account takeover

**Boundary 2 — App → Database (Supabase)**
- SQL injection if the ORM is bypassed anywhere in the codebase
- An overly broad database role turning an application-level bug into a full data breach instead of a contained one
- A leaked service-role key bypassing authentication entirely

**Boundary 3 — App → Object Storage (Cloudflare R2)**
- Unauthenticated or predictable object URLs exposing private uploaded files
- Missing upload validation allowing a malicious file disguised as a product image

**Boundary 4 — App → Secrets Manager**
- Hardcoded or logged secrets, an overly broad IAM policy, or no rotation policy — any one of these turns a single leaked key into a full-system compromise

**Boundary 5 — App → Payment Gateway**
- Trusting a client-submitted amount instead of verifying it server-side against the order record (price manipulation / parameter tampering)
- Missing webhook signature verification, allowing forged "payment succeeded" events
- Missing idempotency handling, allowing a captured webhook to be replayed

## 5. Which components carry the greatest risk?

Ranked for this system specifically:

1. **Payment gateway boundary** — direct financial impact; webhook trust is a common point of failure
2. **Database boundary (Supabase)** — the largest concentration of sensitive data behind a single set of credentials
3. **Authentication/authorization within Django** — every other boundary's security assumes this holds
4. **Secrets manager boundary** — a single leak here cascades into every other boundary

Object storage and the public-facing boundary carry real risk but a smaller blast radius than the four above.

## 6. Which security controls reduce those risks?

**Boundary 1 — Internet → App**
- TLS enforced
- Input validation on all user-supplied data
- Authentication and authorization checks on every protected route
- Rate limiting on auth and checkout endpoints
- CSRF protection (Django default — not disabled)
- Security headers: CSP, HSTS

**Boundary 2 — App → Database**
- Least-privilege database role, scoped per service rather than superuser
- Parameterized queries / ORM usage only, no raw SQL string interpolation
- Secrets management for DB credentials, not hardcoded
- Strong credential policy

**Boundary 3 — App → Object Storage**
- Private-by-default storage
- Upload validation (file type, size limits)
- Object-level access controls
- Restricted, scoped storage credentials
- Signed, expiring URLs for private object access

**Boundary 4 — App → Secrets Manager**
- IAM roles scoped to least privilege
- No hardcoded AWS credentials
- Scheduled secret rotation
- Secrets never written to logs or error tracebacks

**Boundary 5 — App → Payment Gateway**
- Server-side amount verification against the stored order — never trust a client-submitted price
- Webhook signature verification on every incoming event
- Idempotency keys / event-ID tracking to prevent replay and double-processing
- No raw card data stored (keeps the app out of PCI scope)
- TLS to the gateway

---

## Summary

The highest-value target in this system is the payment flow, followed closely by the database boundary. Both share a common root cause when compromised: trusting client-supplied data (a submitted price, a forged webhook) instead of verifying against a server-side source of truth. The controls above are designed around that principle — verify server-side, scope credentials narrowly, and assume any single boundary may eventually be probed.
