# Quality Gates
## Requirements for Code Integration
## CHORAS Scalability Project - EngD 2026

---

## Purpose

Quality gates ensure that all code merged into our repository meets minimum standards for:
- **Functionality**: Code works as expected
- **Reliability**: Tests verify correctness
- **Maintainability**: Code is clean and documented
- **Performance**: No regression in speed/resource usage

---

## Before Committing (Local Development)

### Code Quality ✅
- [ ] Code follows team style guide (PEP 8 for Python, Airbnb for React)
- [ ] No commented-out code (use git history instead)
- [ ] No debug print statements (`console.log`, `print()` for debugging)
- [ ] No hardcoded credentials or API keys
- [ ] Variable names are descriptive
- [ ] Functions have docstrings (Python) or JSDoc comments (JavaScript)

### Testing ✅
- [ ] Local tests pass: `cd backend && pytest`
- [ ] New code has unit tests
- [ ] Test coverage for your changes ≥70%
- [ ] No skipped tests without documented reason

### Docker (if applicable) ✅
- [ ] Docker containers build successfully: `docker-compose build`
- [ ] Containers run without errors: `docker-compose up`
- [ ] No resource leaks (containers clean up properly)
- [ ] Dockerfile changes are documented

### Files ✅
- [ ] No unnecessary files committed (check `.gitignore`)
- [ ] `.env` files not committed (only `.env.example`)
- [ ] No large binary files (use Git LFS if needed)
- [ ] No IDE-specific files (`.vscode/`, `.idea/`, etc.)

### Verification Commands
```bash
# Before committing, run:
cd backend
pytest -v                    # Run tests
pytest --cov=app            # Check coverage
cd ..
docker-compose build        # Build containers (if changed)
git status                  # Review what you're committing
```

---

## Before Creating Pull Request

### Branch Management ✅
- [ ] Branch is up to date with `dev`:
  ```bash
  git checkout dev
  git pull origin dev
  git checkout your-feature-branch
  git merge dev
  ```
- [ ] No merge conflicts
- [ ] Branch name follows convention: `feature/`, `bugfix/`, `test/`
- [ ] Commits are logical and atomic (not "fix", "update", "asdf")

### Code Review Ready ✅
- [ ] All new code has tests
- [ ] Tests pass locally: `pytest -v`
- [ ] Test coverage hasn't decreased: `pytest --cov=app`
- [ ] Code is self-explanatory or has comments
- [ ] No "TODO" or "FIXME" without GitHub issue

### Documentation ✅
- [ ] README updated (if adding new features)
- [ ] API documentation updated (if changing endpoints)
- [ ] Setup instructions updated (if changing environment)
- [ ] `docs engd 2026/` updated (if relevant to project goals)
- [ ] Docstrings added for new functions/classes

### Changelog (Optional but Recommended) ✅
- [ ] CHANGELOG.md updated with:
  - What changed
  - Why it changed
  - Impact on existing functionality

