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


---
#### SYSTEM INTERFACE DIAGRAM /ARCHITECTURE ADD IT 

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
#### TEST PYRAMID (UI LEVEL, CODE LEVEL, INTEGRATION LEVEL), 


## Test Risk Justification

For our tests we have decided to focu primarily on integration tests, with some unit tests being done as well. As we are constraint in terms of time and inputs to test against we felt that doing integration tests that cover the majority of the behavior of the project will be far more suitable for our timeline. 


### Why Integration Tests Are Prioritised
The core risk in this system lies at the boundaries between components, 
specifically how simulation_service routes to the correct executor, how 
LocalExecutor and CloudExecutor handle real Docker and SSH interactions, 
and how the post-processing pipeline behaves after a job completes. These 
risks are not detectable by unit tests alone, making integration tests the 
highest-value investment for our scope.

Unit tests are included where the logic is self-contained and failure would 
be silent or misleading (e.g. _parse_overall_progress, executor_factory, 
the auralization match block).

### What Is Explicitly Excluded and Why

**Mesh generation, REST API layer, and acoustic output accuracy** — these 
are part of the foundation codebase provided to us. We treat them as 
verified externally and out of our testing responsibility.

**Cancellation end-to-end and frontend/discovery integration** — these are 
covered by manual testing performed separately. Automating these would 
require a running frontend and a live Celery worker, which we are not sure if we have enough time to make them.

**Concurrent simulations** — identified as out of scope for this project 
phase. The risk is acknowledged but deferred.

### Residual Risks
The following known gaps remain after our test suite:

- No timeout exists in `_poll_until_complete` — a silently crashed remote 
  job will cause indefinite polling. This is the highest residual risk and 
  is flagged as a bug to fix rather than a test to defer.
- `container.wait()` exit codes are not checked — a failed local simulation 
  can be silently marked as Completed.
- New simulation methods beyond DE and DG silently skip auralization due to 
  an incomplete match block.

These are documented as known bugs in Section 6 and are prioritised for 
resolution before production use.

## 3. Test Design Approach

The primary test design technique used in CHORAS is **equivalence partitioning**,
supplemented by **boundary value analysis** at partition edges. Equivalence
partitioning ensures that the test suite is comprehensive without being redundant,
by identifying groups of inputs or system states that the platform should handle
identically — meaning one representative case per partition is sufficient to
expose defects in that class. Boundary value analysis then targets the edges
between partitions, where defects are statistically most likely to occur.

EP and BVA determine *what data* to use;
TestCompass determines *which behavioral paths* to exercise with that data.

---

## 4. Equivalence Partitions

### 4.1 Simulation Method
| Partition ID | Class | Representative Value |
|---|---|---|
| EP-M1 | Computationally cheap method | `DE` |
| EP-M2 | Computationally expensive method | `DG` |
| EP-M3 | User-added new method | `MyNewMethod` |
| EP-M4 | Method not registered in discovery | `"UnknownMethod"` |
| EP-M5 | Method registered in config but `container_image` is `None` | `discover_container_image` returns `None` |
| EP-M6 | Method registered in config but `entry_file` is `None` | `discover_entry_file` returns `None` |

---

### 4.2 Execution Environment (ResourceType)
| Partition ID | Class | Representative Value |
|---|---|---|
| EP-E1 | Local execution (Docker) | `ResourceType.Local` |
| EP-E2 | Cloud execution (Singularity over SSH) | `ResourceType.Cloud` |
| EP-E3 | Invalid / unsupported resource type | `"GPU"`, `None` |

---

### 4.3 Input Geometry (.obj file)
| Partition ID | Class | Representative Value |
|---|---|---|
| EP-G1 | Simple geometry | `MeasurementRoom.obj` |
| EP-G2 | Moderately complex geometry | `Room2215_simple.obj` |
| EP-G3 | Complex geometry with absorption | `Room2215_withAbs.obj` |



---

