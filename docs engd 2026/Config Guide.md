# Configuration Management Guide
## CHORAS Scalability Project - EngD 2026

---

## Repository Structure

```
CHORAS/
├── .github/                      # GitHub configuration
│   ├── ISSUE_TEMPLATE/          # Issue templates (bug reports, features)
│   └── workflows/               # CI/CD pipelines (GitHub Actions)
├── backend/                      # Backend submodule (Flask API)
├── frontend-v2/                  # Frontend submodule (React)
├── docs engd 2026/              # Team documentation (YOUR docs!)
│   ├── CELERY_CURRENT_STATE.md
│   ├── Config Guide.md          # This file
│   ├── Configuration Management Plan.md
│   ├── Quality Gates.md
│   ├── Scalability Test Plan.md
│   ├── Test Coverage.md
│   ├── Testing Guide.md
│   └── Testing Strategy.md
├── example_geometries/           # Sample room geometries
├── .env.api                      # Backend API environment vars (NOT in git!)
├── .env.db                       # Database environment vars (NOT in git!)
├── .gitignore                    # Files to ignore in git
├── .gitmodules                   # Submodule configuration
├── docker-compose.yml            # Main Docker orchestration
├── docker-compose.test.yml       # Testing environment
├── LICENSE                       # MIT License
├── README.md                     # Repository overview
└── setup_instructions.md         # Docker setup guide
```

---

## Environment Setup

