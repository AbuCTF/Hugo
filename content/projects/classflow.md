---
title: "ClassFlow"
tagline: "Attendance by QR code"
category: "EdTech"
status: "archived"
type: "app"
description: "QR-based attendance tracking in Java. Teacher generates a code, students scan it, PostgreSQL keeps the record."
language: "Java"
stars: 3
forks: 3
tech:
  - "Java"
  - "ZXing"
  - "PostgreSQL"
  - "JavaFX"
  - "Maven"
  - "BCrypt"
github: "https://github.com/AbuCTF/ClassFlow"
images:
  - "/images/projects/classflow/tools.jpg"
  - "/images/projects/classflow/screenshot.jpg"
  - "/images/projects/classflow/hub.jpg"
  - "/images/projects/classflow/diagram.png"
draft: false
---

Taking attendance in a large class is slow, and proxy attendance makes it inaccurate as well as slow. ClassFlow replaces the roll call with a QR code.

Built September to December 2023 at SRMIST.

## How it works

The teacher starts a session and the app generates a QR code: session ID, a UUID, a start timestamp and a duration, encoded into a 300x300 PNG with ZXing's `QRCodeWriter`. Students scan it. Attendance lands in PostgreSQL over JDBC.

Scanning back is ZXing's `MultiFormatReader` over a `HybridBinarizer`, which is the part that makes a photographed screen decode reliably rather than only a clean render.

Sessions are persisted as flat lines:

```
Session1:UUID1:1699123456:3600
Session2:UUID2:1699127056:3600
```

Session ID, UUID, start time, duration in seconds. The duration is what makes a code expire. A screenshot passed to a friend in the next lecture validates against a window that has closed.

## Security

More attention went here than the feature list suggests, because attendance systems get attacked by the people using them.

Passwords are hashed with jBCrypt. Registration validates against [passwdqc](https://www.openwall.com/passwdqc/) rules and rejects anything in a bundled 100,000-entry common-password list. That wordlist is why one commit in the repo reads *5 files changed, 100,118 additions*.

Login issues a JWT, HS256, one-hour expiry, with an in-memory revocation set so a logout actually invalidates rather than just forgetting the token client-side. Deleting a user is gated behind a separate admin password prompt.

Every database write goes through `PreparedStatement`.

## Interfaces

Two. A console menu in `Main.java`, and a JavaFX GUI in `MainGUI.java` covering registration, login and user deletion. The console one came first and stayed, because it's faster to test against.

## Stack

Maven, `com.mycompany:ClassFlow:1.0-SNAPSHOT`, compiled at Java 20. PostgreSQL through the JDBC driver, managed with pgAdmin4. Developed across NetBeans and Android Studio: NetBeans for the desktop build, Android Studio for the scanning side.

## Notes

The handwritten pages above are the actual planning for this, day-numbered, with the session table, the auth package and the JWT decision worked out before any of it was written.

Longer writeup: [ClassFlow docs](/docs/dev/classflow/) and on [Notion](https://deadgawk.notion.site/ClassFlow-QR-Based-Attendance-Tracking-App-7e6b84cc3b3948cb847678906e833c94).
