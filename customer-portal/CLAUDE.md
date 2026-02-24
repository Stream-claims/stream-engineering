# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Common Development Commands

### Development

```bash
# Start development server (requires CUSTOMER_NAME in .env)
npm run dev

# Deploy to a specific stage (e.g. dev, staging, production)
# Requires CUSTOMER_NAME env var (set in .env or inline)
CUSTOMER_NAME={customer} pnpm sst deploy --stage {stage}
```

### Testing

```bash
# Run all tests (unit + integration)
npm run test

# Run unit tests with Vitest
npm run test:unit

# Run E2E tests with Playwright
npm run test:e2e
```

### Code Quality

```bash
# Run linting
npm run lint

# Format code
npm run format

# Type checking with SvelteKit
npm run check
npm run check:watch
```

### Build

```bash
# Build for production
npm run build

# Preview production build
npm run preview
```

## Architecture Overview

### Multi-tenant SaaS Platform

This is a customer portal for medical record processing, deployed as a multi-tenant SaaS application using SST (Serverless Stack). Each customer has:

-   Dedicated deployment configuration in `sst.config.ts`
-   Custom feature flags controlling available modules
-   Separate AWS Secrets Manager secrets
-   Environment-specific configurations (dev/staging/production)

### Core Technology Stack

-   **Frontend**: SvelteKit 5 with TypeScript
-   **Styling**: Tailwind CSS with Flowbite components
-   **Authentication**: AWS Cognito via @auth/sveltekit
-   **Infrastructure**: SST v2 for serverless deployment
-   **PDF Processing**: pdfjs-dist for client-side PDF rendering
-   **State Management**: Svelte stores in `src/lib/store/`

### Key Application Modules

The application is feature-flag controlled with these main modules:

1. **Case Management** (`/case`) - Central hub for medical cases
2. **Document Processing Pipeline**:
    - Segmentation - Document splitting and categorization
    - Review - Manual document review and validation
    - Sift (Dedupe) - Duplicate detection and removal
    - Extraction - Data extraction from documents
    - Summary - Case summarization
    - Insights - Visual analytics with body part mapping
    - PTD (Permanent Total Disability) - Disability assessment
    - Comparison - Document comparison
    - Merge - Document merging

### API Architecture

-   **V2 API Pattern**: Modern endpoints in `src/lib/api/v2/`
-   **Server Routes**: SvelteKit server endpoints in `src/routes/(authed)/`
-   **DTOs**: Type-safe data transfer objects in `src/lib/dto/`
-   **Error Handling**: Centralized error handling with Sentry integration

### Authentication Flow

1. AWS Cognito authentication via `src/auth.ts`
2. Protected routes under `(authed)` route group
3. JWT token refresh mechanism
4. Session management with access/refresh tokens

### Component Organization

-   **Common Components**: `src/lib/components/common/` - Shared UI elements
-   **Module Components**: `src/lib/components/[module]/` - Feature-specific components
-   **Layout Components**: Sidebar navigation with collapsible state management

### Environment Configuration

-   Requires `CUSTOMER_NAME` environment variable
-   Customer-specific configurations in SST stack
-   Feature flags control module availability per customer
-   Secrets stored in AWS Secrets Manager
-   `CUSTOMER_TYPE` controls routing: 'med_review' (default) or 'carrier'

### Customer Type Routing

The application supports route-based access control for different customer types:

1. **Med Review Customers** (default):

    - Full sidebar navigation with all modules
    - Complex workflow for medical record review
    - Standard routes: `/case`, `/segmentation`, `/review`, etc.

2. **Carrier Customers**:
    - Carrier-specific sidebar (`CarrierSidebar.svelte`) with streamlined navigation
    - Simplified interface for insurance carriers
    - Automatically redirected to `/carrier` on login
    - Restricted to `/carrier/*` routes only
    - Navigation focused on case management with paths like `/carrier/cases/*`

Configuration in `sst.config.ts`:

```typescript
customerType: 'carrier', // or 'med_review' (default)
```

**Route Guards**: Implemented server-side in `+layout.server.ts` to enforce access control:

-   Carrier customers can only access `/carrier/*` routes
-   Med review customers cannot access `/carrier/*` routes
-   Unauthorized access attempts result in automatic redirects

See [Route-Based Access Control Documentation](docs/ROUTE_ACCESS_CONTROL.md) for detailed implementation guide.

### Testing Strategy

-   Unit tests with Vitest for utilities and helpers
-   E2E tests with Playwright targeting test environment
-   Test files follow `*.test.ts` pattern

## Best Practices and Recommendations

-   **Linting**:
    -   Never run linters on the entire codebase, limit to specific files you changed


You are able to use the Svelte MCP server, where you have access to comprehensive Svelte 5 and SvelteKit documentation. Here's how to use the available tools effectively:

## Available MCP Tools:

### 1. list-sections

Use this FIRST to discover all available documentation sections. Returns a structured list with titles, use_cases, and paths.
When asked about Svelte or SvelteKit topics, ALWAYS use this tool at the start of the chat to find relevant sections.

### 2. get-documentation

Retrieves full documentation content for specific sections. Accepts single or multiple sections.
After calling the list-sections tool, you MUST analyze the returned documentation sections (especially the use_cases field) and then use the get-documentation tool to fetch ALL documentation sections that are relevant for the user's task.

### 3. svelte-autofixer

Analyzes Svelte code and returns issues and suggestions.
You MUST use this tool whenever writing Svelte code before sending it to the user. Keep calling it until no issues or suggestions are returned.

### 4. playground-link

Generates a Svelte Playground link with the provided code.
After completing the code, ask the user if they want a playground link. Only call this tool after user confirmation and NEVER if code was written to files in their project.
