# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

The Hedera Transaction Tool is a desktop Electron application with an optional backend for organizational workflows. It enables users to create, sign, and submit transactions to Hedera networks with support for both personal mode (single-user) and organizational mode (multi-user signing workflows).

**Architecture:** Monorepo with separate frontend (Electron + Vue 3) and backend (NestJS microservices)

**Prerequisites:**
- Node.js >= 22.12.0
- pnpm >= 9.13.1
- Docker Desktop (for backend development)
- Python setuptools >= 75.6.0 (frontend only)

## Common Development Commands

### Frontend Development

```bash
cd front-end

# Install dependencies and generate Prisma client
pnpm install
pnpm generate:database

# Start development mode (Electron with hot reload)
pnpm dev

# Type checking
pnpm typecheck          # Check both main and renderer processes
pnpm typecheck:node     # Main process only
pnpm typecheck:web      # Renderer process only

# Testing
pnpm test:main                # Run main process unit tests
pnpm test:main:coverage       # With coverage report

# Linting and formatting
pnpm lint
pnpm format

# Build for distribution
pnpm build              # Build renderer and compile TypeScript
pnpm build:mac          # Build .dmg for macOS
pnpm build:win          # Build .exe for Windows
pnpm build:linux        # Build AppImage for Linux

# Prisma operations (after schema changes)
cd front-end
npx prisma generate     # Regenerate Prisma client
npx prisma migrate dev  # Create and apply migrations
```

### Backend Development

```bash
cd back-end

# Install dependencies
pnpm install

# Start all services with Docker (recommended)
docker-compose up               # HTTP mode
docker-compose up --build       # Rebuild containers
docker-compose down             # Stop all services

# For HTTPS mode (requires mkcert):
mkdir -p cert
mkcert -install
mkcert -key-file ./cert/key.pem -cert-file ./cert/cert.pem localhost
docker-compose up

# Database operations
pnpm migration:generate <name>  # Generate migration from entity changes
pnpm migration:run              # Apply pending migrations
pnpm migration:revert           # Revert last migration

# Create admin user for local development
cd scripts
pnpm create-admin               # Interactive prompt for email/password

# Build services
pnpm build:all                  # Build all services
pnpm build:api                  # Build API service only
pnpm build:chain                # Build Chain service only
pnpm build:notifications        # Build Notifications service only

# Testing
pnpm test:all                   # Run all unit tests
pnpm test:cov:all               # Run all tests with coverage

# Per-service testing
cd apps/api
pnpm test:cov                   # Unit tests with coverage
pnpm test:e2e                   # E2E tests (starts testcontainers)

cd apps/chain
pnpm test:cov

cd apps/notifications
pnpm test:cov

# Type checking and linting
pnpm typecheck
pnpm lint
pnpm format
```

**E2E Testing Notes:**
- Ensure Docker is running before starting E2E tests
- Stop `docker-compose up` backend before running E2E tests
- If backend won't start after E2E tests, run: `docker compose up --force-recreate`
- To speed up Hedera Localnet startup: `pnpx hedera restart -d`

### Root-level Commands

```bash
# Format entire codebase
pnpm format
```

## High-Level Architecture

### Frontend Architecture (Electron + Vue 3)

**Three-Process Model:**
1. **Main Process** (`src/main/`): Node.js environment managing app lifecycle, database, and IPC handlers
2. **Preload Scripts** (`src/preload/`): Security bridge exposing safe APIs to renderer via `window.electronAPI`
3. **Renderer Process** (`src/renderer/`): Vue 3 SPA with Pinia state management

**Key Architectural Patterns:**

- **IPC Communication:** Renderer calls `window.electronAPI.local.[module].[action]()` → Main process handles database operations → Returns result to renderer
- **Database:** Local SQLite managed via Prisma ORM in main process
- **State Management:** Pinia stores (Composition API) with composables for reusable logic
- **Dual-Mode Architecture:**
  - **Personal Mode:** Local operations, direct database access, single-user workflows
  - **Organizational Mode:** Server-backed, multi-user signing, JWT authentication, WebSocket notifications

