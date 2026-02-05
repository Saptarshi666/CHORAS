# Quality Gates
## Requirements for Code Integration

### Before Committing
- [ ] Code follows team style guide
- [ ] No commented-out code
- [ ] No debug print statements
- [ ] Local tests pass
- [ ] Docker containers build successfully (if modified)

### Before Creating Pull Request
- [ ] Branch is up to date with dev
- [ ] All new code has tests
- [ ] Tests pass locally
- [ ] Documentation updated
- [ ] CHANGELOG updated
- [ ] Container tests pass (if applicable)

### Before Merging PR
- [ ] Code review approved (1+ reviewer)
- [ ] CI/CD pipeline passes
- [ ] No merge conflicts
- [ ] Test coverage hasn't decreased
- [ ] Performance benchmarks reviewed (if changed)

### Before Release/Demo
- [ ] All integration tests pass
- [ ] Performance tests meet targets
- [ ] No critical bugs open
- [ ] Documentation is complete
- [ ] All containers build and run successfully