# Cryptography — Interview Prep
---

## SECTION 1: HASHING vs ENCRYPTION

### Q1. What's the fundamental difference between hashing and encryption, and why does confusing them lead to real security bugs?
**Answer:** **Encryption** is **reversible** — data is transformed into ciphertext using a key, and the *same or a related key* can transform it back into the original plaintext. **Hashing** is **one-way** — data is transformed into a fixed-length digest with **no key**, and there is deliberately **no way to reverse it** back to the original input.

| | Hashing | Encryption |
|---|---|---|
| **Reversible?** | No (one-way, by design) | Yes (with the correct key) |
| **Uses a key?** | No | Yes |
| **Output length** | Fixed, regardless of input size (e.g., SHA-256 always → 256 bits) | Varies with input size (roughly input size + overhead) |
| **Purpose** | Integrity verification, password storage, fingerprinting/dedup | Confidentiality — hiding data from anyone without the key |
| **Example use** | Storing a password hash, checksumming a file download | Encrypting a database column, TLS traffic |

**Senior-level answer:**
> "The bug I actually see in code review is **encrypting passwords instead of hashing them** — usually well-intentioned, because 'encryption' sounds more secure than 'hashing.' But if you encrypt a password, anyone with the decryption key — including an attacker who breaches the app server — can recover every plaintext password. A properly hashed password (with salt, Q7) can't be reversed even by someone with full database access; the best an attacker can do is guess-and-check, which is exactly why hashing, not encryption, is the correct primitive for anything you need to *verify* but never need to *retrieve*."

**Trap:** Don't say hashing is "encryption you can't undo" — that framing misses that hashing isn't a weaker form of encryption, it's a **different primitive solving a different problem** (integrity/verification vs. confidentiality). They're not on the same spectrum.

---

