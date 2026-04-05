# Falcon: A Secure-by-Construction Programming Model

**Constraint-Oriented, Trust-Aware, and Resource-Bounded Systems**

**Version 2.0 — Draft**

---

## Abstract

Modern software systems are increasingly complex, interconnected, and difficult to reason about, leading to persistent classes of vulnerabilities such as injection attacks, unbounded resource consumption, and unintended side effects. These issues are often not the result of inadequate language capabilities, but of implicit assumptions, inconsistent validation, and reliance on developer memory for defensive practices.

Falcon proposes a Secure-by-Construction Programming (SCP) model that shifts safety from an implicit concern to an explicit, compiler-enforced requirement. By introducing constraint scaffolding, trust-aware data flow, capability-based access control, resource-bounded execution, effect declarations, and a unified type system that encodes all five dimensions, Falcon enforces deliberate specification of system behavior at the point of code creation. This approach aims to reduce common classes of vulnerabilities by making unsafe or undefined behavior structurally impossible to express and trivial to detect.

Falcon targets system-level safety properties — input validation, access control, resource management, and side effect tracking — that remain convention-dependent in every mainstream language, including memory-safe languages such as Rust.

---

## 1. Introduction

### 1.1 Problem Statement

Despite advances in programming languages and tooling, software vulnerabilities remain widespread. Common failure modes include:

- Missing or inconsistent input validation
- Implicit trust of external data
- Unbounded memory or computational usage
- Hidden side effects and global state access
- Incomplete or ignored error handling
- Uncontrolled privilege escalation across module boundaries

These issues are rarely caused by a lack of developer capability, but rather by:

- Cognitive overload in complex systems
- Reliance on implicit assumptions
- Inconsistent application of defensive patterns
- Lack of enforceable structure for safety-critical decisions
- Safety properties distributed across independent, non-interacting tools

### 1.2 Motivation

Current programming models prioritize expressiveness, performance, and developer velocity. Safety is typically optional, library-dependent, and inconsistently applied.

Falcon is motivated by the observation that:

> Most vulnerabilities arise not from what developers cannot do, but from what they forget to do.

Existing languages have demonstrated that moving safety properties from convention to compiler enforcement eliminates entire classes of bugs — Rust's ownership model eliminated use-after-free and data race vulnerabilities not by making developers more careful, but by making the errors structurally inexpressible. Falcon extends this approach to the system-level safety concerns that remain convention-dependent in all mainstream languages.

### 1.3 Scope

Falcon does not attempt to replace existing memory-safe languages. Instead, it layers additional safety guarantees — constraint enforcement, trust tracking, capability-based access, resource bounding, and effect management — on top of established memory safety foundations. In its initial implementation, Falcon leverages Rust's memory safety and performance characteristics while extending safety guarantees into domains Rust does not address.

---

## 2. Design Philosophy

Falcon is built on four core principles:

### 2.1 Explicit Intent Over Implicit Assumptions

All meaningful constraints, trust boundaries, and behaviors must be declared or acknowledged. Ambiguity is treated as an error, not a default.

### 2.2 Safe-by-Default, Flexible-by-Exception

Unsafe or undefined behavior is disallowed by default but may be explicitly enabled with mandatory justification. The burden is on the developer to opt out of safety, not to opt in.

### 2.3 First-Class Constraints

Constraints are not auxiliary logic, annotations, or linter rules — they are part of the type system and execution model. An unconstrained external input is an incomplete type and a compilation error.

### 2.4 System-Level Safety, Not Just Language Safety

Security is treated as a property of the entire system — its inputs, outputs, access patterns, resource consumption, and side effects — not solely as a property of memory layout or syntax correctness.

---

## 3. Execution Model

### 3.1 Compilation Pipeline

Falcon implements a multi-phase compilation model:

```
Falcon Source (.fln)
  → Lexing / Parsing (syntax analysis)
  → Semantic Analysis:
      → Constraint resolution
      → Trust-flow analysis
      → Capability verification
      → Effect checking
      → Resource bound validation
  → Code Generation (target backend)
  → Backend Compilation
```

All Falcon-specific safety checks — unresolved constraints, trust violations, missing effect declarations, capability errors, and resource bound conflicts — are enforced during semantic analysis, prior to code generation. If semantic analysis passes, the generated backend code is guaranteed to compile.

### 3.2 Phase 1: Rust Transpilation

