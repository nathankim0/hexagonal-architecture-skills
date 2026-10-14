---
name: hexagonal-architecture
description: Review, design, and incrementally refactor code using Hexagonal Architecture (Ports and Adapters). Use for requests about hexagonal/헥사고날/헥사코날 architecture, input/output ports, adapter isolation, infrastructure coupling, or testing application boundaries. Avoid unrelated style-only reviews.
license: MIT
---

# Hexagonal Architecture

Keep application behavior independent of how users invoke it and how external systems fulfill its needs. Apply Ports and Adapters to concrete boundaries that improve testing or isolate change. Respond in the user's language.

For a review, report findings without editing. For design, propose the smallest structure that serves the requested feature. For implementation or refactoring, make the requested changes and verify behavior. Do not turn a focused request into a repository-wide migration.

## Read the actual boundaries

- Read repository instructions and the relevant code, callers, dependency declarations, wiring, and tests before judging the architecture.
- Trace one real operation from its entry point to its external effects. Identify the domain rules, application orchestration, input API, and outbound contracts.
- Inspect imports, function signatures, annotations, inheritance, and runtime construction. Directory names alone neither prove nor disprove separation.
- Distinguish source dependencies from runtime calls: a use case can call an injected adapter through a core-owned port without importing its implementation.

Use this vocabulary consistently:

| Part | Responsibility | Typical examples |
| --- | --- | --- |
| Application core | Business rules and use-case orchestration | Pricing policy, register-member operation |
| Input / driving port | Application API offered to a caller | Use-case function or method; an interface when useful |
| Driving / inbound adapter | Turns an external trigger into a call to that API | HTTP handler, CLI, queue consumer, scheduled job, test harness |
| Output / driven port | Contract the application needs an external collaborator to satisfy | Member lookup, payment authorization, notification delivery |
| Driven / outbound adapter | Implements that contract using a specific technology | SQL repository, payment SDK client, in-memory fake |
| Composition root | Constructs and connects the application and adapters | Startup module, dependency-injection configuration |

The hexagon illustrates an inside/outside boundary; it does not require six ports or a fixed number of layers.

## Review the contracts and dependencies

**Core independence**

- Look for application code importing concrete adapters, framework request/response types, ORM sessions or entities, SDK clients, service locators, or infrastructure globals.
- Move technology-specific behavior behind a contract owned by the application that consumes it. Keep construction and configuration outside the core.
- Pure utility libraries are not automatically violations. Report coupling when it ties application behavior to a delivery or infrastructure mechanism.

**Meaningful ports**

- Express outbound operations in the application's vocabulary. A port should describe the required capability, not expose a vendor SDK or a generic database query engine.
- Use the language's existing idioms: functions, structural types, protocols, traits, or interfaces. An input port can simply be a use-case function; not every class needs an interface.
- A single production implementation can still justify a port when it isolates real I/O and permits a test double. Do not add interfaces to unrelated pure helpers for hypothetical replacements.
- Check return values, absence, errors, async behavior, and side-effect semantics. Adapters must satisfy the same observable contract, not just the same method names.

**Adapter responsibilities**

- Inbound adapters parse and validate external input, establish trusted identity, call the application API, and map outcomes to transport responses. Keep business authorization and invariants in the core.
- Outbound adapters translate application operations into infrastructure calls and map vendor data and errors back into application-owned values and outcomes.
- Prevent lazy ORM objects, HTTP objects, vendor exceptions, and serialization details from leaking into application contracts. Reuse suitable domain values; do not require a new DTO for every internal hop.
- Core code must not reach across to another adapter. Adapters can compose technology-specific helpers without needing a port for every helper.

**Behavior and wiring**

- Check that startup selects and injects the intended implementations, including configuration and lifecycle management.
- When moving a boundary, preserve authentication, validation, transaction scope, timeouts, cancellation, and existing retry/idempotency behavior where applicable.
- Do not claim a database transaction makes external API calls atomic. Flag a demonstrated consistency problem and recommend the smallest appropriate fix; do not introduce an outbox or saga speculatively.

## Design or refactor incrementally

1. State the use case and the external boundary that needs separation. Reuse existing contracts and module conventions.
2. Define only the input API, outbound capability, and application-owned data needed for that operation.
3. Keep business decisions in the core, translation in adapters, and concrete assembly at startup.
4. For existing code, first capture the affected behavior with a focused test. Extract one boundary, update all callers and wiring, and preserve externally visible behavior.
5. Run the affected tests and relevant build/type checks. Name unverified behavior and any remaining coupling.

Follow the repository's structure. Do not mandate `domain/`, `application/`, `ports/`, and `adapters/` folders, split deployments, DDD aggregates, CQRS, an event bus, or a dependency-injection framework unless the task needs them. When a simple module already provides the boundary, keep it.

## Verify substitutability and testability

- Exercise the application through its input API using deterministic fakes for outbound ports, without an HTTP server, database, or live vendor service.
- Cover business outcomes and meaningful failure paths, such as missing records or rejected actions. Avoid assertions that only confirm internal call order.
- Check adapter mappings with focused integration or contract tests where their real behavior matters: absence, uniqueness, transactions, errors, or serialization. An in-memory fake cannot establish database correctness.
- If the repository has dependency checks, update or run them for changed boundaries. Do not add a new checking framework merely for one review.

Read [references/example.md](references/example.md) only when a concrete example would help explain input APIs, outbound ports, adapter substitution, and wiring. Its Python example uses only the standard library; translate the pattern into the project's own language and conventions.

## Report useful findings

Lead with the most consequential supported finding. For each issue include:

- File and line, plus the actual import or call path crossing the boundary.
- The concrete consequence: harder isolated testing, transport-dependent rules, leaked infrastructure errors, or a demonstrated behavior risk.
- The smallest fix and the check that would verify it.

Separate observed violations from optional improvements. Do not invent defects to populate a checklist; say when no material issue was found. For a design, show the call flow and ownership of contracts. After implementation, summarize the change, verification, and limitations.

## References and attribution

- [Hexagonal Architecture — Alistair Cockburn](https://alistair.cockburn.us/hexagonal-architecture): original Ports and Adapters pattern.
- Inspired by [nathankim0/clean-architecture-skills](https://github.com/nathankim0/clean-architecture-skills), particularly its architecture-review workflow and shared Claude Code / Codex packaging. This skill applies Hexagonal Architecture rather than requiring Clean Architecture's layer model.
