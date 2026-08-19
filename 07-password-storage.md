# 7. Password & Credential Storage

## 7.1 Why you never store a plaintext password, and why "encrypt it" is also wrong

If you store `password = "hunter2"` and your database ever leaks (and databases leak — breaches
happen to well-run companies too), every user's real password is exposed immediately, and because
of password reuse, so are their accounts on every other site they used the same password on.

**Encryption is also the wrong tool here**, even though it sounds more secure than "plaintext."
Encryption is reversible by design — whoever holds the key can decrypt every password back to
plaintext. If your database leaks alongside the application's encryption key (a very common
combined leak, since the key often lives in the same config/environment), you're back to square
one. What you actually want is a function that is **deliberately irreversible**: a
**cryptographic hash**.

## 7.2 Why a plain hash (MD5, SHA-256) is still wrong

`SHA-256("hunter2")` is irreversible in the sense that you can't mathematically invert it back to
"hunter2" — but that's not the threat that matters. The real threat is **an attacker who has stolen
your hash database going ahead and computing the hash of every password in a giant precomputed
list (a rainbow table) or just brute-force-guessing millions of candidate passwords per second**,
because general-purpose hash functions like MD5 and SHA-256 are *designed to be fast* — that's a
feature for checksums and a fatal flaw for password storage, since fast means an attacker's
offline guessing loop is also fast. Modern GPUs can compute billions of SHA-256 hashes per second;
a weak or common password is recovered in well under a second regardless of how "irreversible" the
hash function is.

## 7.3 What you actually need: a slow, memory-hard, salted hash

**Salting:** a unique, random value stored alongside each password's hash, mixed in before
hashing (`hash(password + salt)`). Without a salt, two users with the same password produce the
same hash, which (a) leaks that fact directly, and (b) lets an attacker precompute one rainbow
table that works against every account in every leaked database that used the same weak hash. A
unique salt per user forces the attacker to attack every hash individually — no precomputation
reuse across users or across breaches.

**Deliberately slow, tunable algorithms — this is the actual point:**
- **bcrypt** — includes a "cost factor" you can increase over time as hardware gets faster,
  keeping the per-guess cost expensive indefinitely without changing your code, only a config
  number.
- **scrypt** and **Argon2** (the current recommended default, winner of the Password Hashing
  Competition) — additionally **memory-hard**: they deliberately require a large amount of RAM per
  hash computation, which specifically defeats the massively-parallel GPU/ASIC cracking rigs that
  make short work of memory-light algorithms like plain SHA-256, because GPUs have many cheap
  cores but comparatively little memory per core.

```
# Argon2 conceptually — the salt and cost parameters travel WITH the hash, so verification
# doesn't require a separate lookup:
$argon2id$v=19$m=65536,t=3,p=4$c29tZXNhbHQ$hashoutputhere...
             ^^^^^^^^^^^^^^^^^ memory (m), iterations (t), parallelism (p) — all tunable
```

**Verification is symmetric and simple:** re-hash the submitted password using the same salt and
parameters stored with the record, compare the result to the stored hash (with a constant-time
comparison — see [06 §6.6](06-session-and-token-security.md)). You never "decrypt" a password hash
— you can only ever check "does this candidate produce the same hash."

## 7.4 Peppering — an extra layer, not a replacement for salting

A **pepper** is a secret value, the same for every user, stored **separately from the database**
(in application config or a secrets manager, never alongside the hashes it protects) and mixed
into the hash input alongside the per-user salt. The point: if the database alone leaks (a SQL
injection, a misconfigured backup) but the application's secret config does not, the pepper is
still missing, and the attacker's offline cracking attempt is still blocked even with the salts and
hashes in hand. This is defense-in-depth on top of salting, not instead of it.

## 7.5 Practical password policy — what actually helps, and what's theater

Modern guidance (NIST SP 800-63B) is a real shift from older intuition:
- **Length matters far more than complexity rules.** A long passphrase (`correct horse battery
  staple`-style) resists brute force better than a short, complex-looking password
  (`P@ssw0rd1!`), which attackers' cracking dictionaries already account for (common
  substitutions like `@`→`a`, `0`→`o` are the first thing a cracking wordlist tries).
- **Forced periodic rotation ("change your password every 90 days") is now actively discouraged**
  — in practice it drives users toward predictable, incrementing passwords (`Summer2024!` →
  `Summer2025!`) and adds no real security benefit against the actual threat (credential theft,
  which is time-independent — an attacker uses a stolen password immediately, not 90 days later).
  Rotate on evidence of compromise, not on a calendar.
- **Check against known-breached password lists** at signup/change time (e.g. the Have I Been
  Pwned Pwned Passwords API, queryable via k-anonymity so you never send the actual password) —
  this stops the single most effective real-world attack, credential stuffing with reused
  passwords, far more effectively than a complexity regex does.
- **Rate limit and lock out** login attempts per account and per source IP, with exponential
  backoff, to make online brute force impractical regardless of password strength.
