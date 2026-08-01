# ADR-0004: One registered domain, NS-delegated environment zones

Status: Accepted · July 2026

## Context

Options considered for platform DNS: a domain per environment, a prod/non-prod
domain split, or one domain with delegated environment zones. Multiple
registered domains multiply the phishing surface (every look-alike domain is a
target to defend), fragment TLS/certificate governance, and drift toward
registrar sprawl.

## Decision

One registered domain. Each environment gets a zone NS-delegated into that
environment's own GCP project (Cloud DNS), so environment teams/automation can
manage records without touching the apex.

Hostname schema: `<service-class>.<env>.<domain>`; production drops the env
segment:

- `api.example.com` (prod), `api.staging.example.com`, `api.dev.example.com`
- `api.internal.dev.example.com` — internal zones are Cloud DNS **private**
  zones per environment
- `auth.` is its own service-class hostname (IdP cookie isolation, stricter WAF)

CI/Kyverno validate that a route's hostname agrees with its `edge-config`
folder and `exposure:` field (ADR-0002).

Reference implementation edge: the public apex rides Cloudflare
(external-dns → Cloudflare for public records); private zones stay in Cloud DNS.

## Consequences

- One domain to defend, one certificate governance story.
- Environment isolation comes from zone delegation (control-plane boundary),
  not domain proliferation.
- Prod URLs are clean; non-prod URLs are self-describing.