### Required Software
- **Docker Desktop** (latest stable) - [Download](https://www.docker.com/products/docker-desktop/)
- **Git** (2.x or higher) - [Download](https://git-scm.com/)
- **Python 3.10+** (for local backend development) - Optional
- **Node.js 18+** (for local frontend development) - Optional

### Verify Installation
```bash
# Check versions
docker --version          # Should show: Docker version 24.x.x or higher
docker-compose --version  # Should show: Docker Compose version 2.x.x
git --version            # Should show: git version 2.x.x
```

---

## First Time Setup

### Step 1: Clone the Repository
```bash
# Clone your team's fork
git clone https://github.com/Saptarshi666/CHORAS.git
cd CHORAS

# Verify you're on the dev branch
git branch
# Should show: * dev
```

### Step 2: Initialize Submodules
The `backend/` and `frontend-v2/` directories are Git submodules (separate repositories).

```bash
# Initialize and clone submodules
git submodule update --init --recursive

# Verify submodules are populated
ls backend/
ls frontend-v2/

# Should see files in both directories (not empty)
```

### Step 3: Set Up Environment Files

**IMPORTANT**: The `.env.api` and `.env.db` files already exist in the repository but should NOT be committed with real credentials!

```bash
# Option 1: If .env files exist, verify they don't have production credentials
cat .env.api
cat .env.db

# Option 2: If you need to create them from scratch
# (They might already be there, so check first!)
```

**Edit `.env.api`** (if needed):
```bash
# Open with your editor
nano .env.api
# or
code .env.api
```

Example content:
```
FLASK_ENV=development
SECRET_KEY=your-local-dev-secret-key
DATABASE_URL=postgresql://choras_user:choras_pass@db:5432/choras
CELERY_BROKER_URL=redis://redis:6379/0
```

**Edit `.env.db`** (if needed):
```bash
nano .env.db
```

Example content:
```
POSTGRES_USER=choras_user
POSTGRES_PASSWORD=choras_pass
POSTGRES_DB=choras
```

⚠️ **CRITICAL**: Never commit `.env` files with real passwords to Git!

### Step 4: Start CHORAS
```bash
# Start all services
docker-compose up

# Or run in background
docker-compose up -d

# Check if everything is running
docker-compose ps
```

Expected services:
- Database (PostgreSQL)
- Backend API (Flask)
- Frontend (React)
- Celery Worker
- Redis (message broker)

### Step 5: Access CHORAS
- **Frontend**: http://localhost:3000 (or check docker-compose.yml for actual port)
- **Backend API**: http://localhost:5000 (or check docker-compose.yml)

---

## Git Workflow

### Branching Strategy

```
main    ───────────────────────────  Stable, production-ready
         ↖                      ↗
dev      ──────●──────●─────●──────  Integration branch (WORK HERE!)
               ↑      ↑     ↑
feature/x ─────┘      │     │        Feature branches
feature/y ────────────┘     │
bugfix/z ───────────────────┘        Bugfix branches
```

- **main**: Stable code only (matches upstream CHORAS)
- **dev**: Your team's integration branch (WORK HERE!)
- **feature/\***: Individual feature branches
- **bugfix/\***: Bug fix branches

### Creating a New Feature

#### 1. Create Feature Branch
```bash
# Make sure you're on dev and it's up to date
git checkout dev
git pull origin dev

# Create your feature branch
git checkout -b feature/your-feature-name

# Example: Working on DE containerization
git checkout -b feature/de-container

# Example: Working on testing
git checkout -b feature/add-integration-tests
```

#### 2. Make Your Changes
```bash
# Edit files
code backend/some_file.py

# Test locally
docker-compose up
# Or run tests
cd backend && pytest

# Check what changed
git status
git diff
```

#### 3. Commit Your Changes
```bash
# Stage files
git add .
# Or stage specific files
git add backend/some_file.py

# Commit with descriptive message (see format below)
git commit -m "feat: add DE simulation container"

# Make multiple commits if needed
git commit -m "test: add tests for DE container"
git commit -m "docs: update containerization guide"
```

#### 4. Push to GitHub
```bash
# First time pushing this branch
git push -u origin feature/your-feature-name

# Subsequent pushes
git push
```

#### 5. Create Pull Request
1. Go to https://github.com/Saptarshi666/CHORAS
2. You'll see: "Compare & pull request" button
3. Click it
4. **Base**: `dev` ← **Compare**: `feature/your-feature-name`
5. Fill in PR description
6. Request review from teammate
7. Wait for CI/CD to pass (GitHub Actions)
8. After approval, merge to `dev`

---

## Commit Message Format

Follow **Conventional Commits** format:

```
<type>: <description>

[optional body]

[optional footer]
```

### Types:
- **feat**: New feature (e.g., `feat: add DG simulation container`)
- **fix**: Bug fix (e.g., `fix: resolve container startup error`)
- **test**: Adding tests (e.g., `test: add Celery task tests`)
- **docs**: Documentation (e.g., `docs: update setup instructions`)
- **refactor**: Code refactoring (e.g., `refactor: simplify task queue logic`)
- **perf**: Performance improvement (e.g., `perf: optimize container startup time`)
- **chore**: Maintenance (e.g., `chore: update dependencies`)
- **ci**: CI/CD changes (e.g., `ci: add Docker build to GitHub Actions`)

### Examples:
```bash
# Good commits
git commit -m "feat: containerize DE simulation method"
git commit -m "fix: correct database connection string in .env.api"
git commit -m "test: add integration tests for Celery tasks"
git commit -m "docs: add containerization strategy to docs"

# Bad commits (avoid these)
git commit -m "update"
git commit -m "fix stuff"
git commit -m "asdfasdf"
```

---

## Environment Variables Reference

### `.env.api` (Backend Configuration)
```bash
FLASK_ENV=development              # or production
SECRET_KEY=your-secret-key         # Generate with: python -c "import secrets; print(secrets.token_hex(32))"
DATABASE_URL=postgresql://user:pass@db:5432/choras
CELERY_BROKER_URL=redis://redis:6379/0
CELERY_RESULT_BACKEND=redis://redis:6379/0
```

### `.env.db` (Database Configuration)
```bash
POSTGRES_USER=choras_user
POSTGRES_PASSWORD=your-secure-password
POSTGRES_DB=choras
POSTGRES_PORT=5432
```

### Security Rules:
- ⛔ **NEVER** commit real passwords to Git
- ⛔ **NEVER** commit API keys or secrets
- ✅ **DO** use different passwords for dev vs production
- ✅ **DO** document what variables are needed (without values)
- ✅ **DO** add `.env` to `.gitignore` (already done)

---

## Working with Submodules

### Understanding Submodules
The `backend/` and `frontend-v2/` are separate Git repositories:
- **backend**: https://github.com/choras-org/backend
- **frontend-v2**: https://github.com/choras-org/frontend-v2

### Common Submodule Commands

```bash
# Initialize submodules (first time)
git submodule update --init --recursive

# Update submodules to latest
git submodule update --remote

# Check submodule status
git submodule status

# If submodules are empty or broken
git submodule deinit -f .
git submodule update --init --recursive
```

### Updating Submodules
If upstream CHORAS updates their backend or frontend:
```bash
# Pull latest from upstream submodules
cd backend
git pull origin main
cd ..

# Commit the submodule update
git add backend
git commit -m "chore: update backend submodule to latest"
git push
```

---

## Docker Commands Reference

### Starting/Stopping Services
```bash
# Start all services (foreground)
docker-compose up

# Start all services (background)
docker-compose up -d

# Stop all services
docker-compose down

# Stop and remove volumes (DELETES DATA!)
docker-compose down -v
```

### Viewing Logs
```bash
# All services
docker-compose logs

# Follow logs (live)
docker-compose logs -f

# Specific service
docker-compose logs backend
docker-compose logs frontend-v2

# Last 50 lines
docker-compose logs --tail=50
```

### Rebuilding Containers
```bash
# Rebuild all containers
docker-compose build

# Rebuild specific service
docker-compose build backend

# Rebuild and start
docker-compose up --build
```

### Checking Service Status
```bash
# List running containers
docker-compose ps

# Enter a running container (for debugging)
docker-compose exec backend bash
docker-compose exec frontend-v2 sh
```

### Testing Environment
```bash
# Run tests in isolated environment
docker-compose -f docker-compose.test.yml up --abort-on-container-exit

# Clean up test environment
docker-compose -f docker-compose.test.yml down -v
```

---

## File Management Best Practices

### What to Commit ✅
- Source code (`.py`, `.js`, `.tsx`, etc.)
- Configuration templates (`.env.example`)
- Documentation (`.md` files in `docs engd 2026/`)
- Docker configs (`docker-compose.yml`)
- CI/CD configs (`.github/workflows/*.yml`)

### What NOT to Commit ⛔
- `.env.api` and `.env.db` with real credentials
- `node_modules/` (frontend dependencies)
- `__pycache__/` (Python cache)
- `.pytest_cache/` (test cache)
- Database files
- IDE-specific files (`.vscode/`, `.idea/`)

**Already handled by `.gitignore`** - but always double-check before committing!

---

## Troubleshooting

### Submodules Are Empty
```bash
git submodule update --init --recursive
```

### Docker Services Won't Start
```bash
# Check logs
docker-compose logs

# Rebuild containers
docker-compose down
docker-compose build
docker-compose up
```


### Database Connection Errors
Check your `.env.db` and `.env.api` match:
```bash
# .env.db has: POSTGRES_USER=choras_user
# .env.api should have: DATABASE_URL=postgresql://choras_user:...
```

---

## Quick Reference

### Daily Workflow
```bash
# 1. Start working
git checkout dev
git pull origin dev
git checkout -b feature/my-feature

# 2. Make changes and test
# ... edit files ...
docker-compose up

# 3. Commit
git add .
git commit -m "feat: description"

# 4. Push and create PR
git push -u origin feature/my-feature
# Go to GitHub and create PR to dev branch
```

### Before Meetings
```bash
# Make sure everything is up to date
git checkout dev
git pull origin dev

# Make sure CHORAS runs
docker-compose up

# Check CI/CD is passing
# Visit: https://github.com/Saptarshi666/CHORAS/actions
```

---

## Configuration Management Contacts

**Config Manager**: Saptarshi Mondal
**Questions**: Ask in team channel or create GitHub issue

**Documentation Location**: `docs engd 2026/`
**Issue Templates**: `.github/ISSUE_TEMPLATE/`
**CI/CD Pipelines**: `.github/workflows/`

---

**Last Updated**: February 9, 2026
**Team**: EngD Software Technology 2026
**Project**: CHORAS Scalability Improvements