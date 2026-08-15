---
title: "MFAuth"
tagline: "TOTP, written twice"
category: "Security"
status: "stable"
type: "library"
description: "Multi-factor auth built for Team Centinels: a TOTP service on Speakeasy, and the same algorithm written from scratch to see how it works."
language: "JavaScript"
forks: 1
tech:
  - "Node.js"
  - "Express"
  - "Speakeasy"
  - "Python"
  - "React"
  - "Firebase"
github: "https://github.com/AbuCTF/MFAuth"
images:
  - "/images/projects/mfauth/firebase.jpg"
draft: false
---

Multi-factor authentication built for [Team Centinels](/work/centinals/), December 2023. Time-based one-time passwords, the thing behind Google Authenticator.

## Written twice, on purpose

The repo has two TOTP implementations. One uses Speakeasy, three endpoints on an Express server:

- `POST /totp-secret`: hand back the shared secret
- `POST /totp-generate`: current token, plus `remaining` seconds in the 30-second step
- `POST /totp-validate`: verify a submitted token

Returning `remaining` alongside the token is a small thing that matters in a UI. Without it the client can't tell you whether your code has 29 seconds left or one.

The other implementation is Python with no TOTP library at all, because using Speakeasy taught me nothing about what TOTP is.

## What TOTP actually is

HMAC of a counter, where the counter is the clock:

```python
counter = int(time.time() / time_step)
counter_bytes = struct.pack('>Q', counter)
mac = hmac.new(key, counter_bytes, digest).digest()
offset = mac[-1] & 0x0f
binary = struct.unpack('>I', mac[offset:offset+4])[0] & 0x7fffffff
otp = str(binary % (10 ** digits)).zfill(digits)
```

That's the whole algorithm. Four interesting lines.

The last byte's low nibble picks where to start reading: dynamic truncation, so the four bytes you keep depend on the hash itself rather than a fixed offset. The `& 0x7fffffff` clears the sign bit, because otherwise the same MAC would produce different digits depending on how a language handles signed integers. Then modulo down to six digits.

Secrets come from `secrets.token_bytes(20)`, base32-encoded, and the provisioning URI gets written out as a QR code:

```
otpauth://totp/{app_name}?secret={secret}&algorithm={ALGORITHM}
```

Scan it with any authenticator app and it works. That's the point of the standard.

## A bug worth keeping

The two implementations don't interoperate. `pyTOTP.py` defaults to SHA-256; Speakeasy defaults to SHA-1. Same secret, same clock, different codes.

RFC 6238 permits both, but almost every authenticator app assumes SHA-1 and most ignore the `algorithm` parameter in the provisioning URI entirely. Which is the actual lesson: the spec allows more than the ecosystem does.

## Front end

React with Firebase email/password auth: signup, login, logout, password reset, email and password updates, and a `PrivateRoute` that bounces unauthenticated users to `/login`.

The console page above is the SDK setup step. Config comes from `REACT_APP_FIREBASE_*` environment variables at build time rather than the inline object Firebase hands you, so nothing from that screen is committed.

## Caveats

`ImplementTOTP/app.js` uses one hardcoded secret shared by all three routes, and `/totp-validate` runs with `window: 0`, no clock-drift tolerance at all. A phone thirty seconds fast fails. Real deployments allow ±1 step.

`secret.txt` and `node_modules/` are both committed. Demo code, not a library to import.

Reference implementation that this borrows from: [susam/mintotp](https://github.com/susam/mintotp/blob/main/mintotp.py). Spec: [RFC 6238](https://datatracker.ietf.org/doc/html/rfc6238). Writeup on [Notion](https://deadgawk.notion.site/Multi-Factor-Authentication-MFA-f8b7b4db6fc0451da2f8170b30277636).
