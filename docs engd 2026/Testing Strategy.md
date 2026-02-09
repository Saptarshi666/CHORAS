# Testing Strategy
## CHORAS Scalability Project - EngD 2026

---

## My Role Responsibilities (from project brief)

As Test Manager, I am responsible for:
- ✅ Management of testing activities and resources
- ✅ Ensure all developed artifacts are properly verified
- ✅ Define the test strategy
- ✅ Perform overall system validation of customer deliverable
- ✅ Work closely with Project Manager
- ✅ Coordinate with Quality Manager on quality assurance

---

## 1. Current State Analysis (Actual Findings)

### Existing Tests in Backend
Based on analysis of `backend/tests/`:

**Simulation Method Tests:**
- ✅ Acoustic Differential Equation (DE) tests
- ✅ Discontinuous Galerkin (DG) simulation tests
- ✅ Pyroomacoustics simulation tests
- ✅ Material creation tests
- ❌ No tests for MyNewMethod (newly added)

**API & Service Tests:**
- ✅ All API service tests present
- ✅ Application creation/destruction tests (backend, database, Celery)
- ✅ General load tests

**Integration Tests:**
- ⚠️ Integration folder exists but has no tests

### Test Frameworks
- **Backend**: pytest (confirmed in requirements.txt)
- **Frontend**: Unknown - needs investigation in `frontend-v2/`

### Test Coverage
- **Current**: To be measured (run `pytest --cov` in backend/)
- **Baseline**: Will establish in Week 1
- **Target**: 70%+ by end of project

### Celery Task Testing
- **Current state**: Needs investigation
- **Location**: Check `backend/app/` for Celery task definitions
- **Priority**: High (critical for containerization project)

---

## 2. Testing Levels for Containerized Architecture

### Unit Tests
**Status**: ✅ Exist (need to measure coverage)

- **Target**: Individual functions and components
- **Responsibility**: Each developer writes for their code
- **Tool**: pytest (backend), Jest or React Testing Library (frontend - TBD)
- **Coverage goal**: 70%+ (will adjust based on current coverage)
- **Celery Tasks**: Test individual task functions using `task.apply()` without full broker

**Current Unit Tests:**
```
backend/tests/
├── test_acoustic_de.py          # DE method tests
├── test_dg_simulation.py        # DG method tests
├── test_pyroomacoustics.py      # Pyroomacoustics tests
├── test_material.py             # Material tests
└── test_api.py                  # API endpoint tests
```

**Gaps to Fill:**
- [ ] Tests for `MyNewMethod/`
- [ ] Celery task unit tests
- [ ] Frontend component tests

---

### Integration Tests
**Status**: ⚠️ Folder exists but empty - HIGH PRIORITY

- **Target**: 
  - API endpoints with database interactions
  - Celery task submission and completion workflow
  - **CRITICAL FOR PROJECT**: CHORAS main container → Simulation containers communication
  
- **Responsibility**: Test Manager (me) to coordinate
- **Tool**: pytest with fixtures, Docker Compose for multi-container tests
- **Coverage goal**: All key workflows covered

**Integration Tests to Create:**
```
backend/tests/integration/
├── test_de_container_integration.py    # Main → DE container
├── test_dg_container_integration.py    # Main → DG container
├── test_celery_workflow.py             # End-to-end task workflow
├── test_database_integration.py        # API ↔ Database
└── test_simulation_pipeline.py         # Full simulation flow
```

**Test Scenarios:**
1. Submit simulation via API → Celery picks it up → Container runs → Results returned
2. Test that main container can spin up DE/DG containers
3. Verify container orchestration via Celery
4. Database persistence across simulation runs

---

### System Tests (End-to-End)
**Status**: ❌ Need to create

- **Target**: Full simulation workflows from user perspective
  - User submits simulation → Celery schedules → Container executes → Results displayed
  
- **Responsibility**: Test Manager
- **Tool**: 
  - pytest with Docker Compose
  - Selenium/Playwright for UI testing (if in scope - TBD with team)
  
- **Coverage goal**: Main user journeys

