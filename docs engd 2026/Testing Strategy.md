# CHORAS Testing Strategy
## CHORAS Scalability Project - EngD 2026

---

## 1. Overview

This document defines the testing strategy for the CHORAS simulation platform.
Unlike traditional web applications, CHORAS integrates heavy computational
simulation methods that must be containerized and executed across distributed
environments (local Docker, cloud, and HPC). Testing must therefore go beyond
functional correctness and address container isolation, distributed execution,
scalability, and reproducibility.

For guidance on how to set up, run, and write tests in practice, refer to
`testing_guide.md`.

---

## 2. Testing Philosophy

Testing in CHORAS follows an incremental, understanding-first approach across
three stages:

1. **Explore** — the existing codebase and runtime behavior are studied before
   writing any tests, to ensure tests reflect actual rather than assumed behavior.
2. **Implement and test in parallel** — as new behavior is introduced during
   re-architecture, corresponding tests are written alongside it rather than
   after the fact.
3. **Deepen and diversify** — once baseline coverage is established, test cases
   are systematically expanded using equivalence partitioning and boundary value
   analysis to uncover edge cases and drive further optimization.

---

## 3. Test Design Approach

The primary test design technique used in CHORAS is **equivalence partitioning**,
supplemented by **boundary value analysis** at partition edges. Equivalence
partitioning ensures that the test suite is comprehensive without being redundant,
by identifying groups of inputs or system states that the platform should handle
identically — meaning one representative case per partition is sufficient to
expose defects in that class. Boundary value analysis then targets the edges
between partitions, where defects are statistically most likely to occur.

Test paths through the system — for example, the sequence of steps from
submitting a simulation method through queuing, execution, and result delivery —
are modeled and managed in **TestCompass**, which generates test scenarios at
different coverage levels (node, edge, full path) and identifies which test cases
are affected when requirements change. EP and BVA determine *what data* to use;
TestCompass determines *which behavioral paths* to exercise with that data.

---

## 4. Equivalence Partitions

### 4.1 Computational Cost of Methods
- Computationally cheap methods
- Computationally expensive methods

### 4.2 Execution Environment
- Methods executed on localhost
- Methods executed on the cloud

### 4.3 Backend Method Count
- 0 methods in the backend
- Some methods in the backend
- Maximum number of methods in the backend
- Maximum minus one methods (boundary)
- Maximum plus one methods (boundary — expect rejection or graceful handling)

### 4.4 Cloud User Load
- One user running one container on the cloud
- One user running the maximum number of containers on the cloud
- Maximum number of users each running one container on the cloud
- Maximum number of users each running the maximum number of containers
  on the cloud (stress boundary)
- Maximum plus one users (boundary — expect rejection or graceful handling)

### 4.5 Method Dependencies
- Methods with no dependencies on other methods
- Methods that depend on the output of another method (chained execution)

### 4.6 Output Determinism
- Methods that produce deterministic outputs
- Methods that produce stochastic outputs

### 4.7 Input Data Validity
- Valid acoustic input parameters
- Invalid acoustic input parameters (e.g., negative frequency, out-of-range values)
- Boundary values for key parameters (minimum and maximum accepted values)
- Empty input
- Minimal input
- Large or complex input datasets
- Malformed input (e.g., wrong file format, missing fields)

### 4.8 Configuration State
- Default configuration
- Fully custom configuration
- Missing configuration file
- Malformed or corrupted configuration file

### 4.9 Method Execution Outcome
- Method completes successfully
- Method fails mid-execution
- Method times out

### 4.10 Cloud Resource Availability
- All cloud resources available and healthy
- A container crashes during execution
- A cloud resource becomes unavailable mid-execution (e.g., network timeout)

### 4.11 Concurrency
- Single user triggering a method
- Multiple users triggering the same method simultaneously
- Multiple users triggering different methods simultaneously

### 4.12 Authentication and Authorization
- Authenticated user with correct permissions
- Unauthenticated request
- Authenticated user attempting to access resources beyond their permission level

---

## 5. Testing Pillars

### 5.1 Functional Correctness
Ensures all API endpoints and simulation logic behave correctly, covering backend
unit tests, API integration tests, Celery task validation, and input/output
schema validation.

### 5.2 Simulation Contract Testing
Each simulation container must comply with a strict interface contract: accepting
standardized input, returning standardized output, respecting exit code
conventions, operating within resource limits, avoiding filesystem side effects,
and producing reproducible results. This ensures new simulation methods can be
added without modifying the core backend.

### 5.3 Container Isolation
Simulation containers must not access the host filesystem, must not access other
simulation containers, must respect memory and CPU limits, and must run
independently of the backend service.

### 5.4 Distributed Execution
Validates the Celery-based distributed processing layer, including task queue
latency, worker crash recovery, retry mechanisms, and concurrent simulation
execution.

### 5.5 Performance and Scalability
Evaluates system behavior under load, measuring parallel simulation capacity,
container startup time, queue throughput, runtime under concurrent load, and
memory usage per simulation. Performance regressions exceeding 10% trigger
investigation.

### 5.6 Cloud and HPC Execution
Validates deployment on remote infrastructure, including remote job submission,
data persistence in object storage, network latency tolerance, failure recovery,
and consistency of results between cloud and local execution.

### 5.7 Reproducibility
For research validity, simulations must produce consistent outputs: identical
inputs must yield identical outputs within a defined numerical tolerance,
deterministic seeds must be applied where applicable, and dependencies must
be version-locked.

---

## 6. Architectural Testing Principles

1. Simulation methods must be isolated — a fault in one method must not affect
   others.
2. Adding a new simulation method must not require modification of the backend
   core.
3. Containers must be self-contained and free of external side effects.
4. Cloud execution must mirror local execution in terms of correctness and output.
5. Scalability must be measurable and tracked across releases.

---

## 7. Continuous Integration

The CI pipeline runs automatically on every push and pull request to protected
branches. It verifies that:

- All unit tests pass
- All contract and interface compliance tests pass
- Docker images build successfully
- Container smoke tests pass
- Performance results are within the accepted baseline threshold

---

## 8. Release Criteria

A release is approved only if all of the following conditions are met:

- All contract tests pass
- Parallel execution has been validated
- Celery load tests pass
- Container isolation is verified
- Cloud execution has been tested successfully
- Reproducibility is confirmed across all deterministic methods
- No performance regression exceeds 10%
- All critical and high-severity defects are resolved or formally accepted
- The requirements traceability matrix confirms full verification coverage

---

**Last Updated**: February 2026  
**Maintained by**: Test Manager  
**Related documents**: `testing_guide.md`, `verification_process.md`