# CHORAS Testing Strategy

## 1. Overview

This document defines the testing strategy for the CHORAS simulation platform,
with a focus on architectural scalability, container isolation, distributed
execution, and reproducibility.

Unlike traditional web applications, CHORAS integrates heavy computational
simulation methods that must be containerized and executed in distributed
environments (local Docker, cloud, and HPC).

Testing therefore validates:

- Functional correctness
- Container isolation
- Distributed execution
- Performance and scalability
- Reproducibility

---

# 2. Testing Pillars

## 2.1 Functional Correctness

Ensures all API endpoints and simulation logic behave correctly.

### Includes:
- Backend unit tests
- API integration tests
- Celery task validation
- Input/output schema validation

---

## 2.2 Simulation Contract Testing

Each simulation container must comply with a strict interface contract.

### Requirements:
- Accept standardized JSON input
- Return standardized JSON output
- Respect exit code conventions
- Operate within resource limits
- Avoid filesystem side effects
- Be reproducible

### Test Categories:
- `test_input_schema.py`
- `test_output_schema.py`
- `test_container_exit_codes.py`
- `test_interface_compliance.py`

This ensures new simulation methods can be added without modifying the core backend.

---

## 2.3 Container Isolation Testing

Simulation containers must:

- Not access host filesystem
- Not access other simulation containers
- Respect memory and CPU limits
- Run independently of backend service

### Tests:
- Container filesystem isolation
- Resource limit enforcement
- Dependency independence

---

## 2.4 Distributed Execution Testing

Validates Celery-based distributed processing.

### Tests:
- Task queue latency
- Worker crash recovery
- Retry mechanisms
- Concurrent simulation execution

---

## 2.5 Performance & Scalability Testing

Evaluates system scalability under load.

### Metrics:
- Parallel simulations supported
- Container startup time
- Queue throughput (tasks/min)
- Runtime under concurrent load
- Memory usage per simulation

Performance regressions >10% trigger investigation.

---

## 2.6 Cloud & HPC Execution Testing

Validates deployment on remote infrastructure.

### Tests:
- Remote job submission
- Data persistence in object storage
- Network latency tolerance
- Failure recovery
- Cloud vs local result consistency

---

## 2.7 Reproducibility Testing

For research validity, simulations must produce consistent outputs.

### Tests:
- Identical inputs → identical outputs (within tolerance)
- Deterministic seeds where applicable
- Version-locked dependencies

---

# 3. Release Criteria

A release is approved only if:

- All contract tests pass
- Parallel execution validated
- Celery load tests pass
- Container isolation verified
- Cloud execution tested successfully
- Reproducibility confirmed
- No performance regression >10%

---

# 4. Continuous Integration

CI pipeline includes:

- Unit tests
- Contract tests
- Docker image build validation
- Container smoke tests
- Performance baseline comparison

---

# 5. Architectural Testing Principles

1. Simulation methods must be isolated.
2. Adding a new method must not modify backend core.
3. Containers must be self-contained.
4. Cloud execution must mirror local behavior.
5. Scalability must be measurable.

---

# 6. Testing Philosophy
For our project, we have decided to first explore the current repository and behavior. Once we are confident with the behavior of the code base, we begin by adding the behavior that we want and then also simultaneously write the tests for the code. Once we are fairly confident with the code and believe that most of the base cases have been taken care of, we focus on optimizing the depth and variety of test cases so that we can find more flaws and optimize the code