**User Journeys to Test:**
1. Submit DE simulation with default geometry
2. Submit DG simulation with custom parameters
3. View simulation results
4. Compare multiple simulations
5. Handle simulation failures gracefully

---

### Performance Tests (CRITICAL FOR SCALABILITY PROJECT!)
**Status**: ⚠️ Basic load tests exist, need enhancement

- **Target**: 
  - Concurrent simulation handling (current vs. containerized)
  - Container startup/teardown time
  - Memory usage per container
  - HPC offloading (future - Week 6-7)
  
- **Responsibility**: Test Manager with dev team
- **Tool**: 
  - `locust` for API load testing
  - `pytest-benchmark` for function-level benchmarks
  - `docker stats` for resource monitoring
  - Custom scripts for container metrics
  
- **Metrics to Track**:
  - Response time for API endpoints (baseline vs. current)
  - Simulation completion time (DE vs. DG)
  - Number of concurrent simulations supported
  - Container resource usage (CPU, memory)
  - Celery queue throughput (tasks/second)
  - Container startup time (target: <5 seconds)

**Performance Test Files to Create:**
```
backend/tests/performance/
├── test_api_load.py                    # API load testing
├── test_concurrent_simulations.py      # Multiple simulations
├── test_container_startup.py           # Container boot time
├── test_memory_usage.py                # Resource monitoring
└── locustfile.py                       # Locust load test scenarios
```

---

### Container-Specific Tests (NEW FOR OUR PROJECT!)
**Status**: ❌ Need to create (Week 2-4)

- **Target**: Individual simulation method containers (DE, DG, future methods)
- **Test scenarios**:
  - Container builds successfully from Dockerfile
  - Container accepts input parameters (JSON/file)
  - Container produces expected output format
  - Container handles errors gracefully (bad input, crashes)
  - Container can be orchestrated by Celery
  - Container cleanup after execution

- **Tool**: pytest with Docker SDK for Python

**Container Test Files to Create:**
```
backend/tests/containers/
├── test_de_container.py                # DE container tests
├── test_dg_container.py                # DG container tests
├── test_container_orchestration.py     # Celery → container
└── test_container_cleanup.py           # Resource cleanup
```

**Example Container Test:**
```python
import docker
import pytest

@pytest.fixture
def docker_client():
    return docker.from_env()

def test_de_container_builds(docker_client):
    """Test that DE container builds successfully"""
    image, logs = docker_client.images.build(
        path="./simulation-methods/de/",
        tag="choras-de-simulation:test"
    )
    assert image is not None

def test_de_container_runs(docker_client):
    """Test that DE container can start and run"""
    container = docker_client.containers.run(
        'choras-de-simulation:test',
        detach=True,
        environment={"INPUT_FILE": "/data/test.json"}
    )
    container.wait(timeout=60)
    assert container.status != 'error'
    container.remove()
```

---

## 3. Test Environment Setup

### Directory Structure
```
backend/
├── tests/
│   ├── __init__.py
│   ├── conftest.py                     # Shared fixtures
│   ├── test_acoustic_de.py             ✅ Exists
│   ├── test_dg_simulation.py           ✅ Exists
│   ├── test_api.py                     ✅ Exists
│   ├── integration/                    ⚠️ Empty - needs tests
│   │   ├── __init__.py
│   │   └── (integration tests here)
│   ├── performance/                    ❌ Create
│   │   ├── __init__.py
│   │   └── (performance tests here)
│   └── containers/                     ❌ Create (Week 2)
│       ├── __init__.py
│       └── (container tests here)
```

### Test Environments
- **Local Development**: Run tests with `pytest` in backend/
- **Docker Isolated**: Use `docker-compose.test.yml` (already exists!)
- **CI/CD**: GitHub Actions (workflow already created)
- **Simulation Containers**: Separate test fixtures for DE and DG

### Test Configuration Files
Located in `backend/`:
- `pytest.ini` or `setup.cfg` - pytest configuration
- `.env.test` - Test environment variables
- `conftest.py` - Shared pytest fixtures

---