In Phase 1, Falcon targets Rust as an intermediate representation:

```
Falcon Source (.fln)
  → Falcon Frontend (semantic analysis + safety checks)
  → Generated Rust Source (.rs)
  → rustc (standard Rust compilation)
  → Binary
```

This approach inherits Rust's memory safety, performance characteristics, and ecosystem while extending safety guarantees to system-level concerns Rust does not address. The Rust backend is a strategic choice: it validates Falcon's semantic model without requiring a full compiler toolchain.

### 3.3 Phase 2: Native Compilation

In Phase 2, Falcon targets LLVM IR directly for native code generation, enabling independent optimization decisions, custom codegen strategies, and removal of Rust-imposed constraints on language expressiveness.

The Falcon semantic model is designed to be backend-agnostic, enabling future migration to direct LLVM code generation without altering the language specification or developer-facing behavior. The frontend is the product; the backend is an implementation detail.

---

## 4. Type System

Falcon's type system encodes five orthogonal dimensions of safety within a unified framework. These dimensions interact and compose, enabling the compiler to enforce cross-cutting safety properties simultaneously.

### 4.1 Constrained Types

In most languages, a type like `String` represents any possible text value. In Falcon, base types are narrowed through constraint blocks that form part of the type's identity:

```
type Username = String {
    max_len = 64
    charset = alphanumeric
    nullable = false
}

type Bio = String {
    max_len = 500
    charset = utf8
    nullable = true
}
```

`Username` and `Bio` are both backed by strings, but the compiler treats them as incompatible nominal types. A `Bio` cannot be passed where a `Username` is expected, even though both are textually representable as strings.

Constrained types encode shape, format, and validity as part of type identity, preventing accidental misuse of structurally similar but semantically distinct data.

### 4.2 Trust Levels

Falcon implements a binary trust model. Data is either `Untrusted` or `Trusted`:

```
let raw: Untrusted<String> = get_input()
let name: Username = validate(raw)
```

**Trust isolation is strictly enforced.** Operations combining values of differing trust levels are rejected at compile time. All operands must be validated to compatible trust levels before combination:

```
// Works — both sides are trusted
let greeting = "Hello, " + name.value

// Compile error — mixed trust levels
let bad = "Hello, " + raw
// Error: Cannot combine Trusted and Untrusted<String>
```

Transition from `Untrusted` to `Trusted` occurs exclusively through typed validator functions, which serve as explicit, auditable trust boundaries:

```
validator fn clean_name(input: Untrusted<String>) -> Username {
    // Constraint checks enforced here
    // Output is Trusted implicitly by validator signature
}
```

Falcon does not prescribe what validation means — it prescribes that validation must occur through a typed boundary before data can be used. The validator is the gate; what happens inside the gate depends on context (format checking, schema validation, cryptographic verification, etc.).

The binary trust model was chosen deliberately over multi-level trust hierarchies to reduce reasoning complexity and eliminate ambiguous intermediate trust states. Data is either verified for its intended use or it is not.

### 4.3 Capabilities

Access to critical systems is controlled through capability tokens — unforgeable values that prove authorization:

```
fn delete_user(user_id: UserId, db: DbWriteCapability) -> Result
```

Capabilities follow three rules:

1. **Cannot be created from nothing** — they are issued by an authorization source
2. **Cannot be duplicated** outside explicit, controlled mechanisms
3. **Must be explicitly passed** — no implicit global access

```
// Capabilities are issued through authorization
let db_write: DbWriteCapability = authorize(current_user, Permission::DbWrite)

// Must be explicitly threaded to functions that need them
delete_user(user_id, db_write)

// Compile error — capabilities cannot be conjured
delete_user(user_id, DbWriteCapability::new())
// Error: DbWriteCapability has no public constructor
```

This eliminates implicit privilege escalation by making every privileged operation's authorization requirements visible in its function signature.

### 4.4 Effect Declarations

Functions explicitly declare their side effects:

```
fn send_email(to: Email, body: MessageBody) -> Result
effects {
    network
    external_io
}
```

Undeclared effects are disallowed. A function that performs network access without declaring the `network` effect produces a compile error. This enables static reasoning about function behavior and prevents hidden operations.

### 4.5 Resource Bounds

Functions define computational resource constraints:

```
fn process_request(input: Data) -> Result
limits {
    max_memory = 32MB
    max_time = 500ms
}
```

