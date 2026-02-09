# Testing Guide for Developers
## CHORAS Scalability Project - EngD 2026

---

## Repository Structure

```
backend/
├── app/                          # Main application code
├── tests/                        # All tests go here
│   ├── __init__.py
│   ├── conftest.py              # Shared fixtures
│   ├── test_acoustic_de.py      # DE simulation tests
│   ├── test_dg_simulation.py    # DG simulation tests
│   ├── test_pyroomacoustics.py  # Pyroomacoustics tests
│   ├── test_material.py         # Material tests
│   ├── test_api.py              # API endpoint tests
│   ├── integration/             # Integration tests (TBD)
│   ├── performance/             # Performance tests (TBD)
│   └── containers/              # Container tests (TBD)
├── requirements.txt             # Dependencies
└── pytest.ini                   # pytest configuration (if exists)
```

---

## Running Tests Locally

### Backend Tests (Python/pytest)

#### Setup
```bash
# Navigate to backend submodule
cd backend

# Install test dependencies (if not already installed)
pip install pytest pytest-cov pytest-mock pytest-benchmark
# Or if using --break-system-packages
pip install pytest pytest-cov pytest-mock pytest-benchmark --break-system-packages

# Verify pytest is installed
pytest --version
```

#### Run All Tests
```bash
# Run all tests in the tests/ directory
pytest

# Run with verbose output
pytest -v

# Run with very verbose output (shows each test)
pytest -vv
```

#### Run Specific Tests
```bash
# Run specific test file
pytest tests/test_acoustic_de.py

# Run specific test function
pytest tests/test_api.py::test_health_endpoint

# Run tests matching a pattern
pytest -k "simulation"  # Runs all tests with "simulation" in name
pytest -k "not slow"    # Skip tests marked as slow
```

#### Run with Coverage
```bash
# Run tests and generate coverage report
pytest --cov=app

# Generate HTML coverage report
pytest --cov=app --cov-report=html

# Open coverage report (HTML file will be in htmlcov/ folder)
open htmlcov/index.html  # macOS
xdg-open htmlcov/index.html  # Linux
start htmlcov/index.html  # Windows

# Show coverage in terminal
pytest --cov=app --cov-report=term

# Generate XML coverage (for CI/CD)
pytest --cov=app --cov-report=xml
```

#### Run Tests with Different Output Formats
```bash
# Show print statements during tests
pytest -s

# Stop at first failure
pytest -x

# Show local variables in tracebacks
pytest -l

# Run last failed tests only
pytest --lf

# Run failed tests first, then the rest
pytest --ff
```

---

### Frontend Tests (React/Jest)

#### Setup
```bash
# Navigate to frontend submodule
cd frontend-v2

# Install dependencies (includes test libraries)
npm install

# Verify tests can run
npm test -- --version
```

#### Run Tests
```bash
# Run all tests (interactive mode)
npm test

# Run all tests once (non-interactive)
npm test -- --watchAll=false

# Run with coverage
npm test -- --coverage --watchAll=false

# Run specific test file
npm test SimulationComponent.test.tsx

# Update snapshots (if using snapshot testing)
npm test -- -u
```

#### Coverage Report
```bash
# Generate coverage report
npm test -- --coverage --watchAll=false

# Coverage report will be in coverage/ folder
open coverage/lcov-report/index.html
```

---

### Running Tests in Docker

#### Using docker-compose.test.yml (Isolated Environment)

```bash
# From project root
cd CHORAS

# Start test environment and run tests
docker-compose -f docker-compose.test.yml up --abort-on-container-exit

# View test results
docker-compose -f docker-compose.test.yml logs

# Clean up after tests
docker-compose -f docker-compose.test.yml down -v
```

#### Manual Docker Testing
```bash
# Build test container
docker build -t choras-backend-test -f backend/Dockerfile backend/

# Run tests in container
docker run --rm choras-backend-test pytest -v

# Run with coverage
docker run --rm choras-backend-test pytest --cov=app --cov-report=term
```

---

## Writing Tests

### Backend Test Structure

#### Basic Test Example
```python
# tests/test_example.py
import pytest

def test_simple_assertion():
    """Simple test example"""
    result = 2 + 2
    assert result == 4

def test_string_operations():
    """Test string operations"""
    text = "CHORAS"
    assert text.lower() == "choras"
    assert len(text) == 6
```