### PR Description ✅
Your PR should include:
- **What**: Brief description of changes
- **Why**: Reason for the change (reference issue #)
- **How**: Technical approach (if complex)
- **Testing**: How you tested it
- **Screenshots**: For UI changes

**Example PR Description:**
```markdown
## What
Containerize DE simulation method (#42)

## Why
Part of scalability project - each simulation method needs its own container
to avoid Python dependency conflicts.

## How
- Created Dockerfile for DE method
- Added Celery task to spin up DE container
- Container accepts JSON input, outputs JSON result

## Testing
- Unit tests for Dockerfile build
- Integration test for Celery → container workflow
- Verified with manual test: `docker run choras-de-simulation:latest`

## Screenshots
N/A (backend only)
```

### Verification Commands
```bash
# Before creating PR:
git diff dev...your-branch  # Review all changes
pytest --cov=app           # Verify coverage
docker-compose build       # Verify Docker builds
docker-compose up          # Verify system runs
```

---

## Before Merging PR (Reviewer's Checklist)

### Automated Checks ✅
- [ ] **CI/CD pipeline passes** (GitHub Actions green ✓)
  - All tests pass
  - Docker builds succeed
  - Linting passes (if configured)
  - Coverage meets threshold

### Code Review ✅
- [ ] **At least 1 reviewer approval** required
- [ ] Code is understandable and maintainable
- [ ] No unnecessary complexity
- [ ] Security concerns addressed (no hardcoded secrets)
- [ ] Performance considerations reviewed

### Testing ✅
- [ ] Tests are meaningful (not just placeholders)
- [ ] Edge cases are tested
- [ ] Error handling is tested
- [ ] Integration tests added for new features

### Merge Safety ✅
- [ ] No merge conflicts with `dev`
- [ ] Test coverage hasn't decreased from baseline
- [ ] Performance benchmarks reviewed (if changed)
- [ ] No breaking changes (or documented and communicated)

### Container-Specific (if applicable) ✅
- [ ] Container tests pass
- [ ] Container builds successfully in CI
- [ ] Container resource limits defined (CPU, memory)
- [ ] Container cleanup verified (no orphaned containers)

### Review Process
```bash
# Reviewer should:
git checkout pr-branch
docker-compose build
docker-compose up
pytest -v
# Manually test the feature
# Review code on GitHub
# Approve or request changes
```

---

## Before Release/Demo (Client Meeting)

### System Validation ✅
- [ ] All integration tests pass: `pytest tests/integration/`
- [ ] All system tests pass (end-to-end workflows)
- [ ] Performance tests meet targets:
  - Container startup time <5s
  - API response time <500ms
  - Can handle 10 concurrent simulations (target)
- [ ] Load testing completed (if applicable)

### Quality Metrics ✅
- [ ] Test coverage ≥70% overall
- [ ] No P0 (critical) bugs open
- [ ] P1 (high) bugs documented and triaged
- [ ] CI/CD pipeline passing on `dev` branch

### Docker & Deployment ✅
- [ ] All containers build and run successfully:
  ```bash
  docker-compose build
  docker-compose up -d
  docker-compose ps  # All should be "Up"
  ```
- [ ] No container errors in logs: `docker-compose logs`
- [ ] Resource usage is acceptable: `docker stats`
- [ ] Cleanup works properly: `docker-compose down -v`

### Documentation ✅
- [ ] Documentation is complete and accurate
- [ ] Setup instructions are up to date
- [ ] Known issues are documented
- [ ] Demo script prepared (for client meeting)

### Demo Preparation ✅
- [ ] Sample data prepared (test geometries)
- [ ] Demo script tested
- [ ] Screenshots/videos captured (if needed)
- [ ] Backup plan if demo fails (screenshots ready)
- [ ] Team knows who's presenting what

### Pre-Demo Checklist
```bash
# Day before demo:
git checkout dev
git pull origin dev
docker-compose down -v     # Clean slate
docker-compose build       # Fresh build
docker-compose up -d       # Start services
# Test all demo scenarios
# Take screenshots as backup
```

---

## Enforcement

### Automated (CI/CD)
GitHub Actions will automatically check:
- ✅ Tests pass
- ✅ Docker builds
- ✅ Code coverage (if threshold set)
- ✅ Linting (if configured)

**If CI fails, PR cannot be merged.**

### Manual (Code Review)
At least 1 team member must:
- ✅ Review the code
- ✅ Approve the PR
- ✅ Verify quality gates are met

### Exceptions
In rare cases, quality gates may be bypassed:
- **Critical hotfix**: Production is broken
- **Demo pressure**: Client meeting in <24h

**Process for exceptions:**
1. Get approval from Project Manager
2. Create GitHub issue to fix properly later
3. Document exception in PR description
4. Fix within 1 week

---

## Quality Metrics Dashboard

### Track Weekly
| Metric | Current | Target | Status |
|--------|---------|--------|--------|
| Test Coverage | X% | 70% | 🟡 |
| Open P0 Bugs | N | 0 | ✅ |
| Open P1 Bugs | N | <5 | ✅ |
| CI/CD Pass Rate | X% | 95% | 🟢 |
| Avg PR Review Time | Xh | <24h | 🟢 |
| Container Build Time | Xs | <2min | ✅ |

Legend: ✅ Good | 🟢 On Track | 🟡 Needs Attention | 🔴 Critical

---

## Tools & Resources

### Testing
- **pytest**: `pytest -v --cov=app`
- **Coverage report**: `pytest --cov-report=html`
- **Frontend tests**: `npm test -- --coverage`

### Code Quality
- **Linting (Python)**: `flake8 .`
- **Formatting (Python)**: `black .`
- **Linting (JavaScript)**: `npm run lint`

### Docker
- **Build**: `docker-compose build`
- **Test**: `docker-compose -f docker-compose.test.yml up`
- **Monitor**: `docker stats`

### CI/CD
- **GitHub Actions**: https://github.com/Saptarshi666/CHORAS/actions
- **View logs**: Click on workflow run → Click on job

---

## Responsibilities

### Developer
- Ensure code meets quality gates before PR
- Write tests for new code
- Respond to review feedback

### Test Manager (Your Role!)
- Enforce quality gates
- Review test coverage
- Triage bug priority
- Report quality metrics weekly

### Code Reviewer
- Verify quality gates are met
- Provide constructive feedback
- Approve or request changes

### Project Manager
- Approve exceptions to quality gates
- Track quality metrics
- Ensure team follows process

---

## Getting Help

**Questions about quality gates?**
- Ask in team chat
- Create GitHub discussion
- Contact Test Manager: [Your Name]

**Quality gates failing?**
1. Read the CI/CD logs
2. Run tests locally to reproduce
3. Fix the issue
4. Ask team for help if stuck

**Need to bypass quality gates?**
1. Explain situation to Project Manager
2. Get written approval
3. Create issue to fix properly
4. Document in PR

---

**Last Updated**: February 6, 2026  
**Maintained by**: Test Manager & Quality Manager  
**Review Frequency**: Weekly during team sync  
**Next Review**: After first sprint (Week 2)