Resource bounds are enforced through scoped mechanisms described in Section 6.

### 4.6 Unified Type Composition

These five dimensions are not separate systems — they are orthogonal properties composed within a single type. A fully specified value in Falcon carries its constraints, trust level, and required capabilities as part of its type identity:

```
endpoint fn transfer_funds(
    request: Untrusted<TransferRequest>,   // trust tracking
    auth: BankWriteCapability,              // capability
) -> Result
effects { database_write, audit_log }       // effect declaration
limits { max_memory = 16MB, max_time = 200ms } // resource bounds
{
    let valid: TransferRequest = validate(request)  // trust gate
    // Compiler simultaneously verifies:
    //   - Input is trusted (via validator)
    //   - Caller is authorized (via capability)
    //   - Effects are declared (database_write, audit_log)
    //   - Resources are bounded (16MB, 200ms)
    //   - All errors are handled (Result return type)
}
```

The compiler checks all dimensions simultaneously. Violations in any dimension prevent compilation.

---

## 5. Core Mechanisms

### 5.1 Constraint Scaffolding

When defining external inputs, Falcon generates constraint blocks requiring explicit developer resolution:

```
input name: String {
    max_len = UNSET
    charset = UNSET
    nullable = UNSET
}
```

Compilation fails until all constraints are resolved:

```
Error: External input 'name' has unresolved constraints.

  Suggested fixes:
    1. Use standard type: Username (max_len=64, alphanumeric)
    2. Use standard type: DisplayName (max_len=128, utf8)
    3. Define custom constraints:
       name: String {
           max_len = ???
           charset = ???
           nullable = ???
       }
```

An unconstrained external input has an *incomplete type*. This is not a linter warning — the type does not exist until constraints are resolved. This inverts the conventional model: safety is not opt-in, it is opt-out.

### 5.2 Safe Interaction Primitives

Dangerous operations are replaced with structured interfaces that prevent injection by construction:

```
// Safe — parameterized interface
db.select("users").where(name = validated_name)

// Compile error — raw string operations on queries are disallowed
db.query("SELECT * FROM users WHERE name = '" + input + "'")
```

### 5.3 Mandatory Error Handling

All error-producing operations must be explicitly handled or propagated:

```
let result = parse(input)

match result {
    Success(value) -> process(value)
    Error(ParseError) -> return Error(InvalidInput)
    Error(other) -> return Error(other)
}
```

Silent failures — where an error occurs but execution continues with invalid state — are structurally prevented.

### 5.4 Explicit Constraint Overrides

Developers may bypass constraints with mandatory justification:

```
let debug_data: String {
    constraint_policy = intentionally_unbounded
    justification = "Internal debug buffer — never exposed to external input"
}
```

Similarly, trust isolation can be bypassed when necessary:

```
let debug_msg = "Bad input: " + raw {
    constraint_policy = trust_bypass
    justification = "Logging raw input for diagnostics"
}
```

Overrides maintain flexibility while ensuring accountability and auditability. All overrides are logged and reviewable.

---

## 6. Resource Bounding

### 6.1 Memory Bounds

Memory limits are enforced through scoped arena allocators. Each resource-bounded function receives a fixed-capacity allocation pool. All allocations within the function (and its child threads) draw from this pool:

```
fn process_request(input: Data) -> Result
limits { max_memory = 32MB }
{
    // All allocations in this scope draw from a 32MB arena
    // Allocation beyond the pool boundary produces a recoverable error
}
```

Arena allocation provides predictable, structural enforcement with minimal per-allocation overhead.

### 6.2 Computational Bounds

Computational limits are enforced through a dual mechanism:

**Primary: Fuel-based accounting.** Operations consume abstract computation units. The compiler inserts fuel checks at loop boundaries and function call sites. Fuel exhaustion triggers controlled termination. The fuel model is deterministic and testable.

**Secondary: Wall-clock timeout.** An optional runtime timeout provides additional protection against pathological cases not captured by static fuel analysis. This is a runtime safety net, not the primary enforcement mechanism.

### 6.3 Resource Exhaustion Handling

When a resource limit is hit, the function returns a structured error — consistent with Falcon's mandatory error handling:

```
match process_request(data) {
    Success(response) -> send(response)
    Error(ResourceExhausted(Memory)) -> send_error(503)
    Error(ResourceExhausted(Fuel)) -> send_error(504)
    Error(other) -> log_and_fail(other)
}
```