#### Using Fixtures
```python
# tests/conftest.py (shared fixtures)
import pytest
from app import create_app
from app.models import db

@pytest.fixture
def app():
    """Create application for testing"""
    app = create_app('testing')
    return app

@pytest.fixture
def client(app):
    """Create test client"""
    return app.test_client()

@pytest.fixture
def database(app):
    """Create test database"""
    with app.app_context():
        db.create_all()
        yield db
        db.session.remove()
        db.drop_all()

# tests/test_api.py (using fixtures)
def test_health_endpoint(client):
    """Test health check endpoint"""
    response = client.get('/health')
    assert response.status_code == 200
    assert response.json['status'] == 'healthy'

def test_create_simulation(client, database):
    """Test creating a simulation"""
    response = client.post('/api/simulations', json={
        'method': 'DE',
        'geometry': 'test_room.obj',
        'params': {'characteristic_length': 3}
    })
    assert response.status_code == 201
    assert 'id' in response.json
```

#### Testing Celery Tasks
```python
# tests/test_celery_tasks.py
from app.tasks import run_de_simulation

def test_de_simulation_task_exists():
    """Test that DE simulation task is registered"""
    assert run_de_simulation is not None
    assert callable(run_de_simulation)

def test_de_simulation_task_execution():
    """Test task executes synchronously"""
    # Use .apply() to run without broker (synchronous)
    result = run_de_simulation.apply(
        args=['test_geometry.obj', {'characteristic_length': 3}]
    )
    
    assert result.state in ['SUCCESS', 'PENDING']
    # Add more assertions based on expected result
```

#### Testing with Mock Objects
```python
# tests/test_with_mocks.py
from unittest.mock import Mock, patch
import pytest

@patch('app.simulation.run_de_method')
def test_de_simulation_with_mock(mock_run_de):
    """Test DE simulation with mocked method"""
    # Configure mock
    mock_run_de.return_value = {'status': 'completed', 'result': 0.5}
    
    # Call function that uses run_de_method
    from app.simulation import execute_simulation
    result = execute_simulation('DE', 'test.obj')
    
    # Assertions
    assert result['status'] == 'completed'
    mock_run_de.assert_called_once()
```

#### Parameterized Tests
```python
# tests/test_parameterized.py
import pytest

@pytest.mark.parametrize("input,expected", [
    (2, 4),
    (3, 9),
    (4, 16),
    (5, 25),
])
def test_square_numbers(input, expected):
    """Test squaring different numbers"""
    assert input ** 2 == expected

@pytest.mark.parametrize("method,geometry", [
    ("DE", "room_small.obj"),
    ("DG", "room_large.obj"),
    ("Pyroomacoustics", "concert_hall.obj"),
])
def test_simulation_methods(method, geometry):
    """Test different simulation methods"""
    # Test logic here
    pass
```

---

### Frontend Test Structure

#### Component Test Example (React Testing Library)
```tsx
// SimulationButton.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import SimulationButton from './SimulationButton';

test('renders simulation button', () => {
  render(<SimulationButton />);
  const buttonElement = screen.getByText(/Start Simulation/i);
  expect(buttonElement).toBeInTheDocument();
});

test('button click calls handler', () => {
  const handleClick = jest.fn();
  render(<SimulationButton onClick={handleClick} />);
  
  const button = screen.getByText(/Start Simulation/i);
  fireEvent.click(button);
  
  expect(handleClick).toHaveBeenCalledTimes(1);
});

test('button is disabled when loading', () => {
  render(<SimulationButton isLoading={true} />);
  const button = screen.getByText(/Loading.../i);
  expect(button).toBeDisabled();
});
```

---

## Test Naming Conventions

### Backend (Python)
- **File naming**: `test_<module_name>.py`
  - Example: `test_api.py`, `test_celery_tasks.py`
- **Function naming**: `test_<what_is_being_tested>_<scenario>`
  - Example: `test_de_simulation_with_valid_input`
  - Example: `test_api_returns_404_for_invalid_simulation_id`

### Frontend (TypeScript/JavaScript)
- **File naming**: `<ComponentName>.test.tsx` or `<ComponentName>.test.jsx`
  - Example: `SimulationButton.test.tsx`
- **Test description**: Use descriptive strings
  - Example: `test('renders correctly when loading', ...)`

---

## Best Practices

### General Testing Principles
1. ✅ **Arrange, Act, Assert** (AAA Pattern)
   ```python
   def test_example():
       # Arrange - set up test data
       user = create_test_user()
       
       # Act - perform the action
       result = user.calculate_score()
       
       # Assert - verify the result
       assert result > 0
   ```

2. ✅ **One assertion per test** (ideally)
   - Each test should verify one behavior
   - Makes failures easier to diagnose

