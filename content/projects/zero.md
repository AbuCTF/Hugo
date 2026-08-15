---
title: "Zero"
tagline: "Run an event on free email tiers"
category: "Platform"
status: "active"
type: "platform"
description: "ZeroPool: registration, email orchestration, certificates and prizes for CTF organizers, without paying for an email service."
language: "Python"
stars: 3
tech:
  - "FastAPI"
  - "SvelteKit"
  - "PostgreSQL"
  - "Redis"
  - "ARQ"
  - "Docker"
github: "https://github.com/AbuCTF/Zero"
demo: "https://app.h7tex.com"
image: "/images/projects/zero-architecture.png"
draft: false
---

ZeroPool handles participant registration, email, certificates and prizes for an event, without spending anything on email delivery.

## The idea

Every transactional email provider has a free tier. Brevo, Mailgun, SES, a Gmail app password, a plain SMTP box. Each one is too small to run an event on. Together they aren't.

Zero treats them as one pool. Providers are sorted by priority and tried in order, and a send retries up to five times with the delay scaled by attempt number.

## Not blowing the free tiers

Rate limiting is per provider, per window, in Redis, and all four counters move in one pipeline:

```python
windows = [
    ("second", 1),
    ("minute", 60),
    ("hourly", 3600),
    ("daily", 86400),
]

pipe = self.redis.pipeline()

for window_name, ttl in windows:
    key = f"ratelimit:{provider_id}:{window_name}"
    pipe.incr(key)
    pipe.expire(key, ttl)

await pipe.execute()
```

Each provider also sits behind a circuit breaker. Three consecutive failures opens it for 300 seconds, and the failure counter itself expires after 600, so a provider that fails twice an hour never trips.

Credentials are encrypted at rest with Fernet and decrypted at send time.

## Certificates

Generated as PNG with Pillow and PDF with ReportLab, each carrying a QR code that points at a public verification endpoint. Anyone can check `/api/certificates/verify/{code}` without an account.

Participants can customise their own certificate, which is an obvious abuse vector, so migration 002 added `name_locked` and `edit_count`.

## CTFd integration

Zero talks to CTFd's `/api/v1` to provision users and teams, pull the scoreboard, and sync results. A cron job re-syncs every ten minutes. Prize rules are evaluated per event, and prizes get claimed against voucher pools uploaded as CSV.

Passwords are hashed with pbkdf2:sha256 at 600,000 iterations, which is CTFd-compatible on purpose, because the two systems share accounts.

## Background work

ARQ handles anything slow. Campaign processing runs at :00, :15, :30 and :45, CTFd sync every ten minutes, expired sessions get cleaned at 03:00 daily, old email logs on Sundays at 04:00.

```python
functions = [
    send_email_task,
    process_campaign_task,
    send_verification_email_task,
    generate_certificate_task,
    bulk_generate_certificates_task,
    sync_ctfd_results_task,
    cleanup_expired_sessions_task,
    cleanup_old_email_logs_task,
    bulk_import_participants_task,
]
```

## Stack

FastAPI with Pydantic v2, SQLAlchemy 2.0 async over asyncpg, PostgreSQL 15 and Redis 7. Frontend is SvelteKit 2 on Svelte 5 with Tailwind. Caddy terminates TLS and does the routing.

The whole thing runs on a 2 vCPU / 2GB GCP box behind Cloudflare.
