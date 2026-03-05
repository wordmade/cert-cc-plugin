---
name: certification
description: |
  Integrate with Wordmade AI Certification — an inverse CAPTCHA that proves
  agents are AI, not human. Use when: implementing agent verification, embedding
  the certification widget, solving certification challenges, or integrating the
  certification API into a site or agent workflow.
  Keywords: certification, captcha, inverse-captcha, verification, agent, AI, challenge, widget.
allowed-tools: WebFetch, Bash(curl *)
---

# Wordmade AI Certification

**Inverse CAPTCHA** — CAPTCHAs prove you're human. We prove you're *not*.

**Free for agents** — no account, no cost. Site owners pay for CertGate; agents certify free.

## Quick Integration

```
1. POST /v1/challenge/request   → Get a challenge (instruction + nonce + signature)
2. Solve the instruction        → You have 60 seconds
3. POST /v1/challenge/respond   → Submit answer, receive certificate (wmc_) + passes (wmn_)
4. Present a pass to the site   → Site verifies pass via POST /v1/siteverify
```

## Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/challenge/request` | POST | Request a challenge |
| `/v1/challenge/respond` | POST | Submit your answer |
| `/v1/nonces` | POST | Refill passes — exchange certificate for more single-use passes |
| `/v1/verify/{token}` | GET | Check your certificate |
| `/v1/siteverify` | POST | Server-side verification (for site owners) |
| `/.well-known/jwks.json` | GET | Ed25519 public key (Enterprise tier local verification) |
| `/api/v1/customers/*` | Various | Customer portal (register, login, manage sites) |
| `/api/v1/sites/*` | Various | Site management (create, configure, rotate secrets) |

## CertPass (Portable Certification)

Certify once, present passes everywhere. Omit `site_key` from the challenge request
to get a portable certificate. Then request site-bound passes for any site:

```
POST /v1/nonces  {"certificate": "eyJ...", "site_key": "sk_pub_...", "count": 5}
```

Each pass is single-use and site-bound. Certificate validity: 60 minutes. Pass validity: 10 minutes.

## Scoring

```
>= 0.8  →  Certified (certificate issued, valid 60 minutes)
0.6-0.8 →  Borderline
< 0.6   →  Rejected
```

Speed matters: under 5 seconds is ideal, under 15 is good.

## For Agents Solving Challenges

Full step-by-step with examples:
**https://certification.wordmade.world/agents.md**

**Critical:** Each challenge is unique in data AND problem type. Do NOT build pre-built
solvers or dispatch by category. Read the instruction from scratch, decode it, solve it
ad-hoc, and submit the exact answer. Echo ALL challenge fields back verbatim (especially
`expires_at`) — any modification causes HMAC signature failure. If verification fails,
read the `reason` field — it includes an actionable hint explaining what went wrong.

## Enterprise Tier: Local Verification (Ed25519)

Enterprise tier certificates include an `esig` field — an Ed25519 asymmetric signature.
Site owners on the Enterprise tier can verify certificates locally using only the public
key from `/.well-known/jwks.json`. No `secret_key` or network call needed.

Full details: **https://certification.wordmade.world/guide.md#enterprise-integration-local-validation**

## For Site Owners Integrating

Widget embedding, server-side verification, security model, enterprise validation:
**https://certification.wordmade.world/guide.md**

## Full API Reference

All endpoints, request/response formats, error codes, rate limits:
**https://certification.wordmade.world/api-reference.md**

## Claude Code Plugin

Install the certification skill as a Claude Code plugin:
**https://github.com/wordmade/cert-cc-plugin**

## Interactive Demo

Preview and customize the widget on the customer portal:
**https://certification.wordmade.world/demo**

## Request Example

```bash
# 1. Request challenge
curl -s -X POST https://certification.wordmade.world/v1/challenge/request \
  -H "Content-Type: application/json" \
  -d '{"site_key": "sk_pub_...", "action": "register"}'

# 2. Submit answer (echo all fields back + your response)
curl -s -X POST https://certification.wordmade.world/v1/challenge/respond \
  -H "Content-Type: application/json" \
  -d '{
    "challenge_id": "...",
    "nonce": "...",
    "level": 1,
    "expires_at": "...",
    "signature": "...",
    "response": "your answer here",
    "site_key": "sk_pub_...",
    "action": "register"
  }'
```

---

*Not flesh. Not bot. Certified.*

*https://certification.wordmade.world*
