# Go Security Review Notes

Threat model matters. Scope comments to the code under review: libraries, CLIs, services, workers, and internal tooling have different attack surfaces. Prefer findings backed by a concrete trust boundary or abuse path.

## Cross-Cutting Questions

- What inputs are untrusted or only partially trusted?
- What files, processes, network destinations, or credentials can this code touch?
- What authn or authz assumptions does the code make, and where are they enforced?
- What sensitive data can flow into logs, metrics, traces, crash dumps, or client-visible errors?
- Does the security posture rely on deployment controls that are not visible in the code? If so, call out the missing evidence.

## Untrusted Input and Parsing

### Query and Command Construction
**Problem**: Untrusted input is concatenated into SQL, shell commands, or ad hoc query languages.

```go
// Bad: shell injection risk
cmd := exec.Command("sh", "-c", "tar -xf "+archivePath)

// Better: validate input and pass arguments directly
cleanPath, err := sanitizeArchivePath(archivePath)
if err != nil {
    return err
}

cmd := exec.Command("tar", "-xf", cleanPath)
```

Prefer parameterized queries for databases and argument lists for subprocesses.

### Filesystem Paths and Archive Extraction
**Problem**: User-controlled paths, ZIP or TAR entries, or path joins can escape the intended directory.

Review `filepath.Clean`, `filepath.Join`, path-prefix validation, symlink handling, and archive entry names. Extraction code needs the same scrutiny as HTTP file-serving code.

### Templates, Serialization, and Decoding
**Problem**: The code assumes decoded or rendered input is safe without validating size, structure, or encoding context.

For browser-facing HTML, prefer `html/template` over manual string interpolation. For JSON, YAML, gob, protobuf, or custom binary formats, review schema validation, unknown-field handling, recursion limits, and size limits.

## Authentication, Authorization, and Secrets

### Authorization Gaps
**Problem**: Sensitive operations rely on caller intent without verifying permissions at the point of use.

Authorization checks should be close to the state change, not hidden only in routing or UI code.

### Password and Secret Handling
**Problem**: Passwords or long-lived secrets are stored or compared using general-purpose hashes, plaintext, or logs.

Use established password-hashing algorithms such as bcrypt or argon2id based on system requirements. Avoid hardcoded secrets in source, fixtures, examples, or logs.

### Token Validation
**Problem**: Tokens are parsed but not fully validated for signature, issuer, audience, expiry, or intended use.

Modern JWT libraries often reject insecure defaults, but reviewers should still verify the code enforces the expected algorithm and validates the claims the system depends on.

### Randomness for Security Decisions
**Problem**: Security-sensitive identifiers or secrets rely on predictable randomness.

Use `crypto/rand` for tokens, reset links, API secrets, and cryptographic nonces.

## Network, TLS, and Transport

### Insecure TLS Shortcuts
**Problem**: Certificate verification is disabled or transport settings weaken production security.

```go
client := &http.Client{
    Transport: &http.Transport{
        TLSClientConfig: &tls.Config{
            InsecureSkipVerify: true,
        },
    },
}
```

Do not disable certificate verification outside tightly controlled test code. Prefer Go's secure defaults unless compliance requirements force explicit TLS policy.

### Missing Timeouts and Resource Limits
**Problem**: Clients or servers allow unbounded connection, header, body, or idle time.

For network-facing code, review client timeouts, server read and write deadlines, body size limits, and connection shutdown behavior.

## HTTP-Specific Concerns

Only apply these comments when the code is serving browser or HTTP traffic.

### Browser Security Controls
- Review CORS only when browser clients are part of the threat model.
- Review security headers when the service renders browser-consumed responses.
- Review CSRF protections when authentication relies on cookies or other ambient browser credentials.

### Request Handling
- Validate redirects before reflecting user-supplied targets.
- Limit request body size before reading into memory.
- Rate-limit public, abuse-prone endpoints when the service is exposed to hostile traffic.

```go
const maxBodySize = 1 << 20
body, err := io.ReadAll(io.LimitReader(r.Body, maxBodySize))
if err != nil {
    http.Error(w, "request too large", http.StatusRequestEntityTooLarge)
    return
}
```

## Logging, Errors, and Observability

### Sensitive Data in Logs or Errors
**Problem**: Passwords, tokens, raw credentials, session cookies, personal data, or internal system details leak to logs or client-visible errors.

Return safe external errors and preserve detailed context in controlled logs. When logging request metadata, prefer stable identifiers and redacted fields over raw payloads or headers.

### Panic and Crash Surfaces
**Problem**: Panics, stack traces, or debug endpoints can expose secrets or internal state in production.

Review panic recovery, crash reporting, and debug tooling with the same care as normal logging.

## Dependencies and Unsafe Boundaries

### Vulnerable or High-Risk Dependencies
**Problem**: The code introduces dependencies with known vulnerabilities, weak maintenance, or broad transitive trust.

Use tools such as `govulncheck ./...` and review new dependencies for maintenance posture, privilege, and exposure.

### `unsafe`, `cgo`, and Reflection-Heavy Code
**Problem**: Low-level memory access or boundary-crossing code bypasses the usual Go safety guarantees.

These areas deserve extra scrutiny for bounds checks, ownership, lifetime, input validation, and error propagation.
