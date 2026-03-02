---
name: ddd-architecture
description: This skill should be used when designing or implementing Domain-Driven Design (DDD) architecture for backend systems. It applies when creating new modules, refactoring existing code to DDD patterns, or building layered architectures with proper separation of concerns. Triggers on requests like "create a DDD module", "implement domain-driven design", "design backend architecture", or when working with any backend language (Go, Java, Python, TypeScript, C#) requiring clean, maintainable code structure.
---

# DDD Architecture Skill

A comprehensive, language-agnostic Domain-Driven Design (DDD) specification for building modular backend systems. This skill can be applied to any backend language (Go, Java, Python, TypeScript, etc.) to generate clean, maintainable code following DDD principles.

## Overview

This skill guides you through implementing a complete DDD module with proper layering, dependency injection, and separation of concerns. It's based on proven patterns from production systems and can be adapted to any programming language.

## Architecture Layers

```
module/
├── domain/           # Domain Layer (Core Business Logic)
│   ├── entity/       # Domain Entities (Aggregates, Value Objects)
│   ├── repo/         # Repository Interfaces
│   └── service/      # Domain Services
├── infra/            # Infrastructure Layer (External Dependencies)
│   ├── repo/         # Repository Implementations
│   │   ├── database/ # Database DAOs and Models
│   │   └── cache/    # Cache DAOs
│   ├── rpc/          # External Service Clients
│   └── conf/         # Configuration Providers
├── application/      # Application Layer (Use Cases)
│   ├── convertor/    # DTO ↔ Entity Converters
│   └── service.{ext} # Application Services
├── pkg/              # Module-Specific Utilities
│   └── errno/        # Error Definitions
└── api/              # API Layer (Entry Points)
    ├── handler/      # Request Handlers
    └── router/       # Route Registration
```

## Core Design Principles

### 1. Dependency Direction
- **Rule**: Outer layers depend on inner layers, never the reverse
- **Domain layer** has ZERO external dependencies
- **Infrastructure** implements interfaces defined in domain
- **Application** orchestrates domain services
- **API** depends on application layer

### 2. Interface Segregation
- Domain defines interfaces, infrastructure implements them
- Use dependency injection to wire implementations
- Enables testing with mocks and easy swapping of implementations

### 3. Separation of Concerns
- Each layer has a single, well-defined responsibility
- No business logic in infrastructure or API layers
- No database/framework code in domain layer

### 4. Data Flow

```
HTTP Request
    ↓
[API Handler] Parse request, call Application Service
    ↓
[Application Service] Auth, DTO→Entity, orchestrate Domain Services
    ↓
[Domain Service] Execute core business logic
    ↓
[Repository Interface] Domain-defined persistence contract
    ↓
[Repository Implementation] Coordinate DAOs + Cache
    ↓
[DAO] Actual database/cache operations
```

---

## Implementation Guide

### Step 1: Define Domain Entities

**Purpose**: Core business objects with behavior
**Location**: `domain/entity/`
**Dependencies**: NONE (pure language constructs)

**Key Characteristics**:
- Aggregates and value objects
- Business methods attached to entities
- No framework annotations (ORM, JSON, etc.)
- Immutable where possible
- Rich domain models, not anemic data structures

**Example Structure** (Language-agnostic):
```
Entity: Prompt
  - ID: identifier
  - SpaceID: identifier
  - PromptKey: string
  - PromptBasic: nested entity
    - DisplayName: string
    - Description: string
    - CreatedBy: string
    - UpdatedBy: string
  - PromptDraft: nested entity (optional)
    - PromptDetail: nested entity
    - DraftInfo: metadata
  - PromptCommit: nested entity (optional)
    - PromptDetail: nested entity
    - CommitInfo: metadata

Methods on Entity:
  - GetVersion(): string
  - GetPromptDetail(): PromptDetail
  - FormatMessages(messages, variables): formatted messages
  - Clone(): Prompt
```

**Guidelines**:
- Use composition over inheritance
- Entities can contain other entities (aggregates)
- Add business logic methods directly on entities
- Keep entities focused on domain concepts

---

### Step 2: Define Repository Interfaces

**Purpose**: Define persistence contracts without implementation details
**Location**: `domain/repo/`
**Dependencies**: Domain entities only

**Key Characteristics**:
- Interface methods use domain entities (NOT database models)
- Query parameters encapsulated in parameter objects
- Return domain entities
- Support for options (caching, transactions)
- Generate mocks for testing

**Example Structure**:
```
Interface: IManageRepo
  Methods:
    - CreatePrompt(ctx, prompt: Prompt): promptID
    - GetPrompt(ctx, param: GetPromptParam): Prompt
    - ListPrompt(ctx, param: ListPromptParam): ListPromptResult
    - UpdatePrompt(ctx, param: UpdatePromptParam): error
    - DeletePrompt(ctx, promptID): error
    - SaveDraft(ctx, prompt: Prompt): DraftInfo
    - CommitDraft(ctx, param: CommitDraftParam): error

Parameter Objects:
  GetPromptParam:
    - PromptID: identifier
    - WithCommit: boolean
    - CommitVersion: string (optional)
    - WithDraft: boolean
    - UserID: string (optional)
    - ExpandSnippet: boolean

  ListPromptParam:
    - SpaceID: identifier
    - Keyword: string (optional)
    - CreatedBy: list of strings (optional)
    - PageNum: integer
    - PageSize: integer
    - OrderBy: enum
    - Ascending: boolean

Option Functions:
  - WithCacheEnabled(): option
  - WithTransaction(tx): option
```

**Guidelines**:
- Use parameter objects instead of long argument lists
- Optional parameters via option pattern or builder
- Return rich result objects, not primitives
- Include pagination metadata in list results

---

### Step 3: Define Domain Services

**Purpose**: Business logic that doesn't belong to a single entity
**Location**: `domain/service/`
**Dependencies**: Domain entities, repository interfaces

**Key Characteristics**:
- Interface + implementation in same package
- Constructor injection of dependencies (interfaces only)
- Stateless (dependencies injected, no mutable state)
- Pure business logic, no I/O operations
- Orchestrates multiple entities and repositories

**Example Structure**:
```
Interface: IPromptService
  Methods:
    - CreatePrompt(ctx, prompt: Prompt): promptID
    - GetPrompt(ctx, param: GetPromptParam): Prompt
    - SaveDraft(ctx, prompt: Prompt): DraftInfo
    - FormatPrompt(ctx, prompt, messages, variables): formatted messages
    - ExpandSnippets(ctx, prompt: Prompt): error
    - ValidateLabelsExist(ctx, spaceID, labelKeys): error
    - ExecuteStreaming(ctx, param: ExecuteParam): Reply

Implementation: PromptServiceImpl
  Dependencies (injected):
    - formatter: IPromptFormatter
    - idGenerator: IIDGenerator
    - manageRepo: IManageRepo
    - labelRepo: ILabelRepo
    - configProvider: IConfigProvider
    - llmProvider: ILLMProvider
    - fileProvider: IFileProvider

  Constructor:
    NewPromptService(
      formatter,
      idGenerator,
      manageRepo,
      labelRepo,
      configProvider,
      llmProvider,
      fileProvider
    ): IPromptService
```

**Guidelines**:
- Accept interfaces, return concrete types or interfaces
- Use constructor injection for all dependencies
- Keep methods focused on single business operations
- Validate business rules here
- Coordinate between multiple repositories

---

### Step 4: Implement Database Models and DAOs

**Purpose**: Database-specific data structures and CRUD operations
**Location**: `infra/repo/database/`
**Dependencies**: Database framework (ORM, query builder)

**Key Characteristics**:
- Models are database table mappings (with ORM annotations)
- Separate from domain entities
- DAOs handle CRUD operations
- DAO interfaces for testability
- Support transaction passing

**Example Structure**:
```
Model: PromptBasicPO (Persistent Object)
  - ID: identifier (primary key)
  - SpaceID: identifier (indexed)
  - PromptKey: string (unique with SpaceID)
  - DisplayName: string
  - Description: string
  - PromptType: enum
  - CreatedBy: string
  - UpdatedBy: string
  - CreatedAt: timestamp
  - UpdatedAt: timestamp
  - DeletedAt: timestamp (soft delete)

  Table: "prompt_basic"
  Indexes: [space_id, prompt_key], [created_at]

Interface: IPromptBasicDAO
  Methods:
    - Create(ctx, model: PromptBasicPO, options): error
    - GetByID(ctx, id: identifier, options): PromptBasicPO
    - GetByPromptKey(ctx, spaceID, promptKey, options): PromptBasicPO
    - List(ctx, param: ListParam, options): list of PromptBasicPO
    - Update(ctx, id, updates: map, options): error
    - Delete(ctx, id, options): error
    - BatchGetByIDs(ctx, ids: list, options): list of PromptBasicPO

Options:
  - WithTransaction(tx): pass transaction context
  - WithLock(): use row-level locking
```

**Guidelines**:
- Models use framework-specific annotations
- Keep DAO methods simple (single table operations)
- Support transaction passing via options
- Use batch operations for efficiency
- Implement soft deletes where appropriate

---

### Step 5: Implement Cache DAOs

**Purpose**: Caching layer for frequently accessed data
**Location**: `infra/repo/cache/`
**Dependencies**: Cache client (Redis, Memcached)

**Key Characteristics**:
- Key generation strategies
- TTL management
- Serialization/deserialization
- Cache invalidation methods
- Separate interface for testability

**Example Structure**:
```
Interface: IPromptBasicCacheDAO
  Methods:
    - Get(ctx, promptID): PromptBasicPO
    - Set(ctx, promptID, model: PromptBasicPO, ttl): error
    - Delete(ctx, promptID): error
    - BatchGet(ctx, promptIDs: list): map of PromptBasicPO
    - BatchDelete(ctx, promptIDs: list): error

Implementation:
  - Key pattern: "prompt:basic:{promptID}"
  - TTL: configurable (e.g., 1 hour)
  - Serialization: JSON or MessagePack
  - Null value caching: prevent cache penetration
```

**Guidelines**:
- Use consistent key naming conventions
- Set appropriate TTLs based on data volatility
- Handle cache misses gracefully
- Implement batch operations
- Consider cache warming strategies

---

### Step 6: Implement Repository Layer

**Purpose**: Bridge between domain interfaces and infrastructure DAOs
**Location**: `infra/repo/`
**Dependencies**: Domain repo interfaces, DAOs, convertors

**Key Characteristics**:
- Implements domain repository interfaces
- Coordinates multiple DAOs (database + cache)
- Handles model ↔ entity conversion
- Implements caching strategies (Cache-Aside, Write-Through)
- Manages transactions

**Example Structure**:
```
Implementation: ManageRepoImpl implements IManageRepo
  Dependencies (injected):
    - db: DatabaseProvider
    - idGenerator: IIDGenerator
    - promptBasicDAO: IPromptBasicDAO
    - promptCommitDAO: IPromptCommitDAO
    - promptDraftDAO: IPromptDraftDAO
    - promptBasicCacheDAO: IPromptBasicCacheDAO
    - promptCacheDAO: IPromptCacheDAO

  Method: GetPrompt(ctx, param: GetPromptParam): Prompt
    1. Check cache if enabled
    2. If cache miss, query database
    3. Convert PO → DO (model to entity)
    4. Update cache
    5. Return domain entity

  Method: CreatePrompt(ctx, prompt: Prompt): promptID
    1. Generate ID
    2. Start transaction
    3. Convert DO → PO (entity to model)
    4. Call DAO.Create()
    5. Handle related entities (draft, relations)
    6. Commit transaction
    7. Invalidate related caches
    8. Return ID

  Caching Strategy: Cache-Aside
    - Read: Check cache → DB → Update cache
    - Write: Update DB → Invalidate cache
    - Batch operations: Minimize cache round-trips
```

**Guidelines**:
- Use convertor functions for PO ↔ DO transformation
- Implement proper transaction boundaries
- Cache invalidation on writes
- Handle partial failures gracefully
- Log cache hit/miss metrics

---

### Step 7: Implement DTO Convertors

**Purpose**: Transform between API DTOs and domain entities
**Location**: `application/convertor/`
**Dependencies**: Domain entities, API DTOs

**Key Characteristics**:
- Pure functions (no side effects)
- Bidirectional conversion (DTO → DO, DO → DTO)
- Handle optional fields
- Validation during conversion
- Centralized conversion logic

**Example Structure**:
```
Functions:
  PromptDTO2DO(dto: PromptDTO): Prompt
    - Map DTO fields to entity fields
    - Handle nested objects
    - Set defaults for missing fields
    - Validate required fields

  PromptDO2DTO(entity: Prompt): PromptDTO
    - Map entity fields to DTO fields
    - Format timestamps
    - Handle optional nested objects
    - Omit internal fields

  PromptDetailDTO2DO(dto: PromptDetailDTO): PromptDetail
  PromptDetailDO2DTO(entity: PromptDetail): PromptDetailDTO

  MessageDTO2DO(dto: MessageDTO): Message
  MessageDO2DTO(entity: Message): MessageDTO
```

**Guidelines**:
- Keep conversion logic simple and obvious
- Don't include business logic in convertors
- Handle null/nil values explicitly
- Use helper functions for repeated patterns
- Consider using code generation for boilerplate

---

### Step 8: Implement Application Services

**Purpose**: Orchestrate use cases, handle cross-cutting concerns
**Location**: `application/`
**Dependencies**: Domain services, repositories, RPC providers, convertors

**Key Characteristics**:
- One method per use case
- Handles authentication and authorization
- DTO ↔ Entity conversion
- Orchestrates multiple domain services
- Transaction boundaries
- Audit logging
- Error handling and translation

**Example Structure**:
```
Implementation: PromptManageApplicationImpl
  Dependencies (injected):
    - manageRepo: IManageRepo
    - labelRepo: ILabelRepo
    - promptService: IPromptService
    - authProvider: IAuthProvider
    - userProvider: IUserProvider
    - auditProvider: IAuditProvider
    - configProvider: IConfigProvider

  Method: CreatePrompt(ctx, request: CreatePromptRequest): CreatePromptResponse
    1. Extract user from context
    2. Check permissions (authProvider.CheckSpacePermission)
    3. Validate request parameters
    4. Convert DTO → DO (convertor.PromptDTO2DO)
    5. Call domain service (promptService.CreatePrompt)
    6. Audit log (auditProvider.Log)
    7. Convert DO → DTO (convertor.PromptDO2DTO)
    8. Return response

  Method: GetPrompt(ctx, request: GetPromptRequest): GetPromptResponse
    1. Extract user from context
    2. Check permissions
    3. Build domain query param
    4. Call domain service (promptService.GetPrompt)
    5. Expand snippets if requested
    6. Convert DO → DTO
    7. Return response

  Standard Flow:
    Auth → Validate → DTO→DO → Domain Logic → DO→DTO → Response
```

**Guidelines**:
- Keep application services thin (orchestration only)
- No business logic here (delegate to domain services)
- Handle all cross-cutting concerns (auth, audit, metrics)
- Use consistent error handling patterns
- Transaction boundaries at this layer

---

### Step 9: Configure Dependency Injection

**Purpose**: Wire all dependencies at compile-time or startup
**Location**: `application/wire.go` or `application/di.{ext}`
**Dependencies**: All layers

**Key Characteristics**:
- Compile-time DI (Wire, Dagger) or runtime DI (Spring, Guice)
- Provider functions for each component
- Dependency sets for modularity
- Init functions for each application service
- Minimal manual wiring

**Example Structure (Wire-style)**:
```
Provider Sets:
  domainSet = [
    NewPromptFormatter,
    NewPromptService,
    NewManageRepo,
    NewLabelRepo,
    NewPromptBasicDAO,
    NewPromptCommitDAO,
    NewPromptDraftDAO,
    NewPromptBasicCacheDAO,
    NewPromptCacheDAO,
    NewConfigProvider,
    NewLLMProvider,
    NewAuthProvider,
    NewFileProvider,
    NewSnippetParser
  ]

  manageSet = [
    NewPromptManageApplication,
    domainSet
  ]

Init Function:
  InitPromptManageApplication(
    idGenerator: IIDGenerator,
    db: DatabaseProvider,
    cache: CacheClient,
    meter: Meter,
    configFactory: IConfigFactory,
    llmClient: LLMClient,
    authClient: AuthClient,
    fileClient: FileClient,
    auditClient: AuditClient
  ): PromptManageService

  Wire analyzes:
    - NewPromptManageApplication needs what?
    - Recursively find providers for each dependency
    - Generate construction code
```

**Guidelines**:
- Group related providers into sets
- Use compile-time DI when possible (catches errors early)
- Provide all infrastructure dependencies from main
- Keep provider functions simple (just construction)
- Use interfaces for all injected dependencies

---

### Step 10: Implement API Handlers

**Purpose**: HTTP request entry points
**Location**: `api/handler/`
**Dependencies**: Application services, request/response types

**Key Characteristics**:
- Parse and validate HTTP requests
- Call application service
- Format HTTP responses
- Handle HTTP-specific concerns (status codes, headers)
- Minimal logic (delegate to application layer)

**Example Structure**:
```
Handler: CreatePromptHandler
  Dependencies:
    - promptManageApp: PromptManageService

  Function: HandleCreatePrompt(ctx, httpRequest)
    1. Parse request body → CreatePromptRequest
    2. Validate request (basic validation)
    3. Call application service
       response = promptManageApp.CreatePrompt(ctx, request)
    4. Handle errors → HTTP status codes
    5. Format response → JSON
    6. Return HTTP response

Handler: GetPromptHandler
  Function: HandleGetPrompt(ctx, httpRequest)
    1. Extract path parameters (promptID)
    2. Extract query parameters (version, withDraft)
    3. Build GetPromptRequest
    4. Call application service
    5. Return response

Error Handling:
  - Business errors → 400 Bad Request
  - Not found → 404 Not Found
  - Permission denied → 403 Forbidden
  - Internal errors → 500 Internal Server Error
```

**Guidelines**:
- Keep handlers thin (parsing + calling + formatting)
- Use middleware for cross-cutting concerns (logging, auth)
- Consistent error response format
- Validate request structure, not business rules
- Use framework-specific patterns

---

### Step 11: Register Routes

**Purpose**: Map HTTP routes to handlers
**Location**: `api/router/`
**Dependencies**: Handlers, middleware

**Example Structure**:
```
Router Registration:
  RegisterPromptRoutes(router, promptManageApp, middleware)
    Group: /api/v1/prompts
      Middleware: [Auth, RateLimit, Logging]

      POST   /                    → CreatePromptHandler
      GET    /:promptID           → GetPromptHandler
      PUT    /:promptID           → UpdatePromptHandler
      DELETE /:promptID           → DeletePromptHandler
      GET    /                    → ListPromptsHandler
      POST   /:promptID/draft     → SaveDraftHandler
      POST   /:promptID/commit    → CommitDraftHandler
      GET    /:promptID/commits   → ListCommitsHandler
      GET    /:promptID/labels    → GetLabelsHandler
      PUT    /:promptID/labels    → UpdateLabelsHandler
```

**Guidelines**:
- RESTful route design
- Consistent URL patterns
- Apply middleware at appropriate levels
- Version your APIs
- Document routes (OpenAPI/Swagger)

---

## Language-Specific Adaptations

### Go
- Use interfaces extensively
- Wire for dependency injection
- GORM for ORM
- Hertz/Gin for HTTP
- Context for request scoping

### Java
- Use Spring Framework
- Spring Boot for DI
- JPA/Hibernate for ORM
- Spring MVC for HTTP
- Lombok for boilerplate reduction

### Python
- Use Protocol/ABC for interfaces
- Dependency Injector or manual DI
- SQLAlchemy for ORM
- FastAPI/Flask for HTTP
- Pydantic for validation

### TypeScript
- Use interfaces and abstract classes
- InversifyJS or TSyringe for DI
- TypeORM or Prisma for ORM
- Express/NestJS for HTTP
- Class-validator for validation

### C#
- Use interfaces
- Built-in DI container
- Entity Framework for ORM
- ASP.NET Core for HTTP
- FluentValidation for validation

---

## Testing Strategy

### Unit Tests
- **Domain entities**: Test business methods
- **Domain services**: Mock repository interfaces
- **Convertors**: Test bidirectional conversion
- **Repository implementations**: Mock DAOs

### Integration Tests
- **Repository layer**: Test with real database (test containers)
- **Application services**: Test with real domain services
- **API handlers**: Test with mock application services

### End-to-End Tests
- Full stack tests with real dependencies
- Test complete user flows
- Use test databases and external service mocks

---

## Common Patterns

### Transaction Management
```
Application Layer:
  StartTransaction()
  try:
    Call Domain Service 1
    Call Domain Service 2
    Commit()
  catch:
    Rollback()
```

### Caching Strategy (Cache-Aside)
```
Repository Layer:
  Read:
    1. Check cache
    2. If miss, query database
    3. Update cache
    4. Return result

  Write:
    1. Update database
    2. Invalidate cache
```

### Error Handling
```
Domain Layer:
  - Define domain-specific errors
  - Return errors, don't throw (Go-style) or use exceptions

Application Layer:
  - Catch domain errors
  - Translate to application errors
  - Add context

API Layer:
  - Translate to HTTP status codes
  - Format error responses
```

### Pagination
```
Request:
  - PageNum/PageToken
  - PageSize
  - OrderBy
  - Ascending

Response:
  - Items: list
  - Total: count
  - NextPageToken: string (optional)
```

---

## Best Practices

### Domain Layer
- ✅ Pure business logic
- ✅ No external dependencies
- ✅ Rich domain models
- ✅ Testable in isolation
- ❌ No database code
- ❌ No HTTP code
- ❌ No framework dependencies

### Infrastructure Layer
- ✅ Implement domain interfaces
- ✅ Handle technical concerns
- ✅ Optimize for performance
- ✅ Proper error handling
- ❌ No business logic
- ❌ Don't leak implementation details

### Application Layer
- ✅ Orchestrate use cases
- ✅ Handle cross-cutting concerns
- ✅ Transaction boundaries
- ✅ DTO conversion
- ❌ No business logic
- ❌ No direct database access

### API Layer
- ✅ Parse requests
- ✅ Format responses
- ✅ HTTP-specific concerns
- ❌ No business logic
- ❌ No direct repository access

---

## Checklist for New Module

- [ ] Define domain entities with business methods
- [ ] Define repository interfaces (use domain entities)
- [ ] Define domain service interfaces
- [ ] Implement domain services (use repository interfaces)
- [ ] Create database models (separate from entities)
- [ ] Implement DAOs (CRUD operations)
- [ ] Implement cache DAOs (if needed)
- [ ] Implement repository layer (coordinate DAOs, handle caching)
- [ ] Create DTO convertors (bidirectional)
- [ ] Implement application services (orchestration)
- [ ] Configure dependency injection
- [ ] Implement API handlers
- [ ] Register routes
- [ ] Write unit tests (domain + services)
- [ ] Write integration tests (repositories)
- [ ] Write API tests (handlers)
- [ ] Document APIs (OpenAPI/Swagger)

---

## Example: Creating a New "Task" Module

### 1. Domain Entity
```
Task:
  - ID
  - ProjectID
  - Title
  - Description
  - Status (enum: TODO, IN_PROGRESS, DONE)
  - AssigneeID
  - CreatedAt
  - UpdatedAt

Methods:
  - Assign(userID)
  - MarkInProgress()
  - MarkDone()
  - IsOverdue(): boolean
```

### 2. Repository Interface
```
ITaskRepo:
  - CreateTask(ctx, task): taskID
  - GetTask(ctx, taskID): Task
  - ListTasks(ctx, param): ListTaskResult
  - UpdateTask(ctx, taskID, updates): error
  - DeleteTask(ctx, taskID): error
```

### 3. Domain Service
```
ITaskService:
  - CreateTask(ctx, task): taskID
  - AssignTask(ctx, taskID, userID): error
  - TransitionStatus(ctx, taskID, newStatus): error
  - GetTasksForUser(ctx, userID): list of Task

Implementation:
  - Validates status transitions
  - Checks user permissions
  - Sends notifications
```

### 4. Infrastructure
```
TaskPO (database model):
  - id, project_id, title, description
  - status, assignee_id
  - created_at, updated_at

TaskDAO:
  - Create, GetByID, List, Update, Delete

TaskCacheDAO:
  - Get, Set, Delete (cache by task ID)

TaskRepoImpl:
  - Implements ITaskRepo
  - Uses TaskDAO + TaskCacheDAO
  - Converts TaskPO ↔ Task
```

### 5. Application Layer
```
TaskApplicationImpl:
  - CreateTask(request): response
    1. Auth check
    2. DTO → DO
    3. Call taskService.CreateTask
    4. DO → DTO
    5. Return response

  - AssignTask(request): response
    1. Auth check
    2. Call taskService.AssignTask
    3. Return response
```

### 6. API Layer
```
POST   /api/v1/tasks              → CreateTaskHandler
GET    /api/v1/tasks/:taskID      → GetTaskHandler
PUT    /api/v1/tasks/:taskID      → UpdateTaskHandler
DELETE /api/v1/tasks/:taskID      → DeleteTaskHandler
GET    /api/v1/tasks              → ListTasksHandler
POST   /api/v1/tasks/:taskID/assign → AssignTaskHandler
```

---

## Conclusion

This DDD architecture provides:
- **Clear separation of concerns**: Each layer has a single responsibility
- **Testability**: Easy to mock and test each layer independently
- **Maintainability**: Changes are localized to specific layers
- **Scalability**: Can scale different layers independently
- **Flexibility**: Easy to swap implementations (database, cache, external services)
- **Language-agnostic**: Core principles apply to any backend language

Follow this guide to build robust, maintainable backend modules that stand the test of time.