Resource exhaustion is a recoverable error, not a panic. Callers must handle it explicitly.

---

## 7. Concurrency Model

Falcon extends its safety guarantees across thread boundaries. In Phase 1, Falcon leverages Rust's ownership-based concurrency model for data race prevention, layering capability, effect, and resource enforcement as additional compile-time checks.

### 7.1 Capability Transfer

Capabilities follow move semantics by default. Transferring a capability to a spawned thread revokes it from the parent scope:

```
let handle = spawn(move || {
    store_user(name, db_write)  // db_write moved to this thread
})
// db_write is no longer usable here — compile error if attempted
```

Cloneable capabilities are available as an explicit opt-in with centralized revocation:

```
let db_write: DbWriteCapability<Cloneable> = authorize(...)
let db_write_2 = db_write.clone()
spawn(move || { use(db_write_2) })
// Both copies valid; revocation invalidates all copies
```

### 7.2 Resource Partitioning

When a resource-bounded function spawns child threads, the total resource allocation is partitioned. By default, all threads (including the parent) receive equal shares:

```
fn process(input: Data)
limits { max_memory = 32MB }
{
    // 5 shares: parent + 4 children = ~6.4MB each
    let t1 = spawn(work_1)  // ~6.4MB
    let t2 = spawn(work_2)  // ~6.4MB
    let t3 = spawn(work_3)  // ~6.4MB
    let t4 = spawn(work_4)  // ~6.4MB
    // parent retains ~6.4MB
}
```

Developers may override individual thread allocations:

```
let t1 = spawn(work_1) limits { max_memory = 16MB }  // heavy worker
let t2 = spawn(work_2)  // remaining split evenly among the rest
```

The compiler enforces that the sum of all child allocations plus the parent's retained allocation cannot exceed the parent scope's total limit. This prevents both resource starvation across sibling threads and circumvention of limits through concurrency.

### 7.3 Effect Inheritance

Effect declarations are inherited by spawned threads. Undeclared side effects remain illegal regardless of which thread performs them:

```
fn fetch_data() -> Result
effects { network }
{
    spawn(|| {
        file.write(...)  // Compile error: 'file_io' effect not declared
    })
}
```

### 7.4 Phase 2 Considerations

In Phase 2 (native compiler), Falcon may explore more opinionated concurrency models such as actor systems or structured concurrency. The Phase 1 model — Rust's data race prevention plus Falcon's capability, effect, and resource enforcement — establishes the semantic foundation that any future concurrency model must preserve.

---

## 8. Developer Experience

Falcon requires more upfront specification than conventional languages at system boundaries. This is by design — the specification captures security decisions that developers must make regardless, but that conventional languages allow to remain implicit. However, Falcon employs several mechanisms to minimize unnecessary friction.

### 8.1 Standard Type Library

Falcon ships a standard library of pre-constrained types covering common patterns:

```
// Built-in standard types
type Username = String { max_len=64, charset=alphanumeric, nullable=false }
type Email = String { max_len=254, charset=email_format, nullable=false }
type Port = Int { min=1, max=65535 }
type HttpBody = Bytes { max_len=1MB }
```

Developers use these directly without writing constraint blocks for common fields. Custom constraints are written once per domain-specific type and reused:

```
type MilitaryCallsign = String { max_len=12, charset=alphanumeric_dash, nullable=false }
```

### 8.2 Constraint Inference

Constraints flow forward through operations when the compiler can statically prove the result:

```
fn format_greeting(name: Username) -> String {
    return "Hello, " + name.value
    // Return type String inherits max_len = 70 (6 + 64) automatically
}
```

Developers annotate only when the compiler cannot infer constraints from context.

### 8.3 Project-Level Defaults

A project configuration file sets baseline policies:

```toml
# falcon.toml
[defaults]
string_max_len = 256
nullable = false

[defaults.limits]
max_memory = "64MB"
max_time = "500ms"
```

Individual functions inherit project defaults and override only when necessary:

```
// Inherits 64MB / 500ms from falcon.toml
fn handle_request(req: Request) -> Response { ... }

// Overrides for specific needs
fn heavy_computation(data: Dataset) -> Result
limits { max_memory = 512MB, max_time = 5s }
```

### 8.4 Instructive Diagnostics

