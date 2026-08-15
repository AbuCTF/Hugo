---
title: "NodeCrypt"
tagline: "The file-sharing half of CryptoShield"
category: "Web Dev"
status: "archived"
type: "backend"
description: "Node backend for a password-gated file sharing site. Continuation of the AES-256-over-HTTPS project."
language: "JavaScript"
forks: 1
tech:
  - "Node.js"
  - "Express"
  - "MongoDB"
  - "EJS"
  - "bcrypt"
github: "https://github.com/AbuCTF/NodeCrypt"
images:
  - "/images/projects/nodecrypt/backend.jpg"
  - "/images/projects/nodecrypt/pm2.jpg"
  - "/images/projects/nodecrypt/mongodb.jpg"
  - "/images/projects/nodecrypt/server.jpg"
draft: false
---

The backend for a file-sharing site, written to continue [CryptoShield](/projects/cryptoshield/), the AES-256-over-HTTPS project, with somewhere to actually put the files.

## What it does

Upload a file, get back a link. Optionally set a password on it, in which case anyone following the link has to enter it before the download starts.

The whole server is about sixty lines. Express, EJS for two rendered pages, multer for the upload, Mongoose for the record.

```javascript
const fileSchema = new mongoose.Schema({
  path: { type: String, required: true },
  originalName: { type: String, required: true },
  password: { type: String },
  downloadCount: { type: Number, required: true, default: 0 },
});
```

Passwords are bcrypt-hashed at cost 10 on upload and compared on download. Files land on disk under multer's hashed names with no extension, and the original filename is only reattached at the end, by `res.download(file.path, file.originalName)`.

## What it isn't

Worth being precise, because the name suggests otherwise: NodeCrypt does not encrypt the files. The password is hashed, the transport was HTTPS, but the bytes on disk are the bytes you uploaded. Encryption was CryptoShield's job; this was the delivery half.

## Hosting

Same setup as CryptoShield: a free TLD with SSL enabled on InfinityFree, at `datavault.rf.gd`. Deploys were FileZilla into `/htdocs`, and the process ran under PM2 so it came back after a restart.

## What I'd change

`multer({ dest: "uploads" })` with no size limit and no MIME filter is an open door. The share link is also built from `req.headers.origin`, which is client-supplied, so you can make the server hand you a link pointing anywhere.

Writeup on [Notion](https://deadgawk.notion.site/NodeCrypt-Backend-for-File-Sharing-Site-a4c7c2f271e54173995d3775b166973a). Inspired by [WebDevSimplified's file-sharing app](https://github.com/WebDevSimplified/file-sharing-node-js).
