---
title: "CryptoShield"
tagline: "File encryption over HTTPS"
category: "Security"
status: "archived"
type: "tool"
description: "AES-256 file encryption in Java, served over HTTPS from a free-tier host. Built to understand the cipher, not to wrap a library."
language: "Java"
forks: 1
tech:
  - "Java"
  - "AES-256"
  - "Bouncy Castle"
  - "HTTPS"
  - "Maven"
github: "https://github.com/AbuCTF/CryptoShield"
images:
  - "/images/projects/cryptoshield/aes-flow.png"
  - "/images/projects/cryptoshield/aes-rounds.png"
  - "/images/projects/cryptoshield/project-tree.png"
  - "/images/projects/cryptoshield/encrypt-files.png"
  - "/images/projects/cryptoshield/build-success.png"
  - "/images/projects/cryptoshield/decrypt-files.png"
  - "/images/projects/cryptoshield/hosting-panel.png"
  - "/images/projects/cryptoshield/repo.jpg"
draft: false
---

Encrypt a file with AES-256, get it back over HTTPS. October to November 2023.

The point wasn't to build a product. It was to stop treating `Cipher.getInstance("AES")` as a black box.

## AES, briefly

A block cipher standardised by NIST in 2001. 128-bit blocks, and for AES-256 a 256-bit key run over 14 rounds.

Each round is four steps: **SubBytes** substitutes every byte through an S-box, **ShiftRows** rotates each row left by its index, **MixColumns** multiplies each column in GF(2⁸), and **AddRoundKey** XORs in that round's key.

The round keys come from a key schedule that expands the one 256-bit key into fourteen of them. The diagrams above are that expansion and the round structure, the part that's easy to nod along to and hard to actually follow.

## The implementation

```java
private static void encryptFile(String inputFile, String outputFile, SecretKey key) throws Exception {
    Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
    cipher.init(Cipher.ENCRYPT_MODE, key);

    FileInputStream in = new FileInputStream(inputFile);
    FileOutputStream out = new FileOutputStream(outputFile);

    byte[] input = new byte[64];
    int bytesRead;
    while ((bytesRead = in.read(input)) != -1) {
        byte[] output = cipher.update(input, 0, bytesRead);
        if (output != null) out.write(output);
    }

    byte[] output = cipher.doFinal();
    if (output != null) out.write(output);

    in.close();
    out.close();
}
```

Streaming in 64-byte chunks through `cipher.update()`, with `doFinal()` flushing the padded tail. It never holds the whole file in memory, which matters less for a demo than it does for the habit.

## CBC, not ECB

ECB encrypts each block independently, so identical plaintext blocks produce identical ciphertext blocks. Structure survives encryption. The classic demonstration is an ECB-encrypted bitmap where you can still make out the picture.

CBC XORs each block with the previous ciphertext block first, so identical input blocks land differently depending on what came before them.

That chain needs a starting value, which is the IV: 16 random bytes for AES-CBC, XORed into the first block. It isn't secret and it's stored next to the ciphertext, because decryption needs it. What it can't be is reused, because two messages encrypted under the same key and IV leak their common prefix.

Which is why the output is two files. `enc2.enc` and `enc2.enc.iv`.

## Keys

Symmetric ciphers have the key distribution problem: both sides need the same secret, and getting it there is the hard part. That's what the asymmetric algorithms solve.

RSA rests on integer factorisation. It's simple to explain and increasingly awkward to keep secure, because resisting attack means longer and longer keys.

ECC rests on the elliptic curve discrete logarithm problem and gets equivalent strength from far shorter keys, which makes it cheaper everywhere it matters: smaller certificates, less handshake work, faster on constrained hardware. Measured against RSA on the same operation here, roughly 75ms against 150ms.

## Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.bouncycastle</groupId>
        <artifactId>bcprov-jdk15on</artifactId>
        <version>1.70</version>
    </dependency>
    <dependency>
        <groupId>javax</groupId>
        <artifactId>javaee-api</artifactId>
        <version>8.0.1</version>
    </dependency>
</dependencies>
```

Bouncy Castle as the JCE provider, Java EE for the servlet side.

## Hosting

InfinityFree, free tier, a `.rf.gd` domain with SSL enabled, `datavault.rf.gd`. Deploys over FTP into `/htdocs`.

Free hosting is a genuinely instructive constraint. No shell, no daemons, hard limits, and a control panel that tells you a new domain may take 72 hours to propagate everywhere. You end up understanding what your application actually needs from a host.

## Then

The file-sharing half became [NodeCrypt](/projects/nodecrypt/): same domain, same host, Node instead of Java.