Compiler errors are designed to teach, not just reject. Error messages include context, explanations, and suggested fixes (as demonstrated in Section 5.1). IDE integration provides:

- Inline ghost text showing UNSET constraint fields before compilation
- Autocomplete for standard types
- Quick-fix menus offering standard type suggestions
- Visual color-coding of trust boundaries (untrusted vs. trusted variables)
- Capability flow visualization

### 8.5 Progressive Disclosure

Falcon distinguishes between boundary functions and internal logic. Boundary functions (marked with `endpoint` or similar keywords) require full specification — constraints, trust handling, capabilities, effects, and resource limits. Internal functions operating on already-trusted, already-constrained values receive lighter treatment:

```
// Boundary — full annotation required
endpoint fn create_user(input: Untrusted<CreateUserRequest>)
-> Result
effects { database_write, logging }
limits { max_memory = 32MB }
{ ... }

// Internal — minimal annotation, constraints inherited
fn calculate_age(birthdate: Date, today: Date) -> Int {
    return today.year - birthdate.year
}
```

This concentrates developer effort at system boundaries — where security decisions actually matter — while minimizing ceremony for internal logic.

### 8.6 Net Effort Comparison

Falcon requires more specification than conventional languages at trust boundaries, but less total specification than manually implementing equivalent validation, capability checks, resource management, and effect tracking. The overhead is not additional work — it is work that was always necessary but previously left implicit and inconsistently applied.

---

## 9. Security Model

Falcon reduces common vulnerability classes through overlapping, compiler-enforced mechanisms:

| Vulnerability Type | Mitigation Mechanism |
|---|---|
| Injection attacks | Trust-aware data flow + safe interaction primitives |
| Unbounded input | Constraint scaffolding + constrained types |
| Resource exhaustion | Resource bounds with arena allocation + fuel model |
| Privilege escalation | Capability-based access control |
| Hidden side effects | Effect declarations |
| Silent failures | Mandatory error handling |
| Data races | Rust ownership model (Phase 1) / Falcon concurrency model (Phase 2) |
| Cross-thread resource abuse | Resource partitioning with compile-time sum validation |
| Uncontrolled privilege sharing | Capability move semantics with opt-in cloning |

These mechanisms are not independent — they interact and reinforce each other. Resource exhaustion produces errors that mandatory error handling forces callers to address. Trust boundaries determine which capabilities are required. Effect declarations constrain spawned thread behavior. The compositional nature of the safety model means coverage gaps between mechanisms are minimized.

---

## 10. Comparison to Existing Approaches and Novelty

### 10.1 Prior Art

Falcon's individual concepts each exist in prior work:

| Concept | Prior Art |
|---|---|
| Trust / taint tracking | Perl taint mode, Flow, PHP taint extensions |
| Capability-based access | Pony, E, Cap'n Proto, WASI |
| Effect systems | Haskell, Koka, Unison |
| Constraint / refinement types | Ada/SPARK, Liquid Haskell |
| Resource bounding | Erlang (per-process), WASI, Linux cgroups |
| Mandatory error handling | Rust, Haskell |
| Memory safety | Rust, Cyclone |

### 10.2 Falcon's Contribution

Falcon's contribution is threefold.

**Unified enforcement at a single compilation boundary.** In practice, achieving Falcon-equivalent safety today requires stitching together multiple independent tools: a memory-safe language, a taint analysis tool, a capability framework, an effect system library, OS-level resource limits, and custom linter rules. These tools do not interact — a taint tracker does not understand the capability model, and a resource limiter does not know about effect declarations. Violations fall through the gaps between tools. Falcon unifies these mechanisms within a single type system enforced by a single compiler, enabling cross-cutting interactions: capabilities carry constraints, resource bounds interact with thread partitioning, and trust levels determine capability requirements. The safety model is compositional.

**Constraint scaffolding as a compile-time forcing function.** Other systems with refinement types (Liquid Haskell, Ada/SPARK) allow developers to optionally add constraints. Falcon inverts this model — constraints on external inputs are required, and code does not compile until they are resolved. The difference is philosophical but profound: opt-in constraints rely on developer memory; opt-out constraints require developers to explicitly choose to remove safety. No mainstream language or tool implements opt-out constraints on external inputs. This directly addresses the core thesis that vulnerabilities arise from omission, not inability.