### 4.4 Simulation Configuration (`sim_config`)
| Partition ID | Class | Representative Value |
|---|---|---|
| EP-C1 | Valid config with all required fields | `{"env": {"JSON_PATH": "/app/uploads/task1/input.json"}}` |
| EP-C2 | `JSON_PATH` missing from env | `{"env": {}}` |
| EP-C3 | `JSON_PATH` points to non-existent file | `{"env": {"JSON_PATH": "/app/uploads/missing.json"}}` |
| EP-C4 | `JSON_PATH` exists but file is unreadable (permissions) | `{"env": {"JSON_PATH": "/app/uploads/locked.json"}}` |
| EP-C5 | `JSON_PATH` exists but file is malformed JSON | `{"env": {"JSON_PATH": "/app/uploads/corrupt.json"}}` |
| EP-C6 | `solverSettings` is `None` | `simulation.solverSettings = None` |
| EP-C7 | `solverSettings` is malformed | `simulation.solverSettings = {"bad": "data"}` |

---

### 4.5 Docker / Local Executor State
| Partition ID | Class | Representative Value |
|---|---|---|
| EP-D1 | Docker daemon running, image present | Normal setup |
| EP-D2 | Docker daemon is down | `docker.from_env()` raises |
| EP-D3 | Container name already in use (duplicate run) | Same `simulation_id` run twice |
| EP-D4 | Host mount path cannot be resolved | No matching mount in `get_host_path_for_container_path` |
| EP-D5 | Container exits with non-zero status code | `container.wait()` returns `{"StatusCode": 1}` |

---

### 4.6 SSH / Cloud Executor State
| Partition ID | Class | Representative Value |
|---|---|---|
| EP-S1 | SSH connection healthy, all operations succeed | Normal cloud setup |
| EP-S2 | SSH authentication fails | `paramiko.AuthenticationException` |
| EP-S3 | SSH connection times out | `socket.timeout` |
| EP-S4 | SFTP upload fails mid-transfer | Network drop during `sftp.put` |
| EP-S5 | Remote disk full (Singularity build fails) | `SSHCommandError` on `_build_singularity_image` |
| EP-S6 | Remote sandbox already exists from a previous failed run | `_build_singularity_image` behaves unexpectedly |

---

### 4.7 Remote Job Progress (Polling)
| Partition ID | Class | Representative Value |
|---|---|---|
| EP-P1 | Progress increments normally and reaches 100% | `percentage`: 0 → 50 → 100 |
| EP-P2 | Progress reaches 100% on first poll | `percentage`: 100 immediately |
| EP-P3 | Remote JSON is temporarily corrupt (recovers within retries) | Corrupt on attempt 1, valid on attempt 2 |
| EP-P4 | Remote JSON is corrupt across all 3 retries every cycle | Never readable → infinite loop |
| EP-P5 | Remote simulation crashes silently, progress never reaches 100% | Stuck at e.g. 50% forever → infinite loop |
| EP-P6 | Cancel flag created before polling starts | `{task_id}.cancel` file exists at poll entry |
| EP-P7 | Cancel flag created mid-polling | `{task_id}.cancel` file created after cycle 2 |

---

### 4.8 Database State
| Partition ID | Class | Representative Value |
|---|---|---|
| EP-DB1 | `SimulationRun` exists in DB | Normal run |

---

### 4.9 Output and Post-Processing
| Partition ID | Class | Representative Value |
|---|---|---|
| EP-O1 | Output JSON and CSV present after completion | Normal completion |
| EP-O2 | Output JSON missing after completion | No `.json` in remote `/app` dir |
| EP-O3 | `ExportHelper.parse_json_file_to_xlsx_file` succeeds | Returns `True` |
| EP-O4 | `ExportHelper.parse_json_file_to_xlsx_file` fails | Returns `False` → `raise "Error..."` (invalid Python) |
| EP-O5 | `auralization_calculation` succeeds (DE) | WAV file written |
| EP-O6 | `auralization_calculation_DG` succeeds (DG) | WAV file written |
| EP-O7 | `auralization_calculation` raises after XLSX already written | Partial export left in DB |
| EP-O8 | `simulation_method` is not `DE` or `DG` | match falls through, no auralization, no error |

---

### 4.10 Discovery Service
| Partition ID | Class | Representative Value |
|---|---|---|
| EP-DS1 | All methods well-formed in repo | All appear correctly in frontend |
| EP-DS2 | New method added to repo | Appears in frontend after discovery |
| EP-DS3 | Existing method removed from repo | Disappears from frontend after discovery |
| EP-DS4 | Method definition malformed (e.g. missing `entryFile`) | Discovery does not crash, valid methods still shown |
| EP-DS5 | Zero methods in backend | Frontend shows empty list, no crash |

---

### TEST RESULTS

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
**Related documents**: `Test Design.md`, `Test Result.md`