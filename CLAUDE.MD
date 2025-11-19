# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Essential Commands
```bash
# Development server (recommended - ensures port 3000)
npm run dev


# Database operations - Never execute a data operation that could trigger data loss or database reset whithout user confirmation
npx prisma generate          # Generate Prisma client after schema changes
npx prisma migrate dev       # Apply database migrations

## DB Safety for risky db operations.
#1. Backup.  
#2. Run operations.  
#3. Check: `SELECT COUNT(*) FROM users;`  
#   - If 0 → rollback + stop + rethink  
#   - Else → log success

# Linting and building
npm run lint                 # Run ESLint
npm run build               # Production build
npm start                   # Start production server

# Utility scripts
npm run create-singles      # Create singles teams for users
npm run generate-dev-token  # Generate development master token
```

### Database Management
```bash
# Manual database backups
node scripts/backup-database.js

# Check database schema and data integrity
node check_db_schema.js
node check_duplicate_teams.js
```

### Testing and Development Credentials

To be determined

### API Testing with Development Master Token

### Frontend Development Tools (MCP)

**Microsoft Playwright MCP Server:**
- **Installation**: `claude mcp add playwright npx -- @playwright/mcp@latest`
- **Status**: ✅ Installed and available in this project
- **Usage**: For UI testing, design feedback, and frontend debugging
- **Capabilities**:
  - Browser automation without screenshots (accessibility tree-based)
  - Real-time UI testing and validation
  - Design feedback through direct element interaction
  - Mobile and desktop responsive testing
  - Authentication testing with visible browser sessions
- **Agent Access**: Available to `frontend-developer` specialized agent
- **Activation**: Use `/mcp` command and navigate to "playwright" for available tools
- **First Use**: Say "playwright mcp" explicitly to ensure proper tool selection

### Deployment (Fly.io) only on user explicit request
```bash
# Deploy to Fly.io - Only on user explicit request.
# ⚠️ WARNING: Regular deploy often fails with timeouts. Use extended timeout commands.

# RECOMMENDED: Deploy with extended timeout (ALWAYS use this first)
timeout 30m fly deploy

# Alternative for persistent timeout issues:
fly deploy --no-cache --remote-only

# If still failing, try longer timeout:
timeout 30m fly deploy --no-cache

# Last resort - manual deployment steps:
fly deploy --build-only    # Build image first
fly deploy --image <image>  # Deploy pre-built image

# Check deployment status
fly status

# View logs
fly logs

# SSH into production
fly ssh console

# ⚠️  CRITICAL: NEVER scale without authorization
# Scaling changes cost structure - always ask permission first
# fly scale count 1  # DO NOT RUN without explicit user approval
```

### Deployment Troubleshooting

**Common Issues & Solutions:**

1. **Timeout during build/push:**
   - Always use `timeout 30m fly deploy` as the default command
   - If build succeeds but push fails, use `--no-cache` flag
   - Network issues: retry with longer timeout

2. **App not listening on expected address:**
   - This warning is usually harmless if health checks pass
   - The app listens internally on port 3000, fly-proxy handles external routing
   - Check health endpoint: `curl https://ball-e-sqlite.fly.dev/api/health`

3. **DNS/Health check failures:**
   - Wait 2-3 minutes for DNS propagation
   - Check logs: `fly logs` for startup errors
   - Verify database connectivity in health endpoint

### ⚠️ Cost Control Guidelines

**CRITICAL DEPLOYMENT RULES:**
- **NEVER run `fly scale count X` without explicit user authorization**
- **NEVER change machine configurations without permission**
- **ALWAYS check current scaling with `fly status` before making changes**
- **ASK USER PERMISSION before any cost-affecting operations**

The application uses auto-scaling (min 0, auto-stop/start) to minimize costs. Scaling commands can significantly impact hosting expenses.

## Architecture Overview

### Tech Stack
- **Framework**: Next.js 15.4.1 with App Router
- **Database**: SQLite
- **Authentication**: JWT + bcrypt
- **Validation**: Zod (backend + frontend)
- **UI**: Tailwind CSS + React 19
- **Language**: TypeScript

### Core Database Models
- **ALWAYS check database schema before coding for or querying the database**

#### Usuarios y Accesos
- **Usuario**: Users with roles (SuperAdmin or regular), availability, and soft delete support
- **Sociedad**: List of companies (SAS) whose books are being managed
- **UsuarioXSociedad**: Access for users to a certain "sociedad" (Owner, Read-Write, Read-Only)

