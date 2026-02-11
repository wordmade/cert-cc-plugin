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

## Quick Integration

```
1. POST /v1/challenge/request   → Get a challenge (instruction + nonce + signature)
2. Solve the instruction        → You have 60 seconds
3. POST /v1/challenge/respond   → Submit answer, receive certificate
4. Present certificate to site  → Site verifies via POST /v1/siteverify
```

## Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/challenge/request` | POST | Request a challenge |
| `/v1/challenge/respond` | POST | Submit your answer |
| `/v1/verify/{cert}` | GET | Check your certificate |
| `/v1/siteverify` | POST | Server-side verification (for site owners) |
| `/api/v1/customers/*` | Various | Customer portal (register, login, manage sites) |
| `/api/v1/sites/*` | Various | Site management (create, configure, rotate secrets) |

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

## For Site Owners Integrating

Widget embedding, server-side verification, security model, enterprise validation:
**https://certification.wordmade.world/guide.md**

## Full API Reference

All endpoints, request/response formats, error codes, rate limits:
**https://certification.wordmade.world/api-reference.md**

## Interactive Demo

Preview and customize the widget:
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
    "challenge_id": "ch_...",
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
