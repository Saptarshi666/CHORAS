# Scalability Test Plan
## CHORAS Scalability Project - EngD 2026

---

## Project Goal

**Primary Objective**: Improve CHORAS scalability through containerization and cloud deployment

**Success Criteria**:
- Support 10+ concurrent simulations (vs. current 1-2)
- Reduce container startup time to <5 seconds
- Enable cloud deployment (SURF research cloud)
- Each simulation method in its own isolated container

---

## Performance Metrics to Track

### 1. Response Time

#### API Endpoints
| Endpoint | Baseline | Target | Measurement |
|----------|----------|--------|-------------|
| GET `/health` | TBD | <100ms | Monitor |
| POST `/api/simulations` | TBD | <500ms | Monitor |
| GET `/api/simulations/{id}` | TBD | <200ms | Monitor |
| GET `/api/simulations/{id}/results` | TBD | <1s | Monitor |

**How to measure**:
```bash
# Use curl with timing
curl -w "@curl-format.txt" -o /dev/null -s http://localhost:5000/health

# Or use Apache Bench
ab -n 1000 -c 10 http://localhost:5000/health
```

#### Simulation Completion Time
| Method | Geometry | Baseline | Target | Notes |
|--------|----------|----------|--------|-------|
| DE | small_room.obj | TBD | Maintain | Should not regress |
| DE | large_room.obj | TBD | Maintain | Should not regress |
| DG | small_room.obj | TBD | Maintain | Should not regress |
| DG | large_room.obj | TBD | Maintain | Should not regress |

**Important**: Containerization should NOT make simulations slower!

---

### 2. Throughput

#### Concurrent Simulations
- **Current (Baseline)**: 1-2 simultaneous simulations
- **Target**: 10+ simultaneous simulations
- **Measurement**: Number of simulations running concurrently without errors

#### Requests Per Second (API)
- **Baseline**: TBD (measure with locust)
- **Target**: 50 req/s for simple endpoints
- **Target**: 10 req/s for simulation submission

**How to measure**:
```bash
# Use locust for load testing
cd backend/tests/performance
locust -f locustfile.py --host=http://localhost:5000
```

---

### 3. Resource Usage

#### Container Resources
Monitor each container individually:

| Container | CPU Limit | Memory Limit | Current Usage |
|-----------|-----------|--------------|---------------|
| Backend API | 2 cores | 2GB | TBD |
| Database | 1 core | 1GB | TBD |
| Celery Worker | 2 cores | 2GB | TBD |
| Redis | 0.5 core | 512MB | TBD |
| DE Container | 4 cores | 4GB | TBD |
| DG Container | 4 cores | 4GB | TBD |

**How to measure**:
```bash
# Monitor resource usage
docker stats

# Monitor specific container
docker stats choras_backend_1

# Export to CSV for analysis
docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}" > metrics.csv
```

#### System Resources
- **CPU utilization**: Keep <80% under load
- **Memory usage**: Keep <80% of available RAM
- **Disk I/O**: Monitor for bottlenecks
- **Network**: Monitor for bandwidth limits

---

### 4. Container-Specific Metrics

#### Startup Time
| Container | Current | Target | Critical? |
|-----------|---------|--------|-----------|
| Backend API | TBD | <10s | Medium |
| DE Container | TBD | <5s | **High** |
| DG Container | TBD | <5s | **High** |

**Why critical**: Fast startup = better user experience, lower resource waste

**How to measure**:
```python
import time
import docker

client = docker.from_env()

start = time.time()
container = client.containers.run('choras-de-simulation:latest', detach=True)
container.wait(timeout=30)
end = time.time()

print(f"Startup time: {end - start:.2f}s")
```

#### Container Cleanup
- **Test**: Containers are removed after simulation completes
- **Test**: No orphaned containers after 100 simulations
- **Test**: Memory is properly released

---

### 5. Database Performance

| Metric | Current | Target |
|--------|---------|--------|
| Connection pool size | TBD | Optimize |
| Avg query time | TBD | <50ms |
| Concurrent connections | TBD | 20+ |

**How to measure**:
```bash
# Monitor PostgreSQL
docker exec -it choras_db_1 psql -U choras_user -d choras

# Check active connections
SELECT count(*) FROM pg_stat_activity;

# Check slow queries
SELECT * FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;
```

---

### 6. Celery Queue Throughput

| Metric | Current | Target |
|--------|---------|--------|
| Tasks per minute | TBD | 60+ |
| Queue depth | TBD | <10 |
| Task success rate | TBD | >95% |
| Avg task execution time | TBD | Document |

**How to measure**:
```bash
# Monitor Celery worker
docker logs choras_celery_1 --tail=100 --follow

# Or use Flower (Celery monitoring tool)
pip install flower
celery -A app.celery flower
# Visit http://localhost:5555
```