#### Libros y Actos
- **Libro**: Digital books for each company (Actas Socios, Actas Administración, Registro Acciones, Diario, Inventarios y Balances, IVA Compras, IVA Ventas)
- **Acto**: Individual acts/documents belonging to a book with states (Draft, Pendiente TAD, Final, Error TAD)
- **ActoMetadata**: Extracted metadata with versioning and traceability
- **ActoVersion**: Draft version history for acts

#### Contexto y Templates
- **Contexto**: Living textual document summarizing the company's corporate situation
- **ContextoVersion**: Version history of the context document
- **Template**: Parameterizable act templates managed by SuperAdmin
- **TemplateVersion**: Template version history

#### Hashes y TAD
- **DocumentHash**: Hash records for each document (hash_real, hash_anterior_declarado)
- **ComprobanteTAD**: TAD receipts uploaded by users
- **CadenaHash**: Hash chain validation records per book

#### Auditoría y Notificaciones
- **AuditLog**: Complete audit log (user, timestamp, action, entity, description)
- **Notification**: Email orchestration with quotas and logging

### AI Integration Strategy
- **Provider**: OpenRouter (Vercel AI SDK)
  - Flexibility to switch models (GPT-4, Claude, Llama, etc.)
  - Automatic fallback if a model is down
  - Easy cost comparison between models
- **Use Cases**:
  - Document classification (FR-06)
  - Data extraction from PDFs (FR-08)
  - Context generation and updates (FR-10, FR-11)
  - Act text generation with templates (FR-17)
  - Summary generation (FR-34)
- **Error Handling**: Fallback to manual classification when confidence is low
- **Cost Management**: Rate limiting and usage quotas per sociedad
- **Prompt Management**: Centralized prompt templates in `/src/lib/ai/prompts/`
- **Configuration**: Model selection per use case in environment variables

### PDF Processing Strategy
- **Text Extraction**:
  - First attempt: `pdf-parse` or similar for text-based PDFs
  - Fallback: Vision API for scanned/image-based PDFs
- **Vision API Options** (via OpenRouter):
  - GPT-4 Vision for complex documents
  - Claude 3 Vision as alternative
  - Gemini Pro Vision for cost optimization
- **Processing Pipeline**:
  1. Detect if PDF has selectable text
  2. If yes → extract with pdf-parse
  3. If no/poor quality → convert to images → Vision API
  4. Combine extracted text with AI for classification/extraction
- **OCR Fallback**: Consider Tesseract.js for offline OCR as budget option
- **Utility Location**: `/src/lib/pdf/`

### File Storage Strategy
- **Storage Location**: Persistent volume at `/data/files/` (same volume as SQLite DB)
- **Structure**: `/data/files/{sociedad_id}/{libro_id}/{acto_id}/`
- **File Types**: PDFs (acts, TAD receipts), generated documents
- **Backup**: Included in database backup strategy
- **Limits**:
  - Max file size: 50MB per PDF
  - Max bulk upload: 100 files per batch
- **Soft Delete**: Files retained when logically deleted

### Hash Calculation (SHA-256)
- **Library**: Node.js native `crypto` module
- **Format**: Hexadecimal string (64 characters)
- **Chain Validation**: Each document's `hash_anterior` must match previous document's `hash_real`
- **First Document**: Uses empty string hash or special marker "PRIMER_DOCUMENTO"
- **Utility Location**: `/src/lib/hash.ts`

### Async Processing Strategy
- **Approach**: Custom implementation with SQLite for MVP
- **Queue Table**: `job_queue` with fields:
  - id, type, payload (JSON), status, attempts, error, created_at, updated_at
- **Job Types**: pdf_process, ai_extraction, hash_calculation, notification
- **States**: pending, processing, completed, failed
- **Worker**: Simple setInterval or cron job checking for pending jobs
- **Retry Logic**: Exponential backoff (2s, 4s, 8s), max 3 attempts
- **Future Migration**: Move to BullMQ + Redis when scaling requires it
- **Utility Location**: `/src/lib/queue/`

### Context Document Strategy
- **Format**: Markdown text stored in database
- **Rendering**: `react-markdown` for frontend display
- **Versioning**: Each update creates new `ContextoVersion` record
- **Diff Display**: Use `jsdiff` library for semantic comparison
- **UI Pattern**: Split view showing previous vs current version
- **Editor**: Simple textarea or basic Markdown editor
- **AI Updates**: AI generates updated Markdown based on new acts
- **Utility Location**: `/src/lib/context/`

### Template Strategy
- **Format**: Markdown with simple placeholders `{{variable}}`
- **Engine**: None - AI performs all substitutions and adaptations
- **AI Responsibilities**:
  - Replace placeholders with values from context/metadata
  - Adapt content for singular/plural (one socio vs multiple)
  - Add/remove sections based on the specific case
  - Adjust language and phrasing contextually