**System-level safety as compiler-enforced invariants.** Rust guarantees memory safety. Haskell guarantees referential transparency. These are language-level properties that protect against bugs within a single program. Falcon targets system-level properties — access control, data provenance, resource consumption, and side effect management — that describe how programs interact with users, networks, databases, and external services. Existing languages treat these as application logic under developer responsibility. Falcon treats them as compiler-enforced invariants, extending the "convention to enforcement" paradigm into domains no mainstream language currently addresses.

> Rust moved memory safety from convention to compiler enforcement, eliminating an entire class of bugs. Falcon aims to do the same for input validation, access control, resource management, and side effect tracking — system-level concerns that remain convention-dependent in every mainstream language, including Rust.

---

## 11. Limitations

Falcon does not eliminate all classes of bugs. Remaining challenges include:

- **Business logic errors** — Falcon can enforce that inputs are validated and operations are authorized, but cannot verify that the business rules themselves are correct.
- **Flawed authorization design** — Falcon enforces that capabilities are required and passed explicitly, but cannot determine whether the authorization model itself grants appropriate permissions.
- **Architectural deficiencies** — System-level design flaws (incorrect service boundaries, improper data flow architecture) are outside the scope of a programming language.
- **Protocol-level vulnerabilities** — Flaws in communication protocols, cryptographic implementations, or external dependencies are not addressed by compile-time checks.
- **Constraint correctness** — Falcon ensures constraints are specified, but cannot verify that the chosen constraints are appropriate for the domain (e.g., a developer may set `max_len = 999999` and satisfy the compiler while providing no real protection).

Falcon's goal is to reduce systematic and preventable failures — the vulnerabilities that arise from omission, inconsistency, and implicit assumptions — not to eliminate all possible defects.

---

## 12. Conclusion

Falcon introduces a Secure-by-Construction Programming model that prioritizes explicit intent, enforceable constraints, and system-level safety. By requiring developers to define data constraints, trust boundaries, access capabilities, side effects, and resource limits at the moment code is written, Falcon reduces reliance on memory and convention, replacing them with structure and accountability.

The model is built on a unified type system that encodes five orthogonal safety dimensions — constraints, trust, capabilities, effects, and resources — within a single compilation framework. These dimensions interact and reinforce each other, providing compositional safety guarantees that are impossible to achieve with independent, non-interacting tools.

Falcon does not attempt to eliminate all bugs. It aims to significantly reduce the most common and preventable classes of failures by making unsafe behavior structurally impossible to express and trivial to detect. As software systems continue to grow in complexity, approaches that emphasize explicitness, constraint, and auditability may provide a more sustainable path toward reliable and secure systems.

---

## Appendix A: Implementation Roadmap

### Phase 1 — Proof of Concept (Rust Transpilation)

- Falcon parser and semantic analyzer
- Constraint scaffolding enforcement
- Trust-aware type checking with binary trust model
- Capability verification
- Effect declaration checking
- Code generation targeting Rust
- Standard type library
- Instructive compiler diagnostics

### Phase 2 — Native Compiler (LLVM)

- LLVM IR code generation backend
- Arena allocator integration for resource bounding
- Fuel-based computational limit enforcement
- Advanced concurrency model exploration (actors, structured concurrency)
- Language server protocol (LSP) implementation for IDE integration
- Expanded standard library

---

## Appendix B: Syntax Summary

```
// Type definition with constraints
type Username = String {
    max_len = 64
    charset = alphanumeric
    nullable = false
}

// Validator function — trust boundary
validator fn clean_name(input: Untrusted<String>) -> Username {
    // validation logic
}

// Endpoint function — full specification
endpoint fn create_user(
    input: Untrusted<CreateUserRequest>,
    db: DbWriteCapability,
) -> Result
effects { database_write, logging }
limits { max_memory = 32MB, max_time = 500ms }
{
    let valid = validate(input)
    db.insert(valid)
}

// Internal function — minimal ceremony
fn calculate_age(birthdate: Date, today: Date) -> Int {
    return today.year - birthdate.year
}

// Constraint override with justification
let debug_data: String {
    constraint_policy = intentionally_unbounded
    justification = "Internal debug buffer"
}

// Thread spawning with resource partitioning
fn parallel_process(data: Dataset)
limits { max_memory = 64MB }
{
    let t1 = spawn(heavy_work) limits { max_memory = 32MB }
    let t2 = spawn(light_work)  // remaining split evenly
}
```
