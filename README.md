# Relyr - Cross-Border Finance Infrastructure for Africa

[![Project Status](https://img.shields.io/badge/Status-Week%201-blue)](https://github.com/yusstech/relyr-app)
[![Progress](https://img.shields.io/badge/Progress-8.3%25-yellow)](./Docs/PROGRESS_TRACKER.md)
[![License](https://img.shields.io/badge/License-Proprietary-red)]()

> Hybrid payment platform bridging traditional finance and blockchain for instant, compliant cross-border payments.

**🎯 Current Phase:** Week 1 - Foundation & Planning (Day 1 Complete)

## 🚀 Project Overview

Relyr enables Nigerian freelancers and businesses to send, receive, and convert money seamlessly across borders using:
- **NGN Virtual Accounts** (via Anchor BaaS)
- **USDC on Polygon** (via Circle)
- **Instant conversions** between fiat and crypto
- **Bank-grade compliance** (KYC via Smile Identity)

**Target:** Pre-seed investor-ready MVP in 12 weeks

---

## 📁 Project Structure

```
relyr-app/
├── backend/              # NestJS API server
├── mobile/               # React Native (Expo) mobile app
├── docs/                 # Project documentation
├── infrastructure/       # Terraform, Docker configs
├── scripts/              # Utility scripts
├── .github/              # GitHub Actions workflows
└── README.md
```

---

## 🛠 Tech Stack

### Backend
- **Framework:** NestJS (TypeScript)
- **Database:** PostgreSQL 15
- **Cache/Queue:** Redis + BullMQ
- **Integrations:** Anchor, Circle, Paystack, Mono, Smile Identity

### Mobile
- **Framework:** React Native + Expo
- **State:** Zustand
- **Data Fetching:** TanStack Query (React Query)
- **UI:** React Native Paper

### Blockchain
- **Network:** Polygon (MATIC)
- **Stablecoin:** USDC (via Circle API)

### Infrastructure
- **Cloud:** AWS (ECS Fargate, RDS, ElastiCache)
- **CI/CD:** GitHub Actions
- **Monitoring:** Sentry, CloudWatch

---

## 📋 Prerequisites

Before you begin, ensure you have installed:

- **Node.js** 20.x LTS ([Download](https://nodejs.org/))
- **npm** 10.x or **yarn** 1.22.x
- **PostgreSQL** 15.x ([Download](https://www.postgresql.org/download/))
- **Redis** 7.x ([Download](https://redis.io/download))
- **Docker Desktop** (optional, for containerized services)
- **Git** ([Download](https://git-scm.com/))

### For Mobile Development:
- **iOS:** Xcode 15+ (macOS only)
- **Android:** Android Studio with SDK 33+
- **Expo CLI:** `npm install -g expo-cli`

---

## 🚀 Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/YOUR_USERNAME/relyr-app.git
cd relyr-app
```

### 2. Install Dependencies

```bash
# Install root dependencies (if using monorepo tools)
npm install

# Install backend dependencies
cd backend
npm install

# Install mobile dependencies
cd ../mobile
npm install
```

### 3. Setup Environment Variables

```bash
# Backend
cp backend/.env.example backend/.env
# Edit backend/.env with your API keys

# Mobile
cp mobile/.env.example mobile/.env
# Edit mobile/.env with your configuration
```

### 4. Setup Database

```bash
# Create PostgreSQL database
createdb relyr_dev

# Run migrations (from backend directory)
cd backend
npm run migration:run
```

### 5. Start Development Servers

```bash
# Terminal 1: Start PostgreSQL (if not running as service)
# psql is usually running as a service after installation

# Terminal 2: Start Redis
redis-server

# Terminal 3: Start backend API
cd backend
npm run start:dev

# Terminal 4: Start mobile app
cd mobile
npm start
# Then press 'i' for iOS simulator or 'a' for Android emulator
```

---

## 📖 Documentation

Comprehensive documentation is available in the [`/Docs`](/Docs) directory:

- **[PRD.md](/Docs/PRD.md)** - Product Requirements Document
- **[ROADMAP.md](/Docs/ROADMAP.md)** - 12-week development roadmap
- **[ARCHITECTURE.md](/Docs/ARCHITECTURE.md)** - Technical architecture & system design
- **[API_INTEGRATION_PLAN.md](/Docs/API_INTEGRATION_PLAN.md)** - Partner API integration guides

---

## 🔑 Partner API Keys

You'll need sandbox/test API keys from:

1. **Anchor** - Virtual NGN accounts
   - Sign up: https://getanchor.co/signup
   - Get keys: Dashboard → API Keys

2. **Circle** - USDC & blockchain wallets
   - Sign up: https://console.circle.com
   - Get keys: Apps → Create App → API Keys

3. **Paystack** - Card payments
   - Sign up: https://paystack.com/signup
   - Get keys: Settings → API Keys & Webhooks

4. **Mono** - Bank verification
   - Sign up: https://app.withmono.com/signup
   - Get keys: Settings → Keys

5. **Smile Identity** - KYC verification
   - Sign up: https://portal.usesmileidentity.com/signup
   - Get keys: Settings → API Keys

---

## 🧪 Testing

```bash
# Backend tests
cd backend
npm run test              # Unit tests
npm run test:e2e          # End-to-end tests
npm run test:cov          # Coverage report

# Mobile tests
cd mobile
npm run test
```

---

## 🚢 Deployment

### Backend (AWS ECS Fargate)

```bash
cd backend
docker build -t relyr-api .
# Push to ECR and deploy via GitHub Actions
```

### Mobile

```bash
# iOS
cd mobile
eas build --platform ios
eas submit --platform ios

# Android
eas build --platform android
eas submit --platform android
```

See [deployment documentation](/Docs/DEPLOYMENT.md) for detailed instructions.

---

## 📊 Development Workflow

### Branch Strategy
- `main` - Production-ready code
- `develop` - Development branch
- `feature/*` - Feature branches
- `fix/*` - Bug fix branches

### Commit Convention
We follow [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add user wallet creation endpoint
fix: resolve transaction status update bug
docs: update API integration guide
chore: upgrade dependencies
```

### Pull Request Process
1. Create feature branch from `develop`
2. Make your changes with tests
3. Ensure all tests pass
4. Create PR to `develop`
5. Get review approval
6. Merge to `develop`

---

## 🤝 Contributing

This is currently a solo founder project. Contributions are not open at this stage but will be in the future.

---

## 📝 License

Proprietary - All rights reserved

---

## 📧 Contact

- **Email:** hi@relyr.app
- **Website:** https://relyr.app (coming soon)

---

## 🗓 Current Status

```
[████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░] 8.3% Complete (Week 1 of 12)
```

**Week 1 of 12** - Foundation & Planning
- ✅ Day 1: Project setup complete
- 🔄 Day 2-5: Development environment & partner accounts

**Latest Updates:**
- ✅ Repository initialized with comprehensive documentation (3,800+ lines)
- ✅ GitHub CI/CD pipeline configured
- ✅ Monorepo structure established
- ⏳ Next: Install development tools and register partner accounts

See [ROADMAP.md](/Docs/ROADMAP.md) for detailed weekly milestones and [PROGRESS_TRACKER.md](/Docs/PROGRESS_TRACKER.md) for real-time progress.

---

## 💡 Quick Commands

```bash
# Backend
npm run start:dev         # Start in development mode
npm run start:prod        # Start in production mode
npm run build             # Build for production
npm run migration:generate # Generate new migration
npm run migration:run     # Run pending migrations

# Mobile
npm start                 # Start Expo development server
npm run ios               # Run on iOS simulator
npm run android           # Run on Android emulator
npm run web               # Run in web browser
```

---

Built with ❤️ for African freelancers and businesses