---

## Test Scenarios

### Scenario 1: Single User Load (Baseline)
**Objective**: Establish baseline performance metrics

**Steps**:
1. Start one DE simulation with small geometry
2. Measure completion time
3. Measure resource usage during simulation
4. Record API response times
5. Verify results are correct

**Expected Results**:
- Simulation completes successfully
- Resource usage is acceptable
- API responds within target times

**Success Criteria**:
- [ ] Simulation completes without errors
- [ ] Completion time < baseline + 10%
- [ ] Memory usage < 4GB
- [ ] CPU usage < 80%

**Test Script**:
```python
# tests/performance/test_single_user.py
import requests
import time
import psutil

def test_single_de_simulation():
    # Start simulation
    start = time.time()
    response = requests.post('http://localhost:5000/api/simulations', json={
        'method': 'DE',
        'geometry': 'small_room.obj',
        'params': {'characteristic_length': 3}
    })
    assert response.status_code == 201
    sim_id = response.json()['id']
    
    # Monitor resources
    process = psutil.Process()
    
    # Wait for completion
    while True:
        status = requests.get(f'http://localhost:5000/api/simulations/{sim_id}')
        if status.json()['status'] == 'completed':
            break
        time.sleep(5)
    
    end = time.time()
    
    # Record metrics
    print(f"Completion time: {end - start:.2f}s")
    print(f"Memory: {process.memory_info().rss / 1024 / 1024:.2f}MB")
```

---

### Scenario 2: Multiple Concurrent Users
**Objective**: Test concurrent simulation handling

**Steps**:
1. Start N simultaneous simulations (N = 5, 10, 20)
2. Measure completion times for each
3. Monitor for failures or errors
4. Check resource usage across all containers

**Expected Results**:
- All simulations complete successfully
- No significant slowdown (<20% increase in time)
- No container crashes
- Resource usage scales linearly

**Success Criteria**:
- [ ] All 10 concurrent simulations complete
- [ ] No errors in logs
- [ ] Average completion time < single user time × 1.2
- [ ] System remains responsive (API <1s response)

**Test Script**:
```python
# tests/performance/test_concurrent.py
import requests
import concurrent.futures
import time

def submit_simulation(sim_number):
    start = time.time()
    response = requests.post('http://localhost:5000/api/simulations', json={
        'method': 'DE',
        'geometry': 'test_room.obj',
        'params': {'characteristic_length': 3}
    })
    sim_id = response.json()['id']
    
    # Wait for completion
    while True:
        status = requests.get(f'http://localhost:5000/api/simulations/{sim_id}')
        if status.json()['status'] == 'completed':
            break
        time.sleep(5)
    
    end = time.time()
    return {'sim': sim_number, 'time': end - start}

def test_10_concurrent_simulations():
    num_simulations = 10
    
    with concurrent.futures.ThreadPoolExecutor(max_workers=num_simulations) as executor:
        results = list(executor.map(submit_simulation, range(num_simulations)))
    
    # Analyze results
    times = [r['time'] for r in results]
    print(f"Average time: {sum(times)/len(times):.2f}s")
    print(f"Max time: {max(times):.2f}s")
    print(f"Min time: {min(times):.2f}s")
    
    assert all(t < 300 for t in times), "Some simulations took too long"
```

---

### Scenario 3: Sustained Load
**Objective**: Test stability over time

**Steps**:
1. Run continuous simulations for 1 hour
2. Submit new simulation every 2 minutes
3. Monitor for memory leaks
4. Check for performance degradation
5. Monitor error rate

**Expected Results**:
- System remains stable
- No memory leaks (memory usage plateaus)
- Performance doesn't degrade over time
- Error rate stays below 1%

**Success Criteria**:
- [ ] System runs for 1 hour without crashes
- [ ] Memory usage increases <10% over time
- [ ] Response times remain consistent
- [ ] Success rate >99%

**Test Script**:
```python
# tests/performance/test_sustained_load.py
import requests
import time
import psutil
import matplotlib.pyplot as plt

def test_sustained_load_1_hour():
    duration = 3600  # 1 hour
    interval = 120   # 2 minutes
    
    start_time = time.time()
    metrics = {'time': [], 'memory': [], 'cpu': []}
    
    while time.time() - start_time < duration:
        # Submit simulation
        response = requests.post('http://localhost:5000/api/simulations', json={
            'method': 'DE',
            'geometry': 'test_room.obj'
        })
        
        # Record metrics
        elapsed = time.time() - start_time
        metrics['time'].append(elapsed)
        metrics['memory'].append(psutil.virtual_memory().percent)
        metrics['cpu'].append(psutil.cpu_percent())
        
        # Wait before next submission
        time.sleep(interval)
    
    # Plot results
    plt.figure(figsize=(12, 4))
    plt.subplot(1, 2, 1)
    plt.plot(metrics['time'], metrics['memory'])
    plt.title('Memory Usage Over Time')
    
    plt.subplot(1, 2, 2)
    plt.plot(metrics['time'], metrics['cpu'])
    plt.title('CPU Usage Over Time')
    
    plt.savefig('sustained_load_metrics.png')
```

