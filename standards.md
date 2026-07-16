# Coding Standards

The bar every line is written to and reviewed against. The target repo's own conventions
(AGENTS.md and the surrounding code) outrank this file — match the codebase first, then
hold everything else to this bar. Every rule applies in the target language's own idiom.

## Write for the auditor

Write boring, explicit, agent-readable code that a non-owner of the language can
understand and audit without mentally interpreting clever language features.

## Before editing

Read the target code, its callers, relevant tests, and nearby conventions. Verify
unfamiliar APIs instead of guessing, and make only the smallest complete change.

## Shape

Use guard clauses, avoid if/else branches and ternaries, allow no more than one nesting
level, and keep functions within 20 logical lines. A flat coordinator may use up to 40.

## One responsibility

Every function and class has one observable responsibility that can be described without
"and." Every statement directly supports that responsibility.

## One abstraction level

Keep each function at one abstraction level, use descriptive intermediate values, and
organize code from high-level intent to low-level implementation.

## Workers and coordinators

Workers modify only one state owner or external system. Coordinators sequence named
workers for one use case, stay flat, and contain no low-level implementation details.

## Errors are explicit values

Errors are values, never control flow. Do not discard failures, force-extract values,
or use generic errors when a named outcome type can describe the possible results.

No exception handling in business code. When a platform or third-party API can only
throw, isolate it inside a boundary adapter, catch the error once, and immediately
convert it into a specific named outcome. Never catch merely to log, ignore, or rethrow.

## Design by Contract

Every type, function, method, and initializer carries documentation describing its exact
purpose, every parameter, every outcome, state changes, I/O, side effects, and
concurrency restrictions.

## Documentation and comments state facts

Documentation and comments describe the code as it is — its behavior, contracts, and the
technical reason it is that way. They never reference the process that produced it. A
comment that cannot stand alone as a fact about the code is deleted or rewritten as one.

## Names

Use precise names; avoid vague verbs such as process, handle, prepare, normalize,
execute, or manage. Function names and documentation reveal exactly what changes or is
returned.

## Code as prose

Keep signatures, calls, and expressions on one line when they fit within 120 columns.
Parameter count alone must not cause wrapping.

Separate validation, reads, transformation, decisions, effects, and returns with exactly
one blank line. No blank lines between statements that form one logical block.

Use named intermediate values instead of dense expressions. Break chains longer than two
operations. Avoid hidden behavior, custom operators, reflection, macros, type erasure,
and advanced generics unless explicitly required.

## Deep modules

No shallow wrappers, trivial forwarding functions, needless helpers, speculative
abstractions, or classes that exist only to rename another call.

## Queries, commands, and the imperative shell

Queries never mutate state. Business decisions stay separate from I/O. Mutable state
stays private. Dependencies enter through initializers.

## Secondary reactions

Model metrics, notifications, auditing, and similar secondary reactions as independent
Domain Event handlers. Use a Transactional Outbox when event delivery must be guaranteed
after state is committed.

## Done means verified

Add focused tests for successful outcomes, failures, boundaries, and state changes.
Before finishing, run the canonical formatter, the linter, compilation, and relevant
tests, then fix every failure introduced by the change.
