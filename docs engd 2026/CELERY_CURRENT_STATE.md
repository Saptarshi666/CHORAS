# Current Celery Setup in CHORAS
   
   ## Broker
   - Type: Redis 
   - Container: redis
   - Configuration: backend/config.py
   
   ## Workers
   - Number of workers: 1
   - Tasks defined: [list task names found]
   - Location: [path to tasks.py or similar]
   
   ## Current Task Flow
   1. User submits simulation via frontend
   2. Backend API creates Celery task
   3. Celery worker picks up task
   4. [Where does simulation actually run?] ans: simulation runs on the backend directly
   5. Results stored and returned
   