- **Template Purpose**: Serve as "examples/guides" not mechanical templates
- **Flexibility**: AI can deviate from template when context requires it
- **Storage**: Templates stored in database with version history
- **Utility Location**: `/src/lib/templates/`

### PDF Generation Strategy
- **Manipulation (existing PDFs)**: `pdf-lib`
  - Insert hash header page at beginning
  - Merge multiple PDFs
  - Extract pages
- **Generation (new PDFs)**: Puppeteer or `@react-pdf/renderer`
  - Generate acts from AI-produced text
  - Create cover pages, indices
- **Hash Header Content**:
  - Hash anterior declarado
  - Fecha del acto
  - Número de acto
  - Optional: QR code linking to verification
- **TAD Receipt Extraction**:
  - Use `pdf-parse` for text extraction
  - Regex: `/Hash:\s*([a-f0-9]{64})/i`
  - Fallback: Manual input if extraction fails
- **Utility Location**: `/src/lib/pdf/`

### Authentication & Authorization
- **JWT Payload**: Contains user info, List of Sociedades it belongs and the access for each one.
- **Middleware**: `to be determined.`
- **Role Hierarchy**:
  - **Global**: SuperAdmin (access to all sociedades)
  - **Per Sociedad**: Owner > Read-Write (RW) > Read-Only (RO)

### Validation Pattern
```typescript
// Backend validation with Zod
const validation = validateAndSanitize(schema, data);
if (!validation.success) {
  return NextResponse.json({ error: validation.error }, { status: 400 });
}

// Frontend validation with useFormValidation hook
const form = useFormValidation({
  schema: frontendValidationSchema,
  initialData: initialFormData,
  onSubmit: handleSubmit
});
```

## Key Architectural Decisions

Always keep here an updated list of Key Architectural Decisions.
Always check other pages / apis before coding new ones to use the same good practices.

### API Structure
- **RESTful Design**: Standard HTTP methods and status codes
- **Nested Resources**
- **Error Handling**: Consistent error responses with field-specific validation errors
- **Authorization**: Route-level middleware with Sociedad-scoped permissions

### Frontend Patterns
- **Form Management**: `useFormValidation` hook for consistent form handling
- **Real-time Validation**: Field-level validation with immediate feedback
- **Component Library**: Reusable `FormInput`, modals, and layout components
- **State Management**: React Context for auth, local state for forms and data

### Desktop Design Guidelines (CRITICAL)
**🚨 DESKTOP-FIRST APPROACH: Always prioritize desktop experience, then enhance for Mobile**

### Performance Guidelines
- Minimize unnecessary DB queries — prefer batched calls .  
- For frontend: memoize heavy computations with `useMemo`/`useCallback` where relevant.  
- Use pagination or lazy loading for large datasets.  
- Test critical pages with `npm run build && npm start` in production mode — don’t rely only on `npm run dev`.

## Development Guidelines

### API Usage
- **Always check for existing APIs** before creating a new one.  
  - Reuse and extend current endpoints whenever possible.  
  - Avoid duplication of business logic across endpoints

### API Development
- Wrap handlers with appropriate auth middleware
- Use `validateAndSanitize` for all input validation
- Return consistent error structures with appropriate HTTP status codes
- Handle file uploads and CSV imports with proper validation

### Testing and Debugging
- Use the debug logging system (`useDebugLog` hook) for collaborative debugging
- Check browser console and server logs for validation errors
- Test form validation with various input types and edge cases
- Verify database constraints and relationships
- **Test existing flows first** before introducing new ones.  
- Always validate input/output against Zod schemas. If a schema is missing, create one.  
- Ensure unit tests or at least manual test cases exist for new APIs, forms, or database operations.

### Error Handling & Logging
- Always return consistent error objects `{ error: string, details?: object }`.  
- Never swallow errors silently — log them with enough context for debugging.  
- Use structured logging (`console.error({ route, userId, error })`) instead of free-form strings.  
- For client-side: display user-friendly messages, but keep full error detail in logs.

## Common Development Tasks

### Database Schema Changes - Always backup first!

### Form Development
1. Create Zod schema in `frontendValidation.ts`
2. Use `useFormValidation` hook in component
3. Build form with `FormInput` components
4. Handle submission with proper error states
5. Test validation edge cases and backend error handling

## File Structure Notes

- **API Routes**: Follow REST conventions with nested resources
- **Components**: Reusable UI components in `/src/components/`
- **Lib**: Utilities for auth, validation, database, and form handling
- **Contexts**: Global state management (primarily authentication)
- **Scripts**: Database utilities and maintenance scripts

## Date Handling Strategy

**📅 CRITICAL: All date-related development must follow the Institution Timezone Approach.**