**Frontend Directory Structure:**
```
src/
├── main/                          # Main process (Electron, Node.js)
│   ├── db/                        # Prisma client initialization
│   ├── services/localUser/        # 20+ data operation services
│   ├── services/organization/     # Organization auth
│   └── modules/                   # IPC handlers, menu, logger, deep linking
├── preload/                       # IPC bridge between main and renderer
├── renderer/                      # Vue 3 SPA
│   ├── components/                # 183+ Vue components
│   │   ├── ui/                    # 35+ reusable UI components
│   │   ├── Transaction/           # Transaction-specific components
│   │   └── Organization/          # Organization mode components
│   ├── pages/                     # 75+ page components
│   ├── stores/                    # Pinia stores (user, network, notifications, etc.)
│   ├── composables/               # 25+ reusable composition functions
│   ├── caches/mirrorNode/         # TTL-based Mirror Node data caching
│   ├── router/                    # Vue Router with mode-based guards
│   └── utils/                     # Helper functions and SDK integration
└── shared/                        # Shared interfaces between main and renderer
```

**Core Pinia Stores:**
- `storeUser` - Authentication, organization selection, key pairs
- `storeNetwork` - Mirror Node configuration and network type
- `storeTransactionGroup` - Transaction grouping (atomic/non-atomic)
- `storeNotifications` - Unread notifications and filtering
- `storeWebsocketConnection` - WebSocket connection state
- `storeTheme` - Light/dark mode toggle

**Prisma Database Models (28+ migrations):**
- `User`, `KeyPair`, `Organization`, `OrganizationCredentials`
- `Transaction`, `TransactionDraft`, `TransactionGroup`, `GroupItem`
- `HederaAccount`, `HederaFile`, `ComplexKey`, `Contact`
- `Mnemonic`, `PublicKeyMapping`, `Claim`, `Migration`

### Backend Architecture (NestJS Microservices)

**Monorepo Structure:**
- **API Service** (`apps/api`): REST API (port 3001) - user auth, transaction management, signature collection
- **Chain Service** (`apps/chain`): Background job scheduler - transaction execution, reminders, cache refresh
- **Notifications Service** (`apps/notifications`): WebSocket gateway (port 3020) + email delivery
- **Common Library** (`libs/common`): Shared modules, entities, utilities, SDK integration

**Inter-Service Communication:**
- **NATS JetStream:** Event-driven pub/sub messaging with durable consumers
  - Streams: `NOTIFICATIONS_QUEUE`, `NOTIFICATIONS_FAN_OUT`
  - Patterns: Transaction status updates, user events, email delivery, WebSocket fan-out
- **Redis:** Caching (cache-manager), distributed locking (MurLock), Socket.IO adapter
- **PostgreSQL:** TypeORM entities with automatic migration system

**Key Backend Patterns:**

1. **Event-Driven Messaging:**
   - API publishes events to NATS (e.g., `TRANSACTION_STATUS_UPDATE`)
   - Notifications service consumes via durable consumers with explicit ack
   - Events fanned out to WebSocket clients via Redis-backed Socket.IO

2. **Distributed Transaction Execution:**
   - `@MurLock` decorator prevents concurrent execution across instances
   - Chain service uses Redis locks with 15s timeout, 5 retries

3. **Sophisticated Cron Scheduling:**
   - Time-based intervals scale from 10s (near-term) to daily (>1 week)
   - Handles transaction execution based on `valid_start` time
   - Cache refresh jobs for Mirror Node data with `lastCheckedAt` tracking

4. **Transaction Signature Model Factory:**
   - Abstract `TransactionBaseModel` per transaction type
   - Computes required signing accounts, receiver accounts, fee payer, new keys
   - Examples: `AccountCreateTransactionModel`, `FileUpdateTransactionModel`

5. **Mirror Node Caching:**
   - Cache entities: `CachedAccount`, `CachedNode`, `CachedAccountKey`, `CachedNodeAdminKey`
   - Refresh cron job updates stale cache data via Mirror Node API
   - Relationship tables: `TransactionCachedAccount`, `TransactionCachedNode`

