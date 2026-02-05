# Testing Guide for Developers
   
   ## Running Tests Locally
   
   ### Backend Tests
```bash
   # Navigate to backend
   cd backend
   
   # Install test dependencies
   pip install pytest pytest-cov pytest-mock
   
   # Run all tests
   pytest
   
   # Run with coverage
   pytest --cov=.
   
   # Run specific test file
   pytest tests/test_api.py
```
   
   ### Frontend Tests
```bash
   # Navigate to frontend
   cd frontend-v2
   
   # Install dependencies
   npm install
   
   # Run tests
   npm test
   
   # Run with coverage
   npm test -- --coverage
```
   
   ### Running in Docker
```bash
   # Run test suite in isolated environment
   docker-compose -f docker-compose.test.yml up --abort-on-container-exit
```
   
   ## Writing Tests
   
   ### Backend Test Example
```python
   import pytest
   from app import create_app
   
   @pytest.fixture
   def client():
       app = create_app('testing')
       with app.test_client() as client:
           yield client
   
   def test_health_endpoint(client):
       response = client.get('/health')
       assert response.status_code == 200
       assert response.json['status'] == 'healthy'
```
   
   ### Frontend Test Example
```javascript
   import { render, screen } from '@testing-library/react';
   import SimulationComponent from './SimulationComponent';
   
   test('renders simulation button', () => {
     render(<SimulationComponent />);
     const buttonElement = screen.getByText(/Start Simulation/i);
     expect(buttonElement).toBeInTheDocument();
   });
```
   
   ## Test Naming Conventions
   - Backend: `test_<feature>_<scenario>.py`
   - Frontend: `<Component>.test.tsx` or `<Component>.test.jsx`
   
   ## Before Submitting a PR
   - [ ] All tests pass locally
   - [ ] New tests added for new features
   - [ ] Code coverage hasn't decreased
   - [ ] Tests documented if complex