### Q2. Where do you use hashing versus encryption in a typical application, concretely?
**Answer:**
- **Hashing:** password storage (Q7), file integrity checks (verifying a download matches its published checksum), detecting duplicate content, Git commit IDs, HMAC for message authentication (verifying a webhook payload hasn't been tampered with).
- **Encryption:** protecting PII/sensitive columns in a database (encryption at rest, Q11), TLS for data in transit (Q11), encrypting files/backups, end-to-end encrypted messaging.

```java
// Hashing (verification only — no key, one-way)
MessageDigest sha256 = MessageDigest.getInstance("SHA-256");
byte[] digest = sha256.digest(fileBytes);  // compare against published checksum

// Encryption (reversible — needs a key)
Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
cipher.init(Cipher.ENCRYPT_MODE, secretKey, gcmSpec);
byte[] ciphertext = cipher.doFinal(plaintextBytes);  // can be decrypted back with the same key
```

**Senior-level answer:**
> "A good litmus test I use: ask 'will I ever legitimately need the original value back?' If yes — a credit card number you need to display the last 4 digits of, or PII you need to process — that's encryption. If the only thing you'll ever need to do is *compare* a future input against this value — a password, a webhook signature — that's hashing (or HMAC). Getting this wrong in either direction is a real vulnerability, not just a style choice."

---

## SECTION 2: SYMMETRIC vs ASYMMETRIC ENCRYPTION

### Q3. Explain symmetric vs asymmetric encryption and the core trade-off between them.
**Answer:**
| | Symmetric | Asymmetric |
|---|---|---|
| **Keys** | One shared secret key, used for both encrypt and decrypt | A key **pair** — public key encrypts, private key decrypts (or vice versa for signing) |
| **Speed** | Fast — suitable for large volumes of data | Much slower (roughly 100–1000x) — computationally expensive |
| **Key distribution problem** | Both parties need the *same* secret key — how do you get it to them securely in the first place? | No shared secret needed — the public key can be distributed openly; only the private key must stay secret |
| **Examples** | AES, ChaCha20 | RSA, ECC (Elliptic Curve) |

```
Symmetric:   plaintext --[key K]--> ciphertext --[same key K]--> plaintext
Asymmetric:  plaintext --[public key]--> ciphertext --[private key]--> plaintext
             (or, for signing: signed with private key, verified with public key)
```

**Senior-level answer:**
> "The trade-off is speed versus the key-distribution problem, and in practice **you almost never choose one exclusively** — real systems use both together in what's called **hybrid encryption**: asymmetric crypto to securely exchange a one-time symmetric key (solving distribution), then fast symmetric crypto to actually encrypt the bulk data. TLS itself works exactly this way — that's the answer I give whenever this question comes up, because 'why not just use RSA for everything' is the natural follow-up and the honest answer is performance: RSA on megabytes of data would be prohibitively slow."

---

### Q4. Walk through how TLS uses both symmetric and asymmetric encryption together (the hybrid approach) at a high level.
**Answer:**
```
1. Client connects; server presents its certificate (contains server's public key, signed by a CA — Q9)
2. Client verifies the certificate chain up to a trusted root CA
3. Client and server perform a key exchange (e.g., ECDHE) — asymmetric crypto —
   to agree on a shared symmetric "session key," without ever transmitting that key directly
4. All subsequent application data is encrypted with fast SYMMETRIC encryption (AES) using that session key
```
- Asymmetric crypto is used **only** for the initial handshake/key agreement and for authenticating the server's identity (via the certificate/signature).
- The actual bulk data transfer uses symmetric encryption for speed.

**Senior-level answer:**
> "This is the canonical real-world example of hybrid encryption, and being able to walk through it — even at a high level, without needing to derive the Diffie-Hellman math — signals genuine understanding rather than memorized definitions. The key insight to emphasize: asymmetric crypto solves the **bootstrapping problem** of establishing a shared secret over an untrusted network, and once that's solved, everything falls back to symmetric crypto because it's orders of magnitude faster."

---

## SECTION 3: AES vs RSA

### Q5. AES and RSA are the two most commonly named algorithms — what specifically distinguishes them beyond "one is symmetric, one is asymmetric"?
**Answer:**
| | AES | RSA |
|---|---|---|
| **Type** | Symmetric block cipher | Asymmetric, based on the difficulty of factoring large prime products |
| **Key sizes (common)** | 128, 192, 256-bit | 2048, 3072, 4096-bit (much larger keys needed for comparable security, due to the different mathematical basis) |
| **Typical use** | Bulk data encryption — files, database columns, TLS session data | Key exchange, digital signatures, certificate signing — small amounts of data |
| **Recommended mode** | **AES-GCM** (authenticated encryption — provides confidentiality AND integrity in one step) over older AES-CBC (confidentiality only, vulnerable to padding oracle attacks without a separate MAC) | OAEP padding for encryption, PSS padding for signatures — never raw/textbook RSA |

```java
// AES-GCM — authenticated symmetric encryption (preferred mode)
Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
GCMParameterSpec spec = new GCMParameterSpec(128, iv);  // 128-bit auth tag
cipher.init(Cipher.ENCRYPT_MODE, aesKey, spec);
byte[] ciphertext = cipher.doFinal(plaintext);  // tampering is detectable on decrypt — throws if tag mismatch

// RSA — typically used to encrypt a small AES key, not bulk data
Cipher rsaCipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
rsaCipher.init(Cipher.ENCRYPT_MODE, rsaPublicKey);
byte[] encryptedAesKey = rsaCipher.doFinal(aesKeyBytes);
```

**Senior-level answer:**
> "I always specifically call out **AES-GCM over AES-CBC** unprompted, because it's the kind of practical detail that separates 'knows crypto exists' from 'has actually implemented it correctly.' CBC mode alone gives you confidentiality but not integrity — an attacker can flip bits in the ciphertext and you won't necessarily detect it without a separate HMAC (encrypt-then-MAC). GCM gives you both in a single, well-vetted construction, which is why it's the modern default and what I'd flag in a code review if I saw CBC used for anything new."

**Trap:** Never say "RSA is used to encrypt files/large data directly" as a normal pattern — RSA's key size limits how much data it can encrypt in one operation (roughly the key size minus padding overhead), and it's far too slow for bulk data. RSA's real jobs are key exchange and signatures, not general-purpose bulk encryption.

---

### Q6. Why is AES-256 still considered secure while RSA needs 2048+ bit keys for comparable protection — why can't you just compare key sizes directly across algorithms?
**Answer:** Key size **isn't directly comparable across algorithm families** because they derive their security from **different mathematical hard problems**, with different known attack efficiencies:
- **AES** security relies on the difficulty of brute-forcing the key space directly (2^256 possibilities for AES-256) — no known shortcut significantly better than brute force exists.
- **RSA** security relies on the difficulty of **factoring the product of two large primes** — and factoring algorithms (e.g., the General Number Field Sieve) are far more efficient than brute force, so RSA needs a *much larger* key (2048–4096 bits) to reach a security level *roughly comparable* to a much smaller AES key.

**Rough equivalence (NIST guidance):** AES-128 ≈ RSA-3072 in security strength; AES-256 has no truly equivalent RSA key size in common practical use, which is part of why **ECC (elliptic curve)** is increasingly preferred over RSA for asymmetric use — it achieves comparable security to RSA with much smaller keys (e.g., a 256-bit ECC key ≈ RSA-3072), because its underlying hard problem (elliptic curve discrete log) doesn't have the same factoring shortcuts.

**Senior-level answer:**
> "This is a good 'do you actually understand it or just memorized the numbers' question. I explain it as: brute-force resistance scales predictably with key size for AES, but RSA's effective security is bounded by the best known factoring algorithm, which improves over time and doesn't scale linearly with key size the same way — that mismatch is exactly why RSA key size recommendations have crept up over the decades (1024-bit RSA, once standard, is now considered broken) while AES-128 has remained comfortably secure since its introduction."

---

## SECTION 4: SHA-256 AND PASSWORD HASHING

### Q7. Why is SHA-256 the wrong choice for hashing passwords, even though it's a strong, widely-used hash function?
**Answer:** SHA-256 is a **general-purpose cryptographic hash**, designed to be **fast** — which is exactly the wrong property for password hashing. Password hashes need to be **slow and computationally expensive** to compute, specifically to make brute-force/dictionary attacks against stolen hash databases impractical. A GPU can compute billions of SHA-256 hashes per second, making brute-forcing a leaked SHA-256 password hash database entirely feasible.

**The correct tools are purpose-built, deliberately slow password hashing functions:**
| Algorithm | Key property |
|---|---|
| **bcrypt** | Configurable "cost factor" (work factor) — tunably slow, includes salting built in |
| **scrypt** | Also memory-hard — expensive in RAM as well as CPU, resisting GPU/ASIC parallelization |
| **Argon2** | Winner of the Password Hashing Competition (2015); current best-practice recommendation — memory-hard, tunable for time/memory/parallelism |

```java
// bcrypt via Spring Security
PasswordEncoder encoder = new BCryptPasswordEncoder(12); // cost factor 12 — tunable slowness
String hashed = encoder.encode(rawPassword);
boolean matches = encoder.matches(rawPassword, hashed);  // handles salt extraction internally
```

**Senior-level answer:**
> "This is one of the highest-signal questions in the whole module because 'just SHA-256 the password' is a genuinely common mistake from developers who know hashing is 'the right general idea' but haven't internalized *why* general-purpose hash speed is a liability specifically in this one use case. I always follow up unprompted with **salting** (Q8) since the two go together — a fast hash with no salt is the worst case, but even bcrypt/Argon2 without proper configuration can be weakened; the cost factor needs to be tuned to something that's deliberately slow on current hardware, and revisited over time as hardware gets faster."

---

### Q8. What is salting, and what specific attack does it prevent that hashing alone doesn't?
**Answer:** A **salt** is a random, unique value generated **per password** and combined with the password before hashing, then stored alongside the hash (salts don't need to be secret, just unique per record).

**What it prevents:** **Rainbow table attacks** and **cross-account pattern detection**. Without a salt, identical passwords produce **identical hashes** — an attacker with a stolen hash database can (1) precompute a lookup table of hash→plaintext for common passwords once, and reuse it against *every* breached database ever ("rainbow tables"), and (2) instantly spot which users share the same password just by comparing hashes. A unique salt per password means even two users with the identical password `"password123"` get **completely different stored hashes**, defeating precomputed tables entirely — the attacker has to attack each hash individually.

```java
// Illustrative — what bcrypt does internally (never hand-roll this)
String salt = generateRandomSalt();               // random, unique per password
String hash = bcrypt(password + salt, costFactor);
store(hash, salt);                                  // salt stored alongside, not secret

// Verification
String candidateHash = bcrypt(inputPassword + storedSalt, costFactor);
boolean matches = candidateHash.equals(storedHash);
```

**Senior-level answer:**
> "The detail worth emphasizing: salts don't need to be kept secret — their entire value comes from being **unique per password**, not from being hidden. bcrypt/Argon2 handle salt generation and storage (usually embedded directly in the output hash string) automatically, which is exactly why you should always use a vetted library rather than hand-rolling `hash(password + globalSalt)` with one shared salt for every user — a single shared salt gives you almost none of the rainbow-table protection, since one precomputed table against *that* salt still works against every user."

---

## SECTION 5: DIGITAL SIGNATURES

### Q9. What problem does a digital signature solve, and how does it use asymmetric crypto in the *opposite* direction from encryption?
**Answer:** A digital signature proves two things about a message: **authenticity** (it really came from the claimed sender) and **integrity** (it wasn't altered in transit) — without requiring a shared secret.

**The key/direction flip versus encryption:** in encryption, the **public** key encrypts and the **private** key decrypts (anyone can send you a secret, only you can read it). In signing, it's **reversed** — the **private** key signs, and the **public** key verifies (only you can produce a valid signature, but anyone can check it).

```
Sign:    hash(message) --[signer's PRIVATE key]--> signature
Verify:  hash(message) compared against decrypting signature with signer's PUBLIC key --> match = authentic & untampered
```

```java
Signature signer = Signature.getInstance("SHA256withRSA");
signer.initSign(privateKey);
signer.update(messageBytes);
byte[] signature = signer.sign();

Signature verifier = Signature.getInstance("SHA256withRSA");
verifier.initVerify(publicKey);
verifier.update(messageBytes);
boolean valid = verifier.verify(signature);  // true only if unmodified AND signed by the matching private key
```

**Senior-level answer:**
> "The 'opposite direction' framing is the detail I check for, because it's exactly what trips people up who half-remember 'public key encrypts, private key decrypts' as a universal rule — it's only true for confidentiality, not for authenticity. I also always mention that you sign a **hash of the message**, not the raw message itself — partly for efficiency (asymmetric operations are slow, Q3), and partly because it fixes the signature to a specific message length regardless of the original message size."

---

### Q10. Where are digital signatures actually used in systems you've worked with — beyond the textbook definition?
**Answer:**
- **TLS certificates** — a Certificate Authority (CA) signs a server's certificate with the CA's private key; browsers verify it with the CA's well-known public key, establishing a chain of trust.
- **Code signing** — verifying a downloaded binary/package genuinely came from the claimed publisher and wasn't tampered with in transit (npm, app store binaries, OS updates).
- **JWTs (JSON Web Tokens)** — when signed with RS256 (RSA) or ES256 (ECDSA), the signature lets any service holding the public key verify a token's authenticity and integrity without needing the private key that issued it — critical for microservices verifying tokens issued by a separate auth server.
- **Blockchain/cryptocurrency transactions** — every transaction is signed by the sender's private key, proving authorization without a central authority.

```java
// JWT signed with RS256 — verifiable by any service holding only the public key
String token = Jwts.builder()
    .setSubject("user123")
    .signWith(privateKey, SignatureAlgorithm.RS256)   // only auth-server holds this
    .compact();

Jws<Claims> claims = Jwts.parserBuilder()
    .setSigningKey(publicKey)                          // any resource server can hold just this
    .build()
    .parseClaimsJws(token);
```

**Senior-level answer:**
> "The JWT example is the one I reach for most in a microservices context, because it directly demonstrates *why* asymmetric signing beats a shared HMAC secret (HS256) at scale: with RS256, only the auth server needs the private key to *issue* tokens, while every downstream resource server only needs the public key to *verify* them — meaning a compromised, less-trusted downstream service can never forge a valid token, since it never had signing capability in the first place. That's a meaningfully different security posture than HS256, where every verifying service must hold the same shared secret used to sign, and a compromise of any one of them compromises the ability to forge tokens system-wide."

---

## SECTION 6: ENCRYPTION AT REST vs IN TRANSIT

### Q11. Distinguish encryption at rest from encryption in transit — and why do you need both, not just one?
**Answer:**
| | Encryption in Transit | Encryption at Rest |
|---|---|---|
| **Protects data** | While moving across a network | While stored on disk (database, backups, file storage) |
| **Typical mechanism** | TLS/HTTPS | Disk/volume encryption, database column/field encryption (e.g., AWS KMS-backed encryption, Transparent Data Encryption) |
| **Protects against** | Network eavesdropping, man-in-the-middle attacks | A stolen disk/backup, unauthorized direct DB/storage access, a compromised storage-layer credential |
| **Doesn't protect against** | A compromised database or leaked backup file — data is decrypted the moment it lands on disk if at-rest isn't also applied | A compromised network path — data is plaintext on the wire if in-transit isn't also applied |

**Senior-level answer:**
> "These are genuinely independent protections against different threat models, and I've seen real architectures get this wrong by treating 'we use HTTPS' as sufficient security — TLS protects the data *while it's moving*, but the moment it's written to a database or an S3 bucket, TLS has nothing more to say about it. If that storage layer is misconfigured (a public S3 bucket, an unencrypted RDS snapshot, a leaked backup), the data is fully exposed regardless of how well the transit leg was secured. The correct answer to 'do we need at-rest encryption if we already have TLS everywhere' is always yes — they protect completely different attack surfaces."

---

### Q12. How would you actually implement encryption at rest for a specific sensitive field (e.g., a customer's SSN) in a Spring Boot application, at a practical level?
**Answer:** Two common levels, often combined:
- **Storage-layer encryption** (transparent, whole-disk/volume) — e.g., AWS RDS encryption, EBS encryption — protects against physical disk theft or unauthorized storage-layer access, but the *application* and any DB user with query access still sees plaintext. Good baseline, insufficient alone for highly sensitive fields.
- **Application/field-level encryption** — encrypt the specific sensitive field *before* it's written to the database, so even a database administrator or a SQL-injection attacker only ever sees ciphertext.

```java
@Convert(converter = SsnEncryptionConverter.class)
private String ssn;   // stored encrypted in the DB column, decrypted transparently in the entity

public class SsnEncryptionConverter implements AttributeConverter<String, String> {
    public String convertToDatabaseColumn(String plainSsn) {
        return aesEncrypt(plainSsn, getKeyFromKms());   // encrypt before persisting
    }
    public String convertToEntityAttribute(String encryptedSsn) {
        return aesDecrypt(encryptedSsn, getKeyFromKms()); // decrypt on read
    }
}
```

**Senior-level answer:**
> "For genuinely sensitive fields — SSNs, payment data, health records — I default to field-level encryption in addition to storage-layer encryption, specifically because it protects against the threat model storage-layer encryption doesn't: a compromised application credential, an over-privileged DB user, or a SQL injection vulnerability that gives raw query access. The trade-off is you lose the ability to query/filter/index on that field directly in the database, so it has to be a deliberate choice per field based on actual sensitivity — you wouldn't field-level-encrypt every column, just the ones that genuinely need it."

---

## SECTION 7: KEY MANAGEMENT

### Q13. Why is key management often described as "the hardest part of cryptography," and what does good key management actually look like in practice?
**Answer:** The cryptographic algorithms themselves (AES, RSA) are well-vetted and rarely the weak point — **the weak point is almost always how keys are generated, stored, rotated, and revoked.** A perfectly implemented AES-256 encryption scheme is worthless if the key is hardcoded in source code, committed to a public Git repo, or never rotated after an employee with access leaves.

**Good key management, concretely:**
- **Never hardcode keys** in source code or config files checked into version control.
- **Use a dedicated key management service** (AWS KMS, HashiCorp Vault, Azure Key Vault) rather than storing keys as plain environment variables — these provide access-controlled, audited key operations, and the raw key material often never even leaves the KMS.
- **Key rotation** — periodically replace keys, so a compromised key has a limited window of usefulness; design the system so old data encrypted under a previous key can still be decrypted (**envelope encryption** — data is encrypted with a data key, which is itself encrypted by a master key that's easier to rotate).
- **Principle of least privilege** — not every service needs decrypt access to every key; scope IAM/access policies per key, per service.
- **Key revocation plan** — a documented process for what happens if a key is suspected compromised (rotate immediately, re-encrypt affected data, audit access logs).

```java
// Envelope encryption pattern — via KMS
byte[] dataKey = kmsClient.generateDataKey(masterKeyId);      // KMS returns a plaintext + encrypted data key
byte[] ciphertext = aesEncrypt(plaintext, dataKey.plaintext);
storeEncrypted(ciphertext, dataKey.encryptedDataKey);          // store the ENCRYPTED data key alongside the data
// Decryption later: ask KMS to decrypt the stored encrypted data key using the master key, then AES-decrypt
```

**Senior-level answer:**
> "Every real-world crypto incident I've read a postmortem on was a key management failure, not an algorithm break — a key in a public repo, an unrotated key from a departed employee, an overly broad IAM policy letting a compromised service decrypt data it never needed to. I treat 'where does this key live, who/what can access it, and how do we rotate it without downtime' as the actual hard engineering problem; picking AES-256-GCM versus AES-256-CBC is comparatively easy by comparison. Envelope encryption specifically is the pattern I reach for at scale, because it means key rotation is a matter of re-wrapping small data keys under a new master key, not re-encrypting every row of a massive dataset."

**Trap:** Don't answer "how do you manage keys" with "use a strong key and keep it secret" — that's necessary but far too shallow for a senior interview. The real substance is in **rotation strategy, access scoping, envelope encryption, and using a managed KMS instead of ad hoc secret storage** — that's what distinguishes a senior answer here.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "What's HMAC, and how is it different from a plain hash?" → HMAC combines a hash function with a secret key, producing a keyed hash that proves both integrity AND authenticity (only someone with the key could have produced it) — a plain hash alone proves only integrity, and anyone can compute it.
- "Is Base64 encryption?" → No — Base64 is an encoding scheme for representing binary data as text; it provides zero confidentiality and is trivially reversible with no key at all. A very common, very wrong interview answer to watch for.
- "What's a nonce/IV, and why does reusing one with AES-GCM matter?" → An IV (initialization vector)/nonce ensures identical plaintexts don't produce identical ciphertexts under the same key; reusing an IV with AES-GCM specifically is catastrophic — it can fully break the authentication guarantee and leak the XOR of two plaintexts, so IVs must be unique (ideally random) per encryption operation with a given key.
- "Why is ECC increasingly preferred over RSA?" → Comparable security with much smaller key sizes (Q6), meaning faster computation, smaller certificates/signatures, and lower bandwidth/storage overhead — increasingly the default for new TLS deployments and mobile/IoT contexts.
- "What's 'perfect forward secrecy,' and why does it matter for TLS?" → A property where a compromise of a server's long-term private key doesn't allow decryption of *past* recorded traffic, because each session negotiates a unique ephemeral key (e.g., via ECDHE) that's discarded after use — modern TLS configurations require this.
- "Should you ever write your own crypto algorithm?" → No — this should be said confidently and without hedging; use well-vetted, widely-reviewed libraries and standard algorithms (AES, RSA, Argon2) exclusively. Hand-rolled cryptography is one of the most common sources of real-world vulnerabilities, even when written by capable engineers.

---

*Study tip: Correctly identifying that passwords need slow, purpose-built hashing rather than SHA-256 (Q7–Q8), the "opposite direction" of key usage in signing versus encryption (Q9), and being able to name concrete key-management failure modes rather than a vague "keep it secret" (Q13) are the three areas where interviewers most reliably separate candidates who've studied definitions from candidates who've actually implemented or reviewed cryptographic code in production.*