See comprehensive documentation: `/docs/DATE_HANDLING_STRATEGY.md`

**Key Rules:**
- Event dates (rounds, matches) are contextual to Argentina timezone
- Use `InstitutionDate` utilities from `/src/lib/institutionDate.ts` 
- Never use raw `new Date()` for event dates
- Always use argentina timezone
- Date comparisons must use proper timezone-aware ranges

**Before implementing ANY date-related features, read the full strategy document.**


## Documentation Guidelines

- **Immediate Documentation Rule**:  
  If you add or modify something important that isn’t documented, update `README.md` or `/docs/` immediately.

1. **Feature Description**: Add to relevant sections (🚀 Características, Architecture Overview, etc.)
2. **Stage Updates**: Mark completed stages as ✅ and add new stages as needed
3. **Changelog**: Add entry in "📋 Changelog Reciente" with version increment
4. **Current Version**: Update "📈 Versión Actual" section
5. **Structure Updates**: Update file structure if new directories/files added
6. **Pending Items**: Update "### **Otros Pendientes**" section to reflect current status

**Purpose**: README.md serves as the architectural overview and maintenance guide. Keep it current to facilitate understanding the application's complete functionality, architecture, and development status for ongoing maintenance and feature additions.

## TypeScript & Code Quality Standards

### Code Consistency
- Follow existing patterns before creating new ones (folder structure, hooks, naming).  
- Prefer composition (small reusable components) over duplication.  
- When adding new utilities, check `/src/lib/` before creating a new helper.  
- Use PascalCase for components, camelCase for functions/variables, SCREAMING_CASE for constants.

### Build Configuration (CRITICAL)
- **No Error Skipping**: `next.config.ts` must NEVER include `ignoreBuildErrors: true` or `ignoreDuringBuilds: true`
- **Strict TypeScript**: All production builds must pass TypeScript compilation without errors
- **ESLint Configuration**: `any` types are warnings (not errors) to allow deployment while maintaining awareness

### TypeScript Best Practices
- **NEVER use `any` type** unless absolutely necessary - use proper interfaces and types
- **Always define return types** for functions, especially API route handlers
- **Use union types** and optional properties instead of `any` for flexible data structures
- **Proper null handling**: Use `!` assertion only when certain, prefer optional chaining `?.`
- **Type array properties** explicitly (e.g., `[] as Array<{...}>`) instead of leaving untyped

### Code Quality Requirements
- **Always test builds locally** with `npm run build` before deployment
- **Fix TypeScript errors immediately** - don't accumulate technical debt
- **Use proper error handling** in API routes with consistent error response structures
- **Validate all inputs** with Zod schemas on both frontend and backend
- **Follow existing patterns** for new API routes and components

### API Route Standards
```typescript
// ✅ GOOD: Properly typed route handler
export async function POST(request: NextRequest) {
  const result = {
    steps: [] as Array<{
      step: number;
      status?: string;
      message?: string;
    }>,
    success: false
  };
  // ... proper error handling
}

// ❌ BAD: Untyped or any types
export async function POST(request: NextRequest) {
  const result: any = {}; // Avoid this!
}
```

### Required Checks Before Deployment
1. **Local Build Test**: Run `npm run build` and ensure it passes
2. **Type Check**: Verify no TypeScript compilation errors
3. **Lint Check**: Run `npm run lint` and address critical issues
4. **Test API Routes**: Ensure all new/modified routes have proper types

## Security Considerations

- JWT tokens include user permissions for each of his Sociedades to avoid privilege escalation
- Input validation on both frontend and backend with Zod
- SQL injection protection
- Password hashing with bcrypt
- Rate limiting should be implemented in production
- Sensitive data (API keys, tokens) properly managed in environment variables
- Never log sensitive information (passwords, JWTs, secrets) — sanitize logs.  
- Always sanitize file uploads and CSV imports before processing.  
- Don’t expose stack traces to clients — catch and wrap errors in API routes.  
- Regularly rotate secrets and document secret usage in `/docs/`.


## Deployment Configuration

To be determined

### Environment Variables (Production)
- `NODE_ENV=production`
- `PORT=3000`
- `DATABASE_URL` should point to persistent volume: `file:/data/dev.db`
- `JWT_SECRET` should be set securely in Fly secrets

### Production Database
- SQLite database persisted in mounted volume at `/data/`
- Migrations run automatically on deployment via release command
- Health checks ensure database connectivity via `/api/health`. It also informs the amount of users. If 0 then the db was deleted.
- \
- **Database location**:
- The development db is located in db/prod.db and is regularly synced with production db located in /data/dev.db
- to use sqlite3 for testing, use /db/dev.db