## 4. Testing Schedule

### On Every Commit (Automated)
- ✅ Unit tests run (fast, <2 min)
- ✅ Linting checks (flake8, black)
- ✅ Basic smoke tests

### On Pull Request to `dev` (Automated)
- ✅ All unit tests
- ✅ Integration tests (moderate, ~5 min)
- ✅ Container tests (if container code changed)
- ✅ Code coverage report

### On Merge to `dev` (Automated)
- ✅ Full test suite including system tests (~15 min)
- ✅ Docker build verification
- ✅ Performance regression checks

### Weekly (Manual/Scheduled)
- ✅ Performance tests on dev branch
- ✅ Load testing with `locust`
- ✅ Container resource monitoring
- ✅ Baseline measurements documented

### Before Client Demos
- ✅ Full system test
- ✅ Performance benchmarks
- ✅ All containers build successfully
- ✅ End-to-end user journeys verified

---

## 5. Defect Tracking (Your Responsibility!)

### Tool
**GitHub Issues**: https://github.com/Saptarshi666/CHORAS/issues

### Labels to Use
Create these labels in GitHub:
- `bug` - General bugs (red)
- `test-failure` - Tests failing (orange)
- `performance-issue` - Scalability problems (yellow)
- `container-issue` - Docker/containerization problems (purple)
- `celery-issue` - Task queue problems (green)
- `integration-issue` - Integration test failures (blue)

### Priority Levels
- **Critical (P0)**: Blocking, cannot proceed
- **High (P1)**: Important for project goals
- **Medium (P2)**: Should fix soon
- **Low (P3)**: Nice to have

### Workflow
1. **Developer** creates issue using bug report template
2. **Test Manager** (you) triages and assigns priority
3. **Team** assigns to developer
4. **Developer** fixes and creates PR
5. **Test Manager** verifies fix with tests
6. **Close** issue after verification

---

## 6. Entry/Exit Criteria for Releases

### Entry Criteria (Before Starting Sprint)
- [ ] All user stories have acceptance criteria
- [ ] Test environment is available and working
- [ ] Required tools are set up (pytest, docker, etc.)
- [ ] Team has access to test data/geometries
- [ ] Previous sprint's critical bugs are resolved

### Exit Criteria (Before Merging/Releasing)
- [ ] **Code Review**: Minimum 1 reviewer approval
- [ ] **All Tests Pass**: Unit, integration, system tests green
- [ ] **No Critical Bugs**: No P0 bugs open, P1 bugs documented
- [ ] **Documentation**: Updated for new features
- [ ] **Performance**: No regression in key metrics
- [ ] **Container Tests**: All containerization tests pass (if applicable)
- [ ] **Test Coverage**: Hasn't decreased from baseline

---

## 7. Special Testing Considerations for Project

### Celery Task Testing
Test Celery tasks without running full broker:

```python
# backend/tests/test_celery_tasks.py
from celery import current_app
from app.tasks import run_de_simulation

def test_de_simulation_task_structure():
    """Test that DE simulation task is properly defined"""
    assert hasattr(run_de_simulation, 'delay')
    assert run_de_simulation.name is not None

def test_de_simulation_task_execution():
    """Test task executes with test data"""
    # Use .apply() to run synchronously without broker
    result = run_de_simulation.apply(
        args=[test_geometry_path, test_params]
    )
    assert result.state == 'SUCCESS'
    assert result.result is not None
```

### Docker Container Testing
Test containers using Docker SDK:

```python
# backend/tests/containers/test_de_container.py
import docker
import pytest
import json

@pytest.fixture
def docker_client():
    return docker.from_env()

def test_de_container_starts(docker_client):
    """Test DE container can be started"""
    container = docker_client.containers.run(
        'choras-de-simulation:latest',
        detach=True,
        environment={"TEST_MODE": "true"}
    )
    
    # Wait for container to start
    container.reload()
    assert container.status == 'running'
    
    # Cleanup
    container.stop()
    container.remove()

def test_de_container_accepts_input(docker_client):
    """Test container accepts and processes input"""
    test_input = {
        "geometry": "room_small.obj",
        "params": {"characteristic_length": 3}
    }
    
    # Mount test data volume
    container = docker_client.containers.run(
        'choras-de-simulation:latest',
        detach=True,
        volumes={'/path/to/test/data': {'bind': '/data', 'mode': 'rw'}},
        environment={"INPUT_JSON": json.dumps(test_input)}
    )
    
    # Wait for completion
    exit_code = container.wait(timeout=300)
    assert exit_code['StatusCode'] == 0
    
    container.remove()
```