**Backend Directory Structure:**
```
back-end/
├── apps/
│   ├── api/                       # REST API service
│   │   └── src/
│   │       ├── auth/              # JWT, OTP, local strategies
│   │       ├── transactions/      # Comments, Signers, Approvers, Groups
│   │       ├── users/             # User management
│   │       └── throttler/         # Rate limiting (IP, email, user)
│   ├── chain/                     # Background job scheduler
│   │   └── src/
│   │       ├── transaction-scheduler/   # Executes transactions at valid_start
│   │       ├── transaction-reminder/    # Reminds signers
│   │       └── cache-management/        # Mirror Node cache refresh
│   └── notifications/             # WebSocket + email delivery
│       └── src/
│           ├── websockets/        # Socket.IO gateway with Redis adapter
│           ├── email/             # Nodemailer integration
│           ├── fan-out/           # WebSocket client publishing
│           └── receiver/          # NATS consumer services
├── libs/common/                   # Shared library
│   └── src/
│       ├── database/              # TypeORM entities (User, Transaction, etc.)
│       ├── nats/                  # NATS JetStream services
│       ├── redis/                 # Redis cache and MurLock modules
│       ├── signature/             # Transaction signature computation
│       ├── execute/               # Transaction execution against Hedera
│       ├── modules/               # Scheduler, Logger, Auth, Health
│       └── utils/                 # SDK helpers, Mirror Node client
└── typeorm/                       # Migrations and data source config
```

**Docker Services (docker-compose.yaml):**
- `migration` - Runs TypeORM migrations on startup
- `api`, `chain`, `notifications` - Node.js services with hot reload
- `nats` - NATS JetStream message broker
- `database` - PostgreSQL with health checks
- `redis` - Redis with persistence
- `pgadmin` - Database admin UI (port 5050)

**NATS Event Patterns (defined in `libs/common/src/constants/eventPatterns.ts`):**
- `TRANSACTION_STATUS_UPDATE` - Transaction lifecycle events
- `TRANSACTION_UPDATE` - General transaction changes
- `TRANSACTION_REMIND_SIGNERS` / `TRANSACTION_REMIND_SIGNERS_MANUAL` - Signature reminders
- `USER_REGISTERED`, `USER_PASSWORD_RESET`, `USER_INVITE` - User lifecycle
- `EMAIL_NOTIFICATIONS` - Email delivery queue
- `FAN_OUT_NEW_NOTIFICATIONS`, `FAN_OUT_DELETE_NOTIFICATIONS` - WebSocket push events

**Transaction Status Flow:**
```
NEW → WAITING_FOR_SIGNATURES → WAITING_FOR_EXECUTION → EXECUTED/FAILED
```

**TypeORM Entity Relationships:**
- `User` → `UserKey[]`, `Transaction[]`, `Notification[]`
- `Transaction` → `TransactionComment[]`, `TransactionSigner[]`, `TransactionApprover[]`, `TransactionObserver[]`
- `TransactionGroup` → `GroupItem[]` (supports atomic and non-atomic grouping)
- `CachedAccount`/`CachedNode` → `TransactionCachedAccount`/`TransactionCachedNode` (many-to-many)

## Key Conventions

### Frontend
- **Naming:** PascalCase for components (`AppButton.vue`), camelCase for composables (`useUserStore.ts`)
- **IPC Calls:** Always via `window.electronAPI.local.[module].[action]()`
- **State:** Pinia Composition API stores with reactive refs
- **Routing:** Meta guards (`onlyOrganization`, `onlyAdmin`) for access control
- **Caching:** TTL-based Mirror Node caches with base class in `/renderer/caches/mirrorNode/`
- **Mode Detection:** Check `storeUser.selectedOrganization` (null = personal mode)