3. ✅ **Use descriptive test names**
   - Good: `test_de_simulation_fails_with_invalid_geometry`
   - Bad: `test_1`, `test_simulation`

4. ✅ **Test edge cases**
   - Empty inputs
   - Invalid data types
   - Boundary values
   - Error conditions

5. ✅ **Keep tests independent**
   - Tests should not depend on each other
   - Should run in any order
   - Use fixtures for shared setup

### Writing Good Tests
```python
# ❌ BAD TEST
def test_stuff():
    x = thing()
    y = x.do_stuff()
    assert y  # What are we testing?

# ✅ GOOD TEST
def test_de_simulation_returns_reverberation_time():
    """Test that DE simulation calculates RT60"""
    # Arrange
    geometry = load_test_geometry('room_small.obj')
    params = {'characteristic_length': 3}
    
    # Act
    result = run_de_simulation(geometry, params)
    
    # Assert
    assert 'RT60' in result
    assert isinstance(result['RT60'], float)
    assert result['RT60'] > 0
```

---

## Before Submitting a PR

### Checklist
- [ ] All tests pass locally: `pytest`
- [ ] New tests added for new features
- [ ] Test coverage hasn't decreased: `pytest --cov`
- [ ] Tests are documented (docstrings)
- [ ] No skipped tests without good reason
- [ ] Complex tests have comments explaining logic
- [ ] Fixtures are used for repeated setup
- [ ] Tests run in reasonable time (<5 min total)

### Running Full Validation
```bash
# Backend - full validation
cd backend
pytest -v --cov=app --cov-report=term
flake8 .  # Linting (if configured)
black --check .  # Code formatting (if configured)

# Frontend - full validation
cd frontend-v2
npm test -- --coverage --watchAll=false
npm run lint  # Linting (if configured)
```

---

## Continuous Integration (CI)

### GitHub Actions
Tests run automatically on:
- Every push to `main` or `dev` branches
- Every pull request to `main` or `dev`

View test results:
- Go to https://github.com/Saptarshi666/CHORAS/actions
- Click on the latest workflow run
- See test results and coverage

### What CI Checks
1. ✅ All unit tests pass
2. ✅ All integration tests pass (when PRing to dev/main)
3. ✅ Code coverage meets threshold
4. ✅ Docker containers build successfully
5. ✅ Linting passes (if configured)

---

## Debugging Failed Tests

### Reading pytest Output
```bash
# Run test with full output
pytest tests/test_api.py -vv

# Show local variables on failure
pytest tests/test_api.py -l

# Show print statements
pytest tests/test_api.py -s

# Start debugger on failure
pytest tests/test_api.py --pdb
```

### Common Issues

#### Import Errors
```python
# Error: ModuleNotFoundError: No module named 'app'
# Solution: Make sure you're in the backend/ directory
cd backend
pytest
```

#### Database/Fixture Errors
```python
# Error: fixture 'database' not found
# Solution: Check conftest.py has the fixture
# Solution: Make sure __init__.py exists in tests/
```

#### Docker Test Failures
```bash
# Error: Cannot connect to Docker daemon
# Solution: Make sure Docker Desktop is running

# Error: Container fails to start
# Solution: Check logs
docker-compose -f docker-compose.test.yml logs
```

---

## Performance Testing

### Using pytest-benchmark
```python
# tests/performance/test_benchmarks.py
def test_de_simulation_performance(benchmark):
    """Benchmark DE simulation execution time"""
    geometry = load_test_geometry()
    params = {'characteristic_length': 3}
    
    # benchmark() will run the function multiple times
    result = benchmark(run_de_simulation, geometry, params)
    
    assert result is not None
```

### Running Benchmarks
```bash
# Run benchmark tests
pytest tests/performance/ --benchmark-only

# Compare benchmarks
pytest tests/performance/ --benchmark-compare

# Save benchmark results
pytest tests/performance/ --benchmark-save=baseline
```

---

## Resources

### Documentation
- **pytest docs**: https://docs.pytest.org/
- **React Testing Library**: https://testing-library.com/react
- **Docker testing**: https://docs.docker.com/

### Getting Help
- **Ask in team chat**: Slack/Teams channel
- **Create GitHub issue**: For bugs or questions
- **Check test logs**: `pytest -vv` for detailed output
- **Contact Test Manager**: [Your name] for testing strategy questions

---

**Last Updated**: February 6, 2026  
**Maintained by**: Test Manager  
**Questions**: Ask in team channel or create GitHub issue