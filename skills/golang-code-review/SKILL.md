---
name: golang-code-review
description: "General Go code review skill for pull requests, single files, snippets, security audits, refactors, legacy code, library/package reviews, concurrency reviews, architecture reviews, and test quality analysis. Use when reviewing Go code for correctness, maintainability, API design, concurrency safety, security, performance, and project conventions."
---

# Golang Code Review

Review Go code with context-aware, evidence-based feedback. Use repository conventions when they exist; otherwise fall back to idiomatic Go guidance and the bundled references.

## Review Process

### 1. Establish Review Context

Before reviewing, identify whatever context is available:
- PR, commit, diff, issue, or ticket summary when present
- File-level intent, callers, tests, and public API surface when no change summary is available
- Whether the code is primarily a library, CLI, service, worker, tool, integration, generated adapter, or test helper
- Whether the main risk is correctness, design, concurrency, security, performance, operability, or test quality

If context is incomplete, say so and keep conclusions limited to what the code supports.

### 2. Load Relevant References

Use the bundled references selectively:
- For general code quality and API design: `references/effective-go.md`
- For bug-prone patterns and implementation hazards: `references/common-mistakes.md`
- For trust boundaries, untrusted input, filesystem or process access, networking, auth, or secret handling: `references/security.md`

Treat the references as review heuristics, not rigid rules. Project-specific guidance can override generic advice.

### 3. Choose One or More Review Lenses

Use the lens that fits the request. Combine them when needed.

**PR or Merge Review**: Focus on behavioral regressions, compatibility risks, missing tests, and whether the change is safe to integrate.

**Single File or Snippet Review**: Focus on local correctness, clarity, API shape, and whether the implementation matches likely intent.

**Architecture or Package Review**: Focus on package boundaries, dependency direction, exported API design, interface placement, cohesion, and long-term maintainability.

**Test Review**: Focus on behavior coverage, failure modes, flakiness risk, isolation, observability, and whether tests verify outcomes rather than implementation details.

**Security Review**: Focus on trust boundaries, authorization, secret handling, filesystem and process safety, dependency risk, and whether the code safely handles hostile or malformed input.

**Refactor or Modernization Review**: Focus on preserving behavior, simplifying control flow, removing obsolete patterns, and documenting compatibility or migration risk.

### 4. Review the Code That Controls Behavior

Prioritize the code that directly determines behavior:
- Public APIs, exported types, and compatibility-sensitive code
- State transitions, data flow, resource lifetime, and cancellation boundaries
- Error handling, retries, logging, and observability contracts
- Concurrency control, goroutine lifecycle, channels, mutexes, and shutdown paths
- Parsing, validation, encoding, filesystem access, network boundaries, and process execution

Supporting wiring still matters, but focus first on code that computes, mutates, or authorizes behavior.

## Core Review Areas

### Correctness and Behavior

- Check that the code matches its documented or implied contract
- Look for silent failure paths, partial writes, nil handling, off-by-one behavior, and incompatible defaults
- Flag behavioral changes that need migration notes, versioning, or explicit rollout strategy

### API and Design

- Review exported names, package boundaries, and what becomes part of the public surface
- Treat interface guidance as contextual: small consumer-owned interfaces are often better, but concrete types can be the right choice when behavior or evolution is easier to reason about that way
- Check for unnecessary abstraction, leaky layering, cyclic dependencies, or hidden coupling through globals
- Consider generics, functional options, and configuration APIs based on clarity and compatibility, not fashion

### Errors, Resources, and Lifecycle

- Prefer errors that preserve actionable context without hiding root causes
- Check cleanup paths for files, locks, network connections, goroutines, and spawned processes
- Verify `context.Context` propagation where cancellation, deadlines, or request scoping matter
- Look for panic usage that crosses package boundaries without a clear recovery strategy

### Concurrency and State Safety

- Review goroutine termination, cancellation, channel ownership, and shutdown coordination
- Look for shared mutable state, map access races, lock ordering issues, and blocking sends or receives
- Check whether concurrency is necessary at all; complexity without measurable value is review-worthy

### Security and Trust Boundaries