### Backend
- **Modules:** Feature modules per domain with explicit imports
- **Entities:** TypeORM with `@Entity()`, relationships via decorators
- **NATS:** Publish via `NatsPublisherService`, consume via `BaseNatsConsumerService` subclass
- **Locking:** Use `@MurLock` decorator for critical sections (transaction execution)
- **Cron:** Define in Chain service with `@Cron()` decorator from `@nestjs/schedule`
- **Testing:** Jest for unit tests, Testcontainers for E2E tests
- **Environment:** `.env` files per service (see `example.env` files)

### API Endpoints (apps/api)
- Base URL: `http://localhost:3001` (dev) or `https://localhost:3001` (with certs)
- Authentication: JWT tokens via Passport strategies (local, jwt, otp-jwt)
- Rate Limiting: IP-based, email-based, and user-based throttlers

### WebSocket Events (apps/notifications)
- Connection: Socket.IO client connects to `:3020`
- Events: `transactions:action`, `notifications:new`, `notifications:indicators:delete`
- Auth: JWT validation via WebSocket middleware
- Scaling: Redis adapter enables horizontal scaling

## Transaction Types

The system supports multiple Hedera transaction types via factory pattern:

**Account Operations:**
- `ACCOUNT_CREATE`, `ACCOUNT_UPDATE`, `ACCOUNT_DELETE`, `ACCOUNT_ALLOWANCE_APPROVE`, `ACCOUNT_ALLOWANCE_DELETE`

**Transfer Operations:**
- `TRANSFER`, `TRANSFER_V2` (supports NFTs, tokens, hbar, account allowances)

**File Operations:**
- `FILE_CREATE`, `FILE_UPDATE`, `FILE_APPEND`, `FILE_DELETE`

**System Operations:**
- `SYSTEM_DELETE`, `SYSTEM_UNDELETE`, `FREEZE`

**Node Operations:**
- `NODE_CREATE`, `NODE_UPDATE`, `NODE_DELETE`

Each type has a corresponding model in `libs/common/src/signature/models/` that computes required signatures.

## Troubleshooting

### Frontend Issues
- **Prisma errors:** Run `npx prisma generate` from `front-end/` directory
- **ENOENT errors:** Rebuild Electron: `pnpm rebuild electron`
- **Module not found:** Delete `node_modules` and reinstall: `pnpm install`

### Backend Issues
- **Docker startup:** Delete `pgdata/` folder and run `docker-compose up --build`
- **Port conflicts:** Ensure no other services on ports 3001, 3020, 5432, 5672, 6379, 4222
- **Migration errors:** Check connection in `.env` files, ensure database is running
- **E2E failures:** Restart backend with `docker compose up --force-recreate`

### Database Reset (Backend)
```bash
cd back-end
docker-compose down
rm -rf pgdata
docker-compose up
cd scripts
pnpm create-admin  # Re-create admin user
```

## Important Files

### Frontend
- [vite.config.ts](front-end/vite.config.ts) - Build configuration
- [prisma/schema.prisma](front-end/prisma/schema.prisma) - Database schema
- [src/main/index.ts](front-end/src/main/index.ts) - Electron main process entry
- [src/renderer/main.ts](front-end/src/renderer/main.ts) - Vue app initialization
- [src/preload/index.ts](front-end/src/preload/index.ts) - IPC bridge
- [src/renderer/stores/storeUser.ts](front-end/src/renderer/stores/storeUser.ts) - Core state

### Backend
- [docker-compose.yaml](back-end/docker-compose.yaml) - Local development setup
- [nest-cli.json](back-end/nest-cli.json) - NestJS monorepo config
- [typeorm/data-source.ts](back-end/typeorm/data-source.ts) - Database connection
- [libs/common/src/constants/eventPatterns.ts](back-end/libs/common/src/constants/eventPatterns.ts) - NATS event definitions
- [libs/common/src/database/entities/](back-end/libs/common/src/database/entities/) - All TypeORM entities
- [libs/common/src/signature/models/](back-end/libs/common/src/signature/models/) - Transaction signature models

## External Documentation

- User Guide: https://docs.hedera.com/hedera-transaction-tool-v2
- Hedera SDK: https://docs.hedera.com/hedera/sdks-and-apis/sdks
- Repository: https://github.com/hashgraph/hedera-transaction-tool
