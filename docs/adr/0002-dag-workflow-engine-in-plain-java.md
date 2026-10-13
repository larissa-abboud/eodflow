# 2. Own workflow engine in plain Java, results kept in memory

**Date:** 2026-10-13
**Status:** Accepted

## Context

The end-of-day process is a set of steps with dependencies. Some steps can run in parallel, and a failure must not produce wrong downstream results. Existing tools (Spring Batch, Airflow, Temporal) solve this, but the goal of this project is to understand and show how such an engine works.

## Decision

- Write a **small, generic workflow engine** in plain Java: jobs, dependency graph, cycle detection, topological ordering, parallel execution on a fixed thread pool, and failure propagation (dependants are SKIPPED).
- Keep the **financial calculations** in a separate pure-Java `domain` package, using **`BigDecimal`** for amounts.
- Use **Spring Boot** only for the REST API and wiring.
- Keep run results **in memory** (`ConcurrentHashMap`) to stay focused on the engine. A database can be added later without changing the engine or the domain.

## Consequences

- Engine and calculations are quick to unit-test, independently of each other.
- I am responsible for concurrency correctness, so the tests run parallel scenarios many times to catch race conditions.
- Results are lost when the application restarts. That's acceptable for this scope.