### Multi-Container Integration Testing
Test container orchestration:

```bash
# Run integration tests with docker-compose
cd backend
docker-compose -f ../docker-compose.test.yml up -d
pytest tests/integration/ -v
docker-compose -f ../docker-compose.test.yml down -v
```

```python
# backend/tests/integration/test_simulation_pipeline.py
import requests
import time

def test_full_de_simulation_pipeline():
    """Test complete DE simulation from API to result"""
    
    # Submit simulation via API
    response = requests.post(
        'http://localhost:5000/api/simulations',
        json={
            'method': 'DE',
            'geometry': 'test_room.obj',
            'params': {'characteristic_length': 3}
        }
    )
    assert response.status_code == 201
    simulation_id = response.json()['id']
    
    # Wait for Celery to process
    time.sleep(10)
    
    # Check simulation status
    status_response = requests.get(
        f'http://localhost:5000/api/simulations/{simulation_id}'
    )
    assert status_response.status_code == 200
    assert status_response.json()['status'] in ['completed', 'processing']
```

---

## 8. Test Metrics to Track (Report Weekly)

### Coverage Metrics
- **Overall test coverage**: X% (run `pytest --cov`)
- **Unit test coverage**: Y%
- **Integration test coverage**: Z%
- **Lines covered vs. total lines**

### Test Count
- **Total tests**: N
- **Unit tests**: N₁
- **Integration tests**: N₂
- **Performance tests**: N₃
- **Container tests**: N₄

### Test Execution
- **Test execution time**: T seconds
- **Longest running test**: X seconds
- **Failed tests**: N failures
- **Skipped tests**: N skips

### Bug Metrics
- **Open bugs by priority**: 
  - P0 (Critical): N
  - P1 (High): N
  - P2 (Medium): N
  - P3 (Low): N
- **Average time to fix**: X days
- **Bug discovery rate**: N bugs/week

### Performance Metrics (Baseline → Current)
- **Container startup time**: 8s → 5s (target)
- **DE simulation time**: 120s → ? (measure)
- **DG simulation time**: 240s → ? (measure)
- **Concurrent simulations**: 1 → 10 (target)
- **API response time**: 200ms → ? (monitor)

---

## 9. Action Items for Test Manager

### Week 1 (Feb 2-9) - CURRENT WEEK
- [x] Audit existing tests
- [x] Measure current test coverage
- [ ] Create Testing Strategy document (this doc)
- [ ] Set up test coverage reporting
- [ ] Document baseline performance metrics
- [ ] Create GitHub issue labels

### Week 2 (Feb 10-16)
- [ ] Create integration tests folder structure
- [ ] Write first container tests
- [ ] Set up performance testing framework (locust)
- [ ] Establish baseline metrics
- [ ] Create test data fixtures

### Week 3-4 (Feb 17-Mar 2)
- [ ] Write DE container integration tests
- [ ] Write DG container integration tests
- [ ] Performance benchmarking
- [ ] Celery task testing
- [ ] Weekly test metrics reporting

### Week 5-6 (Mar 3-16)
- [ ] System test creation
- [ ] Load testing
- [ ] Performance optimization validation
- [ ] Cloud deployment testing

### Week 7 (Mar 17-23)
- [ ] Final test suite validation
- [ ] Demo preparation
- [ ] Test documentation complete
- [ ] Handover to client

---

**Last Updated**: February 6, 2026  
**Test Manager**: Saptarshi Mondal
**Status**: Living document - updated weekly  
**Next Review**: After first test coverage measurement