- Scope security findings to the kind of code under review: libraries, CLIs, services, workers, and internal tooling have different threat models
- Check untrusted input, file paths, process execution, network exposure, serialization, secrets, authn/authz, and logging or redaction
- Treat framework or deployment assumptions as context, not proof; call out missing evidence when the security posture depends on external controls

### Testing and Validation

- Expect tests for both success and failure modes when behavior can realistically fail
- Prefer tests that assert observable outcomes, contracts, and compatibility-sensitive behavior
- Watch for hidden global state, order dependence, race-prone tests, and brittle assertions tied to implementation details
- For performance-sensitive code, look for benchmarks or evidence rather than speculative optimization

## Feedback Style

Organize findings by severity and confidence.

### Critical Issues
Use for security vulnerabilities, data corruption, concurrency hazards, resource leaks, or changes that can break users or production systems.

### Important Issues
Use for correctness bugs, missing validation, missing tests, compatibility problems, or design choices likely to create maintenance pain.

### Suggestions
Use for clarity improvements, simplifications, performance follow-ups, or non-blocking idiomatic improvements.

### Open Questions
Use when the code may be correct but important context is missing.

### Positive Feedback
Include when it adds signal. Skip it if the review is short and there is nothing meaningful to highlight.

For each finding, explain:
- **What** is wrong or risky
- **Why** it matters
- **How** to fix it when a local fix is clear

Provide code examples only when they clarify the recommendation. Higher-level design feedback does not always need replacement code.

## Default Output Format

Use this format when a structured review is helpful. Omit empty sections.

```markdown
## Critical Issues

## Important Issues

## Suggestions

## Open Questions

## Positive Feedback
```

For each finding:

````markdown
### [Area]: [Brief description]

**Problem**: [What's wrong and why it matters]

**Current code**:
```go
[Show only the relevant snippet]
```

**Suggested fix**:
```go
[Show corrected code only when a concrete local fix is appropriate]
```

**Reference**: [Relevant bundled reference or project convention]
````

Inline comments or a shorter summary are also acceptable when the user asks for a lightweight review.

## Example Review Snippets

### Concurrency Review Example

````markdown
### Concurrency: Goroutine can outlive the caller

**Problem**: The worker goroutine blocks on send without observing cancellation. In a library or request-scoped path, this can leak goroutines and hold memory after the caller has gone away.

**Current code**:
```go
func startWorker(ctx context.Context, out chan<- Result) {
    go func() {
        result := compute()
        out <- result
    }()
}
```

**Suggested fix**:
```go
func startWorker(ctx context.Context, out chan<- Result) {
    go func() {
        result := compute()
        select {
        case out <- result:
        case <-ctx.Done():
        }
    }()
}
```

**Reference**: `references/common-mistakes.md` - concurrency and goroutine lifecycle
````

### Security Review Example

````markdown
### Security: Untrusted input reaches process execution

**Problem**: The code builds a shell command from user-controlled input. This is command-injection risk in services and local tooling alike.

**Current code**:
```go
cmd := exec.Command("sh", "-c", "tar -xf "+archivePath)
```

**Suggested fix**:
```go
cleanPath, err := sanitizeArchivePath(archivePath)
if err != nil {
    return err
}

cmd := exec.Command("tar", "-xf", cleanPath)
```

**Reference**: `references/security.md` - process execution and untrusted input
````

## Review Heuristics

Consider skipping or shortening comments when:
- The issue is purely formatting and `gofmt` or the project's formatter already owns it
- The point is personal preference rather than correctness, maintainability, compatibility, or project convention
- The same issue has already been explained elsewhere in the review
- The code is generated and the real fix belongs in the generator or its configuration

## Multi-file Review Strategy

For large changesets:
1. Start with the architectural or compatibility-sensitive parts: public APIs, package boundaries, migrations, and core behavior
2. Review the code that directly controls behavior before adapters and plumbing
3. Review tests for the most important behaviors and failure modes
4. Review supporting files such as configuration, tooling, glue code, or generated artifacts only after the main behavior is understood
5. End with a concise assessment of risk, missing evidence, and any required follow-up

## After Review

If significant issues are found:
- Summarize the most important risks and whether they block integration, release, or adoption
- Call out assumptions and unknowns explicitly
- Offer concrete next steps when the right fix depends on missing context