---

### Scenario 4: Container Startup Performance
**Objective**: Measure and optimize container startup time

**Steps**:
1. Start 10 DE containers sequentially
2. Measure startup time for each
3. Calculate average and variance
4. Identify bottlenecks

**Expected Results**:
- Average startup time <5 seconds
- Consistent startup times (low variance)
- No startup failures

**Success Criteria**:
- [ ] 10/10 containers start successfully
- [ ] Average startup time <5s
- [ ] Standard deviation <1s

**Test Script**:
```python
# tests/performance/test_container_startup.py
import docker
import time
import statistics

def test_container_startup_times():
    client = docker.from_env()
    startup_times = []
    
    for i in range(10):
        start = time.time()
        container = client.containers.run(
            'choras-de-simulation:latest',
            detach=True,
            remove=True
        )
        
        # Wait for container to be ready
        container.reload()
        while container.status != 'running':
            time.sleep(0.1)
            container.reload()
        
        end = time.time()
        startup_time = end - start
        startup_times.append(startup_time)
        
        container.stop()
        print(f"Container {i+1}: {startup_time:.2f}s")
    
    avg = statistics.mean(startup_times)
    std = statistics.stdev(startup_times)
    
    print(f"\nAverage: {avg:.2f}s")
    print(f"Std Dev: {std:.2f}s")
    print(f"Min: {min(startup_times):.2f}s")
    print(f"Max: {max(startup_times):.2f}s")
    
    assert avg < 5.0, f"Average startup time {avg:.2f}s exceeds 5s target"
```

---

## Tools

### Load Testing
- **locust**: API load testing
  ```bash
  pip install locust
  locust -f tests/performance/locustfile.py
  ```

### Resource Monitoring
- **docker stats**: Container resource usage
  ```bash
  docker stats --no-stream
  ```
- **psutil**: Python library for system monitoring
  ```python
  import psutil
  print(psutil.cpu_percent())
  print(psutil.virtual_memory())
  ```

### Performance Profiling
- **pytest-benchmark**: Function-level benchmarks
  ```bash
  pytest tests/performance/ --benchmark-only
  ```
- **cProfile**: Python profiling
  ```python
  python -m cProfile -o output.prof script.py
  ```

### Visualization
- **matplotlib**: Plot metrics over time
- **Grafana**: Real-time dashboard (advanced)
- **Prometheus**: Metrics collection (advanced)

---

## Success Criteria Summary

| Metric | Baseline | Target | Week 7 Goal |
|--------|----------|--------|-------------|
| Concurrent simulations | 1-2 | 10+ | ✅ 10+ |
| Container startup time | N/A | <5s | ✅ <5s |
| API response time | TBD | <500ms | ✅ <500ms |
| Simulation time (no regression) | TBD | ±10% | ✅ ±10% |
| Memory usage per container | TBD | <4GB | ✅ <4GB |
| Success rate under load | TBD | >99% | ✅ >99% |
| Celery throughput | TBD | 60 tasks/min | ✅ 60+ |

---

## Testing Schedule

### Week 1-2: Baseline Measurement
- [ ] Measure current performance
- [ ] Document baseline metrics
- [ ] Identify bottlenecks

### Week 3-4: Containerization Testing
- [ ] Test container startup time
- [ ] Test container isolation
- [ ] Test Celery orchestration

### Week 5: Load Testing
- [ ] Run concurrent simulation tests
- [ ] Run sustained load tests
- [ ] Identify performance issues

### Week 6: Optimization
- [ ] Optimize slow areas
- [ ] Re-run performance tests
- [ ] Verify improvements

### Week 7: Final Validation
- [ ] Run all performance tests
- [ ] Document final metrics
- [ ] Prepare demo

---

## Reporting

### Weekly Performance Report Template
```markdown
# Week N Performance Report

## Metrics This Week
- Concurrent simulations: X (target: 10)
- Avg container startup: Xs (target: <5s)
- API response time: Xms (target: <500ms)

## Tests Run
- [x] Single user load test
- [ ] Concurrent user test (in progress)
- [ ] Sustained load test (planned next week)

## Issues Found
1. Container startup slow (8s) - investigating
2. Memory leak in Celery worker - created issue #45

## Next Week
- Optimize container startup
- Run 10 concurrent simulation test
- Fix memory leak
```

---

**Last Updated**: February 6, 2026  
**Test Manager**: [Your Name]  
**Next Review**: Weekly during team sync  
**Status**: Baseline measurements in progress