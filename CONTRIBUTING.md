# Contributing to Relyr

Thank you for your interest in contributing to Relyr!

## Development Setup

1. **Prerequisites**
   - Node.js 20.x LTS
   - PostgreSQL 15.x
   - Redis 7.x
   - Git

2. **Clone and Install**
   ```bash
   git clone https://github.com/YOUR_USERNAME/relyr-app.git
   cd relyr-app
   npm run install:all
   ```

3. **Setup Environment**
   ```bash
   cp backend/.env.example backend/.env
   cp mobile/.env.example mobile/.env
   # Edit .env files with your API keys
   ```

4. **Create Database**
   ```bash
   createdb relyr_dev
   cd backend
   npm run migration:run
   ```

## Development Workflow

### Branch Naming

- `feature/` - New features (e.g., `feature/wallet-creation`)
- `fix/` - Bug fixes (e.g., `fix/transaction-status-bug`)
- `docs/` - Documentation (e.g., `docs/api-guide`)
- `chore/` - Maintenance (e.g., `chore/upgrade-deps`)

### Commit Messages

We follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation
- `style`: Code style (formatting, missing semicolons, etc.)
- `refactor`: Code refactoring
- `test`: Adding tests
- `chore`: Maintenance

**Examples:**
```
feat(wallet): add USDC wallet creation endpoint

Implemented Circle API integration to create USDC wallets
on Polygon network for new users.

Closes #123
```

```
fix(auth): resolve JWT token expiration bug

Fixed issue where refresh tokens were not properly
invalidating expired access tokens.
```

### Pull Request Process

1. **Create Feature Branch**
   ```bash
   git checkout -b feature/your-feature-name
   ```

2. **Make Changes**
   - Write code
   - Add tests
   - Update documentation

3. **Test Locally**
   ```bash
   npm run lint
   npm run format:check
   cd backend && npm run test
   cd ../mobile && npm run test
   ```

4. **Commit Changes**
   ```bash
   git add .
   git commit -m "feat(scope): your commit message"
   ```

5. **Push to GitHub**
   ```bash
   git push origin feature/your-feature-name
   ```

6. **Create Pull Request**
   - Go to GitHub
   - Create PR from your branch to `develop`
   - Fill in PR template
   - Request review

7. **Address Review Comments**
   - Make requested changes
   - Push updates to same branch
   - PR will update automatically

8. **Merge**
   - Once approved, squash and merge to `develop`

## Code Standards

### TypeScript

- Use TypeScript for all new code
- Enable strict mode
- Define interfaces for all data structures
- Avoid `any` type (use `unknown` if necessary)

### NestJS (Backend)

- Follow NestJS best practices
- Use dependency injection
- Create DTOs for all API inputs
- Use class-validator for validation
- Write unit tests for services
- Write E2E tests for controllers

### React Native (Mobile)

- Use functional components with hooks
- Use TypeScript interfaces for props
- Follow React Native best practices
- Use Zustand for global state
- Use React Query for server state
- Write tests for critical components

### Formatting

- Run `npm run format` before committing
- ESLint and Prettier are configured
- CI will check formatting

## Testing

### Backend Tests

```bash
cd backend

# Unit tests
npm run test

# E2E tests
npm run test:e2e

# Coverage
npm run test:cov
```

### Mobile Tests

```bash
cd mobile

# Unit tests
npm run test

# Component tests
npm run test:watch
```

## Documentation

- Update README.md if adding new setup steps
- Add JSDoc comments for public APIs
- Update Swagger/OpenAPI specs for new endpoints
- Document breaking changes in PR description

## Questions?

Email: hi@relyr.app

---

Thank you for contributing! 🚀
