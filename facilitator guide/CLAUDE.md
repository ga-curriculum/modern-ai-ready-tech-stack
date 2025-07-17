# CLAUDE.md - Project Development Guidelines

## Table of Contents

1. [Core Principles & Domain-Driven Design](#core-principles--domain-driven-design)
2. [Service Architecture & Patterns](#service-architecture--patterns)
3. [Type Safety Rules](#type-safety-rules)
4. [Testing Strategy](#testing-strategy)
5. [Development Workflow](#development-workflow)
6. [Media Asset Seeding](#media-asset-seeding)
7. [Common Commands](#common-commands)
8. [Technology Stack](#technology-stack)
9. [Architecture Patterns](#architecture-patterns)
10. [Platform Thinking](#platform-thinking)
11. [Media Optimization Strategy](#media-optimization-strategy)
12. [Important Reminders](#important-reminders)
13. [Documentation](#documentation)
14. [Development Environment & OAuth Configuration](#development-environment--oauth-configuration)

---

## Core Principles & Domain-Driven Design

### Core Development Principles

- Always use TypeScript with strict type checking
- Use factory functions with const/object pattern (NO classes)
- Use descriptive variable names and avoid abbreviations
- Follow RO-RO pattern (Receive Object, Return Object) for function parameters
- **NEVER use `any` or `unknown`** except when parsing external data with safeParse

### Domain-Driven Design (DDD) & Service Architecture

#### Standard Service Structure

Every new service MUST follow this structure:

##### Example 1: Email Service

```
packages/email-service/
├── src/
│   ├── services/
│   │   ├── email-sender.ts       # Factory: createEmailService(deps)
│   │   └── template-engine.ts    # Factory: createTemplateEngine(deps)
│   ├── types/
│   │   ├── email.ts             # Domain types and interfaces
│   │   └── template.ts
│   ├── dto/                     # Local DTOs (not shared)
│   │   ├── email-queue.dto.ts
│   │   └── retry.dto.ts
│   ├── utils/
│   │   ├── validators.ts
│   │   └── formatters.ts
│   └── index.ts                # Export ONLY factory functions
├── package.json
└── README.md
```

##### Example 2: Analytics Service

```
packages/analytics-service/
├── src/
│   ├── services/
│   │   ├── analytics-tracker.ts  # createAnalyticsTracker(deps)
│   │   ├── event-processor.ts    # createEventProcessor(deps)
│   │   └── report-generator.ts   # createReportGenerator(deps)
│   ├── types/
│   │   ├── events.ts
│   │   ├── metrics.ts
│   │   └── reports.ts
│   ├── dto/                     # Internal DTOs only
│   │   └── batch-events.dto.ts
│   ├── utils/
│   │   └── aggregators.ts
│   └── index.ts
└── tests/
```

---

## Service Architecture & Patterns

### Service Implementation Pattern (NO CLASSES)

```typescript
// ✅ CORRECT: Factory function returning service object
export const createEmailService = (deps: {
  resendApiKey: string;
  supabase?: SupabaseClient;  // Optional, use createClient() if not provided
}) => {
  const supabase = deps.supabase || createClient();

  const sendEmail = async (request: EmailRequest): Promise<EmailResponse> => {
    // Implementation using deps.resendApiKey and supabase
  };

  const sendBulkEmails = async (requests: EmailRequest[]): Promise<EmailResponse[]> => {
    // Implementation
  };

  const getEmailStatus = async (emailId: string): Promise<EmailStatus> => {
    // Implementation
  };

  return {
    sendEmail,
    sendBulkEmails,
    getEmailStatus,
  };
};

// ❌ WRONG: Never use classes
export class EmailService { ... } // NO!

// ❌ WRONG: Don't use function declaration
export function createEmailService() { ... } // Use const instead!
```

### Service Pattern Reference

All services follow the factory pattern. See the Email Service example above. For more examples
(Video Processing, Cache Service), see: `packages/media-rendering/docs/service-patterns.md`

### Shared Utilities Pattern

```typescript
// ✅ CORRECT: Package utils structure
packages/service-name/
├── src/
│   ├── utils/
│   │   ├── service-utils.ts    # Shared service initialization
│   │   └── validation-utils.ts # Environment validation
│   └── index.ts               # Export utilities for reuse
```

**Rules:**

- **Extract functions used in 2+ files** into src/utils/
- **Export utilities from index.ts** for other packages
- **Validate environment variables** with descriptive errors
- **Use proper union types** instead of any assertions

### When to Use Shared vs Local DTOs

**Use @repo/shared-dtos when:**

- DTO crosses service boundaries (API → Service → Worker)
- Used by 2+ packages
- Part of public API contract
- Request/Response pairs for network communication

**Use local dto/ folder when:**

- Internal to one service only
- Implementation detail
- Frequently changing without affecting others
- Service-specific queue messages

### Service Standardization & Discovery

#### When Working on Any Service

You MUST check for standardization and present options if patterns don't match our standards.

##### Example Standardization Checks

**Pattern 1: Class-Based Service Found** "I found a class-based service. Here are the options:

1. **Refactor to Factory** (Recommended)
   - Convert: `class XService` → `const createXService = (deps) => { ... }`
   - Benefits: Consistency, easier testing, clear dependencies
   - Example migration provided

2. **Wrapper Factory**
   - Keep class, export factory: `export const createXService = (deps) => new XService(deps)`
   - Quick fix, allows gradual migration

3. **Document Exception**
   - Add CLAUDE.md in package explaining why class is needed
   - Must have strong justification

Which approach should I take?"

**Pattern 2: Untyped Client Usage** "This service uses untyped Supabase client. Options:

1. **Use Package createClient** (Recommended)
   - Import from package root: `import { createClient } from "@repo/supabase"`
   - Full type safety guaranteed

2. **Service-Specific Typed Client**
   - Wrap with additional context
   - Export for other services

Which approach fits your needs?"

**Pattern 3: Mixed DTOs and Types** "DTOs are mixed with type definitions. Options:

1. **Separate Folders** (Recommended)
   - Move DTOs → `dto/`
   - Keep types → `types/`
   - Clear separation

2. **Migrate to Shared DTOs**
   - Identify reusable DTOs → `@repo/shared-dtos`
   - Keep local DTOs → `dto/`

3. **Document Structure**
   - Add README explaining current structure
   - Define naming convention

What's your preference?"

---

## Code Quality Standards

### ESLint and Prettier Requirements

**MANDATORY**: Every package and app MUST have ESLint and Prettier configured.

#### Required Tools

- **ESLint**: Code linting with strict rules for TypeScript and React
- **Prettier**: Code formatting for consistent style
- **TypeScript**: Strict type checking enabled

#### Package Setup

Every package and app MUST include these in `package.json`:

```json
{
  "scripts": {
    "lint": "eslint . --max-warnings=0",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write \"src/**/*.{ts,tsx}\"",
    "format:check": "prettier --check \"src/**/*.{ts,tsx}\"",
    "type-check": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest watch",
    "test:coverage": "vitest run --coverage"
  },
  "devDependencies": {
    "@repo/eslint-config": "workspace:*",
    "eslint": "^9.18.0",
    "prettier": "^3.4.2",
    "vitest": "^2.0.5"
  }
}
```

#### Build Tools

- **Use esbuild** for fast builds in all packages
- **Vitest** for testing with `vitest.config.ts` in every service package

#### ESLint Configuration

Use appropriate config:

- `@repo/eslint-config/base` - Most packages
- `@repo/eslint-config/next-js` - Next.js apps
- `@repo/eslint-config/react-internal` - React non-Next.js packages

#### Validation Commands

```bash
pnpm lint          # Check for linting errors
pnpm format:check  # Check formatting
pnpm type-check    # Check TypeScript types
```

Hooks automatically run formatting, linting, and type checking after file edits.

---

## Type Safety Rules

### Core Rules

- **NEVER use `any` or `unknown`** except when parsing external data with safeParse
- **ALWAYS use typed clients**: Import createClient from package exports
- **NEVER use loose Json**: Use `ConstrainedJson` from `@repo/shared-dtos/base`
- **ALWAYS validate external inputs**: Use Zod schemas at boundaries
- **EVERY Request DTO MUST have a Response DTO**
- **ALWAYS declare @repo/\* dependencies** in package.json when importing
- **NEVER duplicate service initialization** - create shared utilities in utils/
- **VALIDATE environment variables** before service creation with descriptive errors

### Example Implementation

```typescript
// ✅ CORRECT: Parsing unknown data
const result = safeParse(RequestSchema, req.body);
if (!result.success) {
  return { error: result.error };
}

// ❌ WRONG: Using any
const data: any = req.body; // NO!

// ✅ CORRECT: Using typed client
import { createClient } from "@repo/supabase";
const supabase = createClient();

// ❌ WRONG: Untyped client
import { createClient } from "@supabase/supabase-js";
const supabase = createClient(url, key); // Missing Database type!
```

### DTO Request-Response Pairing

```typescript
// ✅ CORRECT: Every request has a response
export const CreateReportRequestSchema = z.object({
  teamId: z.string().uuid(),
  reportType: z.enum(['summary', 'detailed']),
});

export const CreateReportResponseSchema = z.object({
  reportId: z.string().uuid(),
  status: z.enum(['pending', 'completed']),
});

// ❌ WRONG: Request without response
export const CreateReportRequestSchema = z.object({...}); // Where's the response?
```

### Database Type Reuse

**CRITICAL**: Always reuse database types from `@repo/supabase` instead of creating duplicate type
definitions. This maintains a single source of truth.

```typescript
// ✅ CORRECT: Reuse database types with Zod validation
import type { Database } from "@repo/supabase";

// Use the database type directly
type SocialAccountRow = Database["public"]["Tables"]["social_accounts"]["Row"];

// Create Zod schema that validates against the database type
const SocialAccountSchema = z.object({
  id: z.string().uuid(),
  teamId: z.string().uuid(),
  userId: z.string().uuid(),
  provider: z.string(),
  // ... match all fields from database
}) satisfies z.ZodType<{
  id: string;
  team_id: string; // Note: Use snake_case to match DB
  user_id: string;
  provider: string;
  // ... other fields
}>;

// ❌ WRONG: Duplicating type definitions
const SocialAccountSchema = z.object({
  id: z.string(),
  teamId: z.string(),
  // ... manually defining all fields again
});
```

**Key principles:**

- Import database types from `@repo/supabase`
- Use `satisfies z.ZodType<DatabaseType>` to ensure schema matches
- Transform snake_case to camelCase only at API boundaries
- Never manually duplicate database field definitions

### Critical Build & Type Safety Rules

- **ALWAYS run `pnpm install` and commit pnpm-lock.yaml** when adding dependencies to prevent Vercel
  frozen-lockfile errors
- **NEVER use `as any`** - create proper types with database joins like
  `UserOAuthTokenWithSocialAccounts`
- **ALWAYS run `pnpm db:generate-types`** after database schema changes to sync TypeScript types
- **EVERY package exporting TypeScript MUST have `build:types` script** to prevent "declaration file
  not found" errors
- **NEVER implement service consumers before checking service interfaces** - verify methods exist
  before using them
- **STANDARDIZED TESTING: Use `pnpm test:unit` for all packages** - runs unit tests across all
  packages (LangGraph, Fastify, Next.js, monorepo packages) from root level

---

## Testing Strategy

### Test-Driven Development (TDD)

- Write tests first (TDD approach)
- Focus on integration tests over unit tests
- Always test error states and edge cases
- Use factories for test data generation

### Test Data Creation with Factories

**IMPORTANT**: Always use the test factories from `@repo/supabase/factories` for creating test data.
This ensures consistency, handles circular dependencies, and properly manages RLS policies.

```typescript
import {
  createTestUser,
  createTestOrganization,
  createTestTeam,
  createTestMember,
  createCompleteTestSetup,
} from "@repo/supabase/factories";

// Basic usage - create individual entities
const testUser = await createTestUser();
const { organization } = await createTestOrganization({
  userId: testUser.user.id,
});
const { team } = await createTestTeam({
  organizationId: organization.id,
  userId: testUser.user.id,
});

// Complete setup with multiple users and proper permissions
const setup = await createCompleteTestSetup();
// Returns: admin, member, nonMember users with proper org/team setup
```

Available factories:

- **User**: `createTestUser()`, `createTestUsers()`, `loginTestUser()`
- **Organization**: `createTestOrganization()`, `createTestOrganizationWithMembers()`
- **Team**: `createTestTeam()`, `createTestTeams()`, `deleteTestTeam()`
- **Member**: `createTestMember()`, `createTestMembers()`
- **Media**: `createMediaAssetFactory()`, `createTestMediaCollection()`
- **Content**: `createContentItemFactory()`, `createContentItemsFactory()`
- **Complete Setup**: `createCompleteTestSetup()` - Full org/team/users setup

### Test Speed and Scale Philosophy

**Core Principle**: The majority of tests should be fast to enable rapid development cycles.

- **Fast Tests (< 100ms)**: Unit tests, pure function tests, isolated component tests
- **Medium Tests (100ms-1s)**: Integration tests with database, API calls with real services
- **Slow Tests (1s+)**: E2E tests, full browser automation, complex workflows

**Target Distribution**:

- 70% Fast tests (immediate feedback)
- 25% Medium tests (comprehensive coverage)
- 5% Slow tests (critical user journeys)

### Test Isolation and Dependency Management

**No Mocking Philosophy**: Everything should be faked with proper dependency injection.

```typescript
// ✅ GOOD: Fake with dependency injection
const createUserService = (deps: {
  database: Database;
  emailService: EmailService;
  logger: Logger;
}) => {
  // Service implementation
};

// In tests - inject test implementations
const fakeDatabase = createFakeDatabase();
const fakeEmailService = createFakeEmailService();
const fakeLogger = createFakeLogger();

const userService = createUserService({
  database: fakeDatabase,
  emailService: fakeEmailService,
  logger: fakeLogger,
});

// ❌ BAD: Mocking with vi.mock()
vi.mock("../email-service");
const mockEmailService = emailService as MockedFunction<typeof emailService>;
```

**Benefits of Faking over Mocking**:

- Tests are more realistic and catch integration issues
- Fakes can be reused across multiple tests
- Easier to reason about test behavior
- Less brittle - doesn't break when implementation changes

### Test Execution Strategy

**Tiered Test Execution**: Run different test suites at different stages of development.

#### PR Merge Requirements

```bash
# Fast + Medium tests - must complete < 5 minutes
pnpm test:unit
pnpm test:integration
pnpm test:component
pnpm build
```

#### Post-Merge Pipeline

```bash
# All tests including slow E2E - can take 10+ minutes
pnpm test:unit
pnpm test:integration
pnpm test:component
pnpm test:e2e
pnpm build
```

#### Pre-Production Gate

```bash
# Full test suite + additional checks
pnpm test:e2e:full          # Complete E2E test suite
pnpm test:performance       # Performance regression tests
pnpm test:accessibility     # A11y compliance tests
pnpm test:security         # Security vulnerability tests
```

### Test Organization by Speed

```typescript
// Fast tests - Pure functions, no I/O
describe("UserValidator (Fast)", () => {
  it("should validate email format", () => {
    expect(validateEmail("test@example.com")).toBe(true);
  });
});

// Medium tests - Real database, injected services
describe("UserService (Integration)", () => {
  beforeEach(async () => {
    await setupTestDatabase();
  });

  it("should create user with real database", async () => {
    const userService = createUserService({
      database: testDatabase,
      emailService: fakeEmailService,
      logger: fakeLogger,
    });

    const result = await userService.createUser({
      email: "test@example.com",
      name: "Test User",
    });

    expect(result.success).toBe(true);
  });
});

// Slow tests - Full browser automation (Playwright)
describe("User Registration (E2E)", () => {
  it("should complete full registration flow", async () => {
    await page.goto("/register");
    await page.fill('[data-testid="email"]', "test@example.com");
    await page.click('[data-testid="submit"]');
    await expect(page).toHaveURL("/dashboard");
  });
});
```

### React Component Testing

- Prefer runtime tests over mocks
- Use `renderWithProviders` for full provider context
- Focus on user interactions with `userEvent`
- Test what users see, not internals

```typescript
// Use real components and user interactions
import { renderWithProviders } from "@/test-utils";
import userEvent from "@testing-library/user-event";

const user = userEvent.setup();
renderWithProviders(<MyForm onSubmit={onSubmit} />);
```

## Automatic Feedback Loops

1. Run all test suites after commits - fix any failures
2. Check GitHub Actions status (except main branch)
3. Monitor Vercel deployments for build failures

### Worktree Development

**Setup and Management:**

- `./scripts/manage-worktree.sh create|list|switch|remove <name>` - Manage worktrees
- Each worktree gets its own database schema (e.g., `wt_feature_auth`)
- Always create a feature branch when working in a worktree
- Tests run in complete isolation

**Schema Detection System:**

- Automatic detection via `DATABASE_SCHEMA` environment variable
- Zero configuration needed
- Full TypeScript support

For detailed worktree setup and troubleshooting, see the full documentation in the repository wiki.

---

## Development Workflow

### Git Workflow

- Always create a feature branch for new work (e.g., `feature/chat-message-sync`)
- Never commit directly to main branch
- Write clear, descriptive commit messages using conventional commits format
- Always run tests before committing
- Keep commits focused and atomic
- Create pull requests for code review before merging

### Development Process

1. Start with failing tests
2. Implement minimal code to pass tests
3. Refactor while maintaining test coverage
4. Document patterns and learnings
5. Run type checking: `npx tsc --noEmit`
6. Run all test suites before finishing

---

## Media Asset Seeding

### Overview

The platform includes an automated media seeding system that provides developers with sample media
assets (images, audio, video) enriched with AI-generated metadata. This ensures a consistent
development experience and enables testing of the agentic media composition features.

### Automatic Seeding

Media assets are automatically seeded when you run `pnpm dev` for the first time. The system:

1. Checks if media assets have already been seeded
2. If not, automatically runs the seeding process
3. Uploads sample media to Supabase storage
4. Enriches media with AI metadata (titles, descriptions, emotions, purposes)
5. Generates vector embeddings for semantic search

### Manual Commands

```bash
# Run media seeding independently
pnpm seed:media

# Verify AI metadata was generated
pnpm seed:media:verify

# Reset and re-seed (useful for testing)
pnpm seed:media:reset && pnpm seed:media

# Run demo of the entire flow
./scripts/demo-media-seeding.sh
```

### Technical Details

The seeding system consists of:

1. **TypeScript Orchestrator** (`packages/supabase/scripts/seed-media.ts`)
   - Checks system_settings table for seeding status
   - Manages idempotent execution
   - Records completion metadata

2. **Python Processor** (`scripts/media-processor/process_media_assets_direct.py`)
   - Uploads files to Supabase storage
   - Analyzes content with Google Gemini AI
   - Generates rich metadata and embeddings

3. **Sample Media** (located in project root)
   - `audio/` - Sound effects and music tracks
   - `background-images-for-carousels/` - Background images
   - `green-screen-memes/` - Meme templates
   - `video/` - Video clips

### AI-Generated Metadata

Each media asset receives:

- **Basic Info**: Title, description, tags
- **Emotions**: Detected emotional tones
- **Purposes**: Suggested use cases
- **Scene Analysis**: Objects, actions, settings
- **Technical Data**: Duration, dimensions, format
- **Vector Embeddings**: For semantic search

### Configuration

Environment variables in `scripts/media-processor/.env`:

```bash
# AI Analysis (Google Gemini)
GOOGLE_API_KEY=your_gemini_api_key

# Team Assignment
DEFAULT_TEAM_ID=44444444-5555-6666-7777-888888888888

# Control AI Processing
SKIP_AI_ANALYSIS=false  # Set to true for faster seeding without AI
```

### Troubleshooting

- **Python Dependencies**: The system automatically creates a virtual environment and installs
  dependencies
- **Missing API Key**: Set `SKIP_AI_ANALYSIS=true` to seed without AI analysis
- **Re-seeding**: Use `pnpm seed:media:reset` to clear the seeding flag and run again

---

## Common Commands

### Development

```bash
# Core Development
pnpm dev                                    # Start all services (checks for tunnel automatically!)
pnpm dev:no-temporal                        # Start without Temporal
pnpm dev:no-tunnel-check                    # Start without tunnel check
pnpm db:start && docker compose --profile dev up  # Start full development environment

# Cloudflare Tunnel (for OAuth/HTTPS)
pnpm setup:tunnel                           # One-time setup for HTTPS URLs
pnpm tunnel                                 # Start tunnel manually
pnpm tunnel:stop                            # Stop tunnel

# Testing
pnpm test:unit                              # Run unit tests across all packages (STANDARDIZED)
pnpm test                                   # Run component tests
pnpm test:integration                       # Run integration tests
pnpm test:integration:watch                 # TDD workflow
pnpm test:e2e                              # Run end-to-end tests
monitor-logs.sh                             # Debug RLS issues

# Type Checking
npx tsc --noEmit                           # Check types across monorepo
```

### Video Generation Platform

```bash
pnpm --filter @repo/temporal dev:worker     # Start Temporal worker locally
docker compose --profile temporal up        # Start Temporal server and worker in Docker
pnpm temporal                               # Start Temporal CLI
# Access Temporal Web UI at http://localhost:8233
```

### Infrastructure Services

- **Temporal**: Workflow orchestration at http://localhost:7233 (gRPC) and http://localhost:8233
  (Web UI)
- **Redis**: Caching and session storage at localhost:6379
- **Supabase**: Database and auth via `pnpm db:start` (supabase local stack)
- **API**: Backend services at http://localhost:8080 (local)
  - **Swagger Documentation**: Interactive API docs at http://localhost:8080/swagger
- **Web**: Frontend at http://localhost:3000 (local)
- **LangGraph**: AI workflow orchestration at http://localhost:2024 (local)
- **Inbucket**: Email testing server at http://127.0.0.1:54324 (view sent emails in development)

---

## Technology Stack

### Frontend Stack

#### Next.js 15 with App Router

```typescript
// Dynamic Route Params - Always await params
export default async function Page({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
}

// Cookies & Headers - Await these from next/headers
const cookieStore = await cookies();
```

- Use `(group)` syntax for route groups
- Use `@slot` for parallel routes
- Use `(..)` for intercepting routes
- Use client components by default (migrating to Tanstack Start)

#### shadcn/ui Component System

- **Components**: Located in `apps/web/components/ui/`
- **Theme**: Configured in `apps/web/components.json`
- **Add Components**: `npx shadcn-ui@latest add <component-name>`
- Built on Tailwind CSS with CSS variables for theming
- Dark/light mode support via next-themes

#### Tailwind CSS Guidelines

- Use flex/grid with gap (not margins)
- Mobile-first responsive design
- Consistent spacing scale: 4, 8, 12, 16, 24, 32
- Use `cva` for complex component variants

### Backend Stack

#### Supabase Architecture

```typescript
// Client-side (apps/web/utils/supabase/client.ts)
const supabase = createClient();

// Server-side (apps/web/utils/supabase/server.ts)
const supabase = createClient();

// Admin operations (apps/web/utils/supabase/admin.ts)
const supabaseAdmin = createAdminClient();
```

- PostgreSQL with Row Level Security (RLS)
- Built-in auth with JWT tokens
- File storage with CDN
- WebSocket subscriptions for real-time
- Never use `next-auth-helpers` - use utils/supabase

#### Database Patterns

- UUID v4 primary keys on all tables
- `created_at` and `updated_at` timestamps
- JSONB for extensible data models
- RLS policies with helper functions
- Audit logging is NOT required (no audit trail needed)

### State Management

#### TanStack Query

- **Client-Side Fetching**: Primary data fetching strategy
- **Query Key Factories**: Centralized query key management
- **Optimistic Updates**: For better UX in mutations
- **Background Refetching**: Automatic data synchronization
- **Cache Management**: Intelligent caching with proper invalidation
- **Use the axios instance**: from @/lib/axios.ts

**Query Pattern:**

```typescript
// Query keys factory pattern
export const userKeys = {
  all: ["users"] as const,
  lists: () => [...userKeys.all, "list"] as const,
  list: (filters: Record<string, any>) => [...userKeys.lists(), filters] as const,
  details: () => [...userKeys.all, "detail"] as const,
  detail: (id: string) => [...userKeys.details(), id] as const,
};

// Usage in hooks
function useUser(id: string) {
  return useQuery({
    queryKey: userKeys.detail(id),
    queryFn: () => fetchUser(id),
  });
}
```

- Client-side fetching is primary strategy
- Avoid `useEffect`, always use Tanstack Query
- Handle loading states with `isLoading` and `isFetching`
- Return full query results from hooks

#### Tanstack Store

- Global user state and preferences
- UI state (modals, sidebars)
- Temporary form data

### External Services

- **LangGraph**: AI workflow orchestration
- **Temporal**: Video processing workflows
- **Stripe**: Payment processing
- **OpenAI**: AI content generation

---

## Architecture Patterns

### Authentication & Authorization

- Action-based permissions with RBAC
- Database-level security with RLS
- HOC patterns: `<WithPermission action="manage_team_members">`
- Server and client-side route protection

#### Permission System Synchronization

**CRITICAL**: Keep these files in sync:

1. `packages/supabase/supabase/migrations/20240124000004_tbl_create_permissions.sql`
2. `apps/web/integration/setup.ts`

**Quick Guide**:

- Permission actions must match exactly
- Admin role gets ALL permissions
- Member exclusions: approve_content, manage_team_members, assign_roles, manage_organization,
  manage_organization_members, manage_billing
- Run `pnpm test:integration` to verify sync

### Centralized Error Handling

**402 Insufficient Credits**: Axios interceptor automatically:

- Extracts credit requirements from error response
- Gets current balance from user store
- Shows detailed toast with specific credit info
- Prevents duplicate toasts

```typescript
// In mutation hooks, skip 402 handling as it's handled globally
onError: (error: any) => {
  if (error.response?.status === 402) return; // Already handled
  // Handle other errors...
};
```

## Quick Reference

### Key Patterns

- **TDD**: Integration tests first → RLS policies → component tests
- **Tanstack Query**: Client-side fetching with server actions
- **Supabase Migrations**: Run from `packages/supabase/supabase`
- **Import Paths**: `@repo/core-services/client` (Next.js), `@repo/core-services` (API)

### Global Hooks

Available in `apps/web/hooks/`: credit estimation, job definitions, keyboard shortcuts, mobile
detection, LangGraph integration, user state management.

## Project-Specific Rules

### Client vs Types Explanation

- **Next.js**: Use `@repo/core-services/client`
- **API**: Use `@repo/core-services` directly

### Tanstack Query Patterns

- Client-side fetching with factory pattern query keys
- Handle `isLoading` and `isFetching` states
- Return full query results from hooks

### Query Key Standards

- First element = exact PostgreSQL table name (with underscores)
- Import from `@repo/supabase/lib/query-keys` for type-safe keys
- Example: `tableQueryKeys.team_media_assets.all` // ["team_media_assets"]

### Mutation Metadata Standards

```typescript
import { createQueryMetadata } from "../types/react-query-metadata";

return useMutation({
  meta: createQueryMetadata("create", "table_name"),
  mutationFn: async ({ data }) => createResource({ supabase, data }),
});
```

- Operations: "create", "update", "delete", "read"
- Table names: exact PostgreSQL names with underscores

### TDD Approach

- Start with `.integration.ts` tests
- Test with real database/RLS
- Use `@repo/supabase` factories
- Debug RLS with `monitor-logs.sh`

### Supabase Migration Guidelines

- Navigate to `packages/supabase/supabase` first
- Run `npx supabase migration new <file_name>`
- Naming: `sys_` RPCs, `enum_` enums, `tbl_` tables, `rls_` policies, `fn_` functions
- **IMPORTANT**: Audit trails are NOT required - do not add audit logging triggers

### React Component Patterns

- Await dynamic route params and cookies
- Toast notifications in CRUD layer (react-hot-toast, not sonner)
- Test with `renderWithProviders` from `@/test-utils`

### Next.js Feature Organization

**ALWAYS use the features/ directory pattern:**

```
apps/web/
├── app/
│   └── (application)/
│       └── org/[orgId]/[teamId]/
│           └── feature-name/
│               └── page.tsx  # ONLY the route entry point
├── features/
│   └── feature-name/         # ALL feature logic goes here
│       ├── components/       # Feature-specific components
│       ├── hooks/           # Feature-specific hooks
│       ├── pages/           # Main page components
│       ├── utils/           # Feature utilities
│       ├── types/           # Feature types
│       └── index.ts         # Export everything
```

**Implementation**:

```typescript
// features/your-feature/pages/YourFeaturePage.tsx
export default function YourFeaturePage({ teamId }: { teamId: string }) {
  // All feature logic here
}

// app/(application)/org/[orgId]/[teamId]/your-feature/page.tsx
import { YourFeaturePage } from "@/features/your-feature";

export default async function YourFeatureRoute({ params }: PageProps) {
  const { teamId } = await params;
  return <YourFeaturePage teamId={teamId} />;
}
```

### Global Hooks

Available in `apps/web/hooks/`: credit estimation, job definitions, keyboard shortcuts, mobile
detection, LangGraph integration, user state management.

### API Versioning in Client

**IMPORTANT**: The axios instance in `@/lib/axios.ts` already includes the `/api/v1` prefix from
`config.environment.apiUrl`. Do NOT manually add `/api/v1` to your API calls - just use the endpoint
path starting with `/`.

---

## Platform Thinking

### Core Philosophy

Build platform-first, not feature-first. Each service = composable building block. We are designing
a platform, a product (client side + API + Temporal). This is a production grade platform.

The bet that we are making is that we can build a platform that is flexible enough to support a
range of use cases for media generation. We plan to offer both a great UX (react front end) and also
a powerful, typesafe API (fastify + supabase + temporal).

We value OpenAPI.yml, type safety, and a well-designed API.

**Core Principles:**

- **Infrastructure as Leverage**: Services multiply future output
- **Build Once, Use Many**: Reusable across features
- **Optionality Through Architecture**: More infrastructure = more possibilities
- **Progressive Enhancement**: Platform foundations with pragmatic shortcuts

### Platform vs Product Mindset

**Platform ✅**: Composable services, flexible JSONB schemas, recombinable APIs, infrastructure as
moat

**Product ❌**: Single-purpose features, hardcoded solutions, tight coupling, immediate needs only

### Multi-Format Content Strategy

All content formats in: `./packages/core-services/src/content/types.ts`

- Start with one format → Architecture supports all formats
- Flexible data models → Native content repurposing

### Implementation

1. **Service Design**: Clear boundaries and interfaces
2. **Data Flexibility**: JSONB for extensible models
3. **API First**: Design with API architecture in mind
4. **Cross-Format Intelligence**: Services learn across content types
5. **Tiered Systems**: Fallback strategies for reliability

### DTO Architecture

`@repo/shared-dtos` package organized into:

1. **Base DTOs** (`/base`): API wrappers, pagination, metadata, ConstrainedJson
2. **Progressive DTOs** (`/progressive`): Client → Fastify → Temporal flow
3. **Point-to-Point DTOs** (`/point-to-point`): Auth, credits, simple CRUD

```typescript
// Import from subpaths for tree-shaking
import { ApiResponse } from "@repo/shared-dtos/base";
import { VideoGenerationClientRequest } from "@repo/shared-dtos/progressive";
import { LoginRequest } from "@repo/shared-dtos/point-to-point";

// Use validation utilities
import { safeParse } from "@repo/shared-dtos";
const result = safeParse(LoginRequestSchema, req.body);
```

---

## Media Optimization Strategy

### Core Philosophy: Optimization Service in Disguise

We are fundamentally an **optimization service** secretly disguised as a media generation company.
Our users need not just automation, but intelligent creative testing to discover what truly works.

### Agent-Based Cost Optimization

When implementing the media rendering agent, follow these principles:

#### 1. Cost Function Optimization

- **Minimize cost** while maintaining quality thresholds
- **Weight parameters** based on performance history
- **Dynamic optimization** across multiple content variations

#### 2. Dynamic Budget Management

```typescript
// Conceptual approach for budget threshold management
interface BudgetStrategy {
  iteration: number;
  remainingBudget: number;
  qualityScore: number;
  maxTurns: number;
}

// Dynamic threshold adjustment
// 1st iteration: Conservative (use 30% of budget)
// 2nd iteration: Moderate (use 50% of remaining)
// 3rd iteration: Aggressive (use up to 80% of remaining)
```

#### 3. Margin Strategy

- Apply **1.5x to 2.0x margin** on computed costs
- "Ask for more than you can get away with"
- Balance between value delivery and sustainable pricing

#### 4. User Budget Configuration

- **Always ask users** for maximum budget per content piece
- Implement budget allocation across content variations
- Provide transparent cost breakdowns

### Learning Cycles Architecture

Implement continuous improvement through data-driven cycles:

```
Concept (string) → Media (metadata) → Performance → Creative Format Performance
     ↑                                                          ↓
     └──────────────── Optimization Feedback ──────────────────┘
```

**Key Components**: Concept Analysis → Media Generation → Performance Tracking → Format Learning →
Feedback Loop

### Implementation Guidelines

1. **Quality-Budget Balance**: Quality score + turn limits, never exceed budget
2. **Creative Testing**: A/B test variations, track metrics, build pattern knowledge
3. **Transparent Optimization**: Show budget allocation and performance insights

### Composition Pipeline Architecture

We're building a sophisticated **dual-composition system** that combines:

#### 1. Static Composition Pipeline

- **Pre-defined templates** with optimized rendering paths
- **Predictable resource usage** for cost estimation
- **Fast execution** for common content patterns
- **Version-controlled templates** for consistency

#### 2. Dynamic Composition Pipeline

- **Runtime asset selection** based on performance data
- **AI-driven layout decisions** within budget constraints
- **Adaptive quality settings** based on remaining budget
- **Real-time optimization** during generation

```typescript
// Conceptual composition strategy
interface CompositionStrategy {
  type: "static" | "dynamic";
  budget: BudgetConstraints;
  performanceHistory: PerformanceMetrics[];
  targetPlatform: Platform;
}

// Platform-specific optimizations
type Platform = "tiktok" | "instagram" | "youtube" | "x" | "linkedin";
```

### Multi-Platform Content Evolution

Our platform architecture is designed for **progressive platform expansion**:

**Platforms**: TikTok, Instagram Reels, YouTube (Shorts & Long-form) → Future: X.com, LinkedIn,
Cross-platform campaigns

**Platform Optimization**: Format constraints, algorithm preferences, audience expectations,
performance metrics → maximize ROI across platforms.

---

## Architectural Rules for AI-Native Development

### Service Layer Requirements

#### When to Create a Service Package

```typescript
// Decision framework for service creation
if (
  complexity > 50_lines ||
  consumers >= 2 ||
  hasBusinessRules ||
  hasCreditsOrBilling ||
  needsConsistency
) {
  createServicePackage();
} else {
  keepInFastifyRoute();
}
```

#### Service Implementation Pattern

- Always use factory functions: `createXService(deps)`
- Explicit dependency injection
- Return object with methods
- NO classes, NO direct instantiation

### Platform Extensibility

#### Provider Pattern for Social Platforms

```typescript
// Base provider interface for all platforms
interface SocialProvider {
  post(content: Content): Promise<PostResult>;
  collectAnalytics(postId: string): Promise<Analytics>;
  refreshToken(): Promise<TokenResult>;
  validateContent(content: Content): ValidationResult;
}
```

#### Normalized Metrics

- Use `post_performance` table for ALL platforms
- Platform-specific data in `platform_metrics` JSONB field
- Common metrics: views, likes, comments, shares, saves
- Calculate normalized scores across platforms

### Learning Loop Architecture

```
Creative Concepts → Content Items → Generations → Posts → Performance
         ↑                                                      ↓
         └────────── AI Analysis & Optimization ───────────────┘
```

#### Key Components

1. **Creative Concepts**: AI-generated ideas with predicted performance
2. **Content Items**: Actual content created (with optional creative_concept_id FK)
3. **Generations**: Rendered media (videos, images)
4. **Performance Tracking**: Normalized metrics across platforms
5. **Feedback Loop**: Performance data improves future concepts

---

## Important Reminders

### Pre-Task Checklist

1. Type check: `npx tsc --noEmit`
2. Run tests: `pnpm test` → `pnpm test:integration` → `pnpm test:e2e`

### Pre-Implementation Checklist

- [ ] Check if similar functionality exists in utils/ directories
- [ ] Verify all imported @repo/\* packages are in package.json dependencies
- [ ] Validate environment variables before service initialization
- [ ] Use proper union types instead of any assertions
- [ ] Extract duplicate functions into shared utilities

### Key Rules

- Never use `next-auth-helpers` - use `utils/supabase`
- Always use Vitest, not Jest
- Toast notifications handled in CRUD hooks
- Use `renderWithProviders` for testing
- Delete temporary `.sh` scripts after use
- Import Supabase hooks from the monorepo package
- Never comment out code to get tests passing - always fix the underlying issue
- ALL AI operations must track credits through API layer

### Strategic Priorities

#### Core Product Vision: Optimization Service in Disguise

**We are fundamentally an optimization service disguised as a media generation company.**

**The Complete Optimization Loop:**

1. **Strategize**: AI-powered content strategy based on trends and performance data
2. **Create**: Multi-format content generation (video, images, carousels, text)
3. **Publish**: Automated distribution across platforms with optimal timing
4. **Analyze**: Performance tracking and pattern recognition across all content
5. **Re-strategize**: Continuous optimization based on what actually works

**Customer Value**: 80% reduction in content creation time + continuous performance improvement
through AI-driven optimization.

#### Multi-Content Strategy

- **Long-term vision**: Multi-format content generation (video, images, carousels, text)
- **Cross-platform optimization**: TikTok, Instagram, YouTube, X, LinkedIn
- **Content repurposing**: One concept → multiple formats automatically
- **Performance learning**: Track format effectiveness across platforms

#### User Onboarding Flow

- **Split user types**: Social media manager vs standard user
- **Budget collection**: Always ask for max budget per content piece
- **Website URL**: Capture for brand analysis and content alignment
- **Platform preferences**: Collect target social platforms during signup
- **Flawless UX**: Zero friction from signup to first content generation

### Vector Embeddings with Supabase

**IMPORTANT**: Supabase generates vector columns as `string` type in TypeScript, not `number[]`.
This is intentional, not a bug.

When storing vector embeddings:

- Use `JSON.stringify(embedding)` to convert arrays to strings
- Never use string concatenation like `[${embedding.join(',')}]`
- When retrieving, use `JSON.parse()` if you need array operations

```typescript
// ✅ CORRECT: Storing embeddings
const embedding = await generateEmbedding(text); // Returns number[]
await supabase.from("table").update({
  vector_embedding: JSON.stringify(embedding),
});

// ❌ WRONG: String concatenation
vector_embedding: `[${embedding.join(",")}]`; // Don't do this!
```

This is required because PostgreSQL's vector extension expects string formatting.

## Documentation

### Documentation Organization Rule

**CRITICAL**: To optimize token usage and maintain clarity:

#### Localized Knowledge → Local CLAUDE.md Files

- Package-specific patterns, APIs, and implementation details
- Service-specific configuration and examples
- Domain-specific business logic
- Technical details only relevant to that package/app

#### Global Knowledge → Root CLAUDE.md

- Cross-cutting architectural patterns
- Platform-wide principles and philosophy
- Shared coding standards and conventions
- Project planning and strategic decisions
- Integration patterns between services
- Development workflow and processes

### Package Documentation Rule

**MANDATORY**: Every package in `packages/` and app in `apps/` MUST have its own `CLAUDE.md` file
with package-specific guidelines. Keep root CLAUDE.md focused on cross-domain knowledge to minimize
token usage.

Examples:

- `apps/api/CLAUDE.md` → Fastify patterns, route organization, API gateway rules
- `packages/oauth-service/CLAUDE.md` → OAuth flows, provider patterns, token management
- `packages/supabase/CLAUDE.md` → Migration conventions, RLS patterns, database specifics

---

## Claude Debugging with Evaluation Framework

The `@repo/evaluation-framework` provides **universal code execution recording** and **multimodal AI
evaluation** capabilities specifically designed for Claude debugging and service layer optimization.

### Key Capabilities

1. **Universal Workflow Recording**: Captures ANY type of code execution (functions, classes,
   microservices, pipelines) as complete execution descriptions with full debug context
2. **Multimodal AI Evaluation**: Built-in evaluators for video, image, audio, and text quality
   analysis using Google Gemini
3. **Smart VCR Caching**: SHA256-based cache invalidation that automatically refreshes when source
   code changes
4. **Custom Evaluators**: Extensible system for creating domain-specific evaluators (brand
   compliance, platform optimization, etc.)
5. **Debug Integration**: Seamless integration with existing debugging workflows and tools

### Quick Usage Example

```typescript
import { createWorkflowRecorder } from "@repo/evaluation-framework/vcr";

const debugWorkflow = {
  workflowId: "service-debugging",
  executionDescription: `
    // COMPLETE code execution context
    const service = createUserService(deps);
    const result = await service.createUser(userData);
  `,
  inputs: { userData: { email: "test@example.com" } },
  stdout: "/* actual execution logs */",
  stderr: "/* actual error output */",
  environment: { NODE_ENV: "development" },
};

const { output, cassette } = await recorder.recordWorkflow(
  ["./src/services/user-service.ts"],
  debugWorkflow,
  executeFunction,
  evaluateFunction
);

// Cassette contains EVERYTHING Claude needs for debugging
```

### Available Evaluators

- **Video Quality**: TikTok/Instagram optimization, caption readability, engagement analysis
- **Image Quality**: Composition, color balance, rule of thirds, platform-specific requirements
- **Composite**: Multi-modal evaluation with configurable weights
- **Custom**: Create evaluators for brand compliance, SEO, engagement, or any domain-specific needs

### Integration Patterns

Works seamlessly with:

- **Service Layer Debugging**: Complete microservices execution recording
- **Temporal Workflows**: Quality gates and validation steps
- **API Routes**: Content evaluation endpoints with credit tracking
- **LangGraph Agents**: Self-improving agents with evaluation feedback

For complete documentation, examples, and implementation details, see:
**`packages/evaluation-framework/CLAUDE.md`**

---

## TypeScript Build Order & Declaration Files

### The Problem: "Could not find a declaration file for module" Errors

**Root Cause**: TypeScript declaration files (`.d.ts`) are generated during the build process, but
other packages try to import from them before they exist.

**Example Error**:

```
Could not find a declaration file for module '@repo/social-media-service'
```

### Why This Happens

1. **Turborepo builds dependencies in parallel** by default
2. **TypeScript compilation happens simultaneously** across packages
3. **Declaration files are generated during build**, not before
4. **Import resolution fails** when `.d.ts` files don't exist yet

**The Race Condition**:

```
@repo/temporal (importing) → @repo/social-media-service (not built yet)
     ↓                              ↓
Tries to find .d.ts files    →  Files don't exist = ERROR
```

### The Solution: Two-Phase Build System

We implement a **build:types** phase that runs before the main build:

```json
// turbo.json
{
  "build:types": {
    "dependsOn": ["^build:types"],
    "outputs": ["dist/**/*.d.ts", "dist/**/*.d.cts", "dist/**/*.d.mts"]
  },
  "build": {
    "dependsOn": ["^build", "build:types"]
  },
  "dev": {
    "dependsOn": ["^build:types"]
  }
}
```

**Build Flow**:

1. **Phase 1**: `build:types` - Generate only TypeScript declarations
2. **Phase 2**: `build` - Build JavaScript/bundles with types available
3. **Dev Mode**: Ensure types exist before starting development

### Package.json Requirements

**Every package that exports TypeScript must have**:

```json
{
  "scripts": {
    "build": "...", // Full build including types
    "build:types": "tsc --emitDeclarationOnly --declaration" // Types only
  }
}
```

**Examples by Build Tool**:

```json
// TSup (social-media-service)
"build:types": "tsup --dts-only"

// TypeScript Compiler (supabase, core-services)
"build:types": "tsc --emitDeclarationOnly --declaration"

// Custom Build (shared-dtos)
"build:types": "tsc" // Part of multi-step build
```

### Dependency Graph Requirements

**Critical**: If package A imports from package B, then A must list B as a dependency:

```json
// packages/temporal/package.json
{
  "dependencies": {
    "@repo/social-media-service": "workspace:*", // REQUIRED
    "@repo/core-services": "workspace:*",
    "@repo/shared-dtos": "workspace:*"
  }
}
```

**Missing dependencies cause**:

- Build order confusion
- TypeScript can't find declaration files
- Runtime import errors

### Prevention Checklist

**Before adding imports**:

1. ✅ Add workspace dependency to package.json
2. ✅ Ensure target package has `build:types` script
3. ✅ Test with `turbo run build:types --filter='package-name'`
4. ✅ Verify types exist in `dist/` folder

**During development**:

1. ✅ Run `turbo run build:types` if you see declaration errors
2. ✅ Check that imports match package.json dependencies
3. ✅ Never skip TypeScript errors - fix the root cause

### Common Patterns

**When you see**: `Could not find a declaration file for module '@repo/x'`

**Check these in order**:

1. Does the importing package list `@repo/x` in dependencies?
2. Does `@repo/x` have a `build:types` script?
3. Does `@repo/x/dist/` contain `.d.ts` files?
4. Run `turbo run build:types --filter='@repo/x'` to generate types

**Emergency fix**:

```bash
# Generate types for all packages
turbo run build:types

# Then rebuild the failing package
turbo run build --filter='failing-package'
```

---

## Development Environment & OAuth Configuration

### Live Development Environment

When working with OAuth integrations and scheduling pipelines, use the live development environment:

#### Shared Development Tunnel

**Recommended Approach**: Use the shared `api-dev.octospark.ai` tunnel for OAuth development:

- **Consistent URL**: All developers use the same API endpoint
- **Pre-configured**: Already set up with TikTok OAuth callbacks
- **No DNS conflicts**: Avoids multiple personal subdomains
- **Easy setup**: Just use "dev" as your name when running the setup script

**For personal tunnels**: Future developers can still create personal tunnels by using their own
name (e.g., "alice" → `api-alice.octospark.ai`)

#### TikTok OAuth Configuration

- **Sandbox Environment**: Already configured for development
- **Callback URL**: `https://api-dev.octospark.ai/oauth/tiktok/callback`
- **Client ID**: `sbaw4turoutm4e71gw` (sandbox)
- **Environment**: Development/Sandbox mode
- **Recommended Setup**: Use "dev" as your name when running tunnel setup script

#### Environment Setup Commands

```bash
# Setup tunnel environment (for OAuth testing)
pnpm setup:tunnel
# When prompted for name, use: dev

# Setup local environment (for regular development)
pnpm setup:local

# Test current environment configuration
pnpm test:env

# Start development with auto-detection
pnpm dev
```

#### Cloudflare Tunnel URLs

- **API URL**: `https://api-dev.octospark.ai` (shared development tunnel)
- **App URL**: `http://localhost:3000` (local development)
- **Supabase Local**: `http://localhost:54321`

> **Important**: When using tunnel mode, develop locally at `http://localhost:3000` but OAuth
> callbacks will use the tunnel URL `https://api-dev.octospark.ai/oauth/tiktok/callback`.

### OAuth Provider Configuration

#### TikTok Integration

- **Scopes**: `user.info.basic`, `user.info.stats`, `video.list`, `video.publish`, `video.upload`
- **Redirect URI**: `https://api-dev.octospark.ai/oauth/tiktok/callback`
- **API Endpoint**: `/api/v1/social-media/accounts/connect`

#### Environment Variables (Auto-configured)

```bash
# Tunnel Mode (Recommended for OAuth)
NEXT_PUBLIC_APP_URL=http://localhost:3000
NEXT_PUBLIC_API_URL=https://api-dev.octospark.ai
SUPABASE_AUTH_SITE_URL=http://localhost:3000
NEXT_PUBLIC_TUNNEL_MODE=api-only

# Local Mode (No OAuth)
NEXT_PUBLIC_APP_URL=http://localhost:3000
NEXT_PUBLIC_API_URL=http://localhost:8080
SUPABASE_AUTH_SITE_URL=http://localhost:3000
NEXT_PUBLIC_TUNNEL_MODE=false
```

### Scheduling Pipeline Debugging

#### Key Components

1. **OAuth Flow**: `/oauth/tiktok/callback` → Token storage
2. **Content Creation**: Content service → Slideshow generation
3. **Video Generation**: Temporal workflow → Media rendering
4. **Social Posting**: Scheduling service → TikTok API

#### Debugging Commands

```bash
# Check environment configuration
pnpm test:env

# Restart Supabase with new config
cd packages/supabase && npx supabase stop && npx supabase start

# Check tunnel status
pnpm tunnel:stop && pnpm tunnel

# Test Temporal workflows
cd packages/temporal && pnpm test
```

#### Common Issues & Solutions

- **OAuth Callback Errors**: Ensure tunnel is running and callback URL matches
- **Token Refresh Issues**: Check `user_oauth_tokens` table and refresh logic
- **Video Generation Failures**: Verify Temporal worker is running and has proper permissions
- **Scheduling Conflicts**: Check `scheduled_posts` table and job status

### Future OAuth Providers

This section will expand as more OAuth providers are added:

- Instagram Business API
- YouTube Data API
- LinkedIn API
- Twitter API v2

---
