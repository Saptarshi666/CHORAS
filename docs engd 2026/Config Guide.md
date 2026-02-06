# Configuration Management Guide
   
   ## Environment Setup
   
   ### Required Software
   - Docker Desktop (latest stable)
   - Git (2.x or higher)
   - Python 3.10+ (for local backend development)
   - Node.js 18+ (for local frontend development)
   
   ### First Time Setup
   
   1. **Clone the repository**
```bash
      git clone [YOUR-FORK-URL]
      cd CHORAS
```
   
   2. **Initialize submodules**
```bash
      git submodule update --init --recursive
```
   
   3. **Copy environment templates**
```bash
      cp .env.api.example .env.api
      cp .env.db.example .env.db
```
   
   4. **Edit environment files**
      - Open `.env.api` and set your local values
      - Open `.env.db` and set database credentials
   
   ## Git Workflow
   
   ### Creating a New Feature
   
   1. **Create a feature branch from dev**
```bash
      git checkout dev
      git pull origin dev
      git checkout -b feature/your-feature-name
```
   
   2. **Make your changes**
      - Edit files
      - Test locally
      - Commit frequently
   
   3. **Commit with good messages**
```bash
      git add .
      git commit -m "feat: add scalability improvement for XYZ"
```
   
   4. **Push to GitHub**
```bash
      git push origin feature/your-feature-name
```
   
   5. **Create Pull Request**
      - Go to GitHub
      - Create PR from your feature branch to `dev`
      - Request review from team
   
   ### Commit Message Format
   - `feat:` New feature
   - `fix:` Bug fix
   - `test:` Adding tests
   - `docs:` Documentation
   - `refactor:` Code refactoring
   - `perf:` Performance improvement
   
   ## Environment Variables
   
   ### `.env.api`
   FLASK_ENV=development
   SECRET_KEY=[generate-random-string]
   DATABASE_URL=postgresql://user:pass@db:5432/choras
   
   ### `.env.db`
   POSTGRES_USER=choras_user
   POSTGRES_PASSWORD=[secure-password]
   POSTGRES_DB=choras

   **⚠️ NEVER commit actual .env files to git!**
   
   ## Keeping Your Fork Updated (from perspective of changes to actual repo)
```bash
   # Add upstream remote (only do once)
   git remote add upstream https://github.com/choras-org/CHORAS.git
   
   # Fetch upstream changes
   git fetch upstream
   
   # Merge upstream main into your main
   git checkout main
   git merge upstream/main
   git push origin main
   
   # Update dev branch
   git checkout dev
   git merge main
   git push origin dev
```

