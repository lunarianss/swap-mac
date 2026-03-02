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

DDD architecture enables comprehensive testing at every layer. Each layer should be tested independently with appropriate mocking strategies.

### Testing Pyramid Overview

```
           /\
          /E2E\         End-to-End Tests (Few)
         /------\
        /  Integ \      Integration Tests (Some)
       /----------\
      / Unit Tests \    Unit Tests (Many)
     /--------------\
```

### Layer-by-Layer Testing Guide

#### 1. Domain Entity Tests (Pure Unit Tests)

**Purpose**: Test business logic in entities
**Dependencies**: NONE
**Mock Strategy**: No mocking needed

**Example (Go)**:
```go
func TestContentType_IsMultiModal(t *testing.T) {
    tests := []struct {
        name string
        ct   ContentType
        want bool
    }{
        {name: "text is not multimodal", ct: ContentTypeText, want: false},
        {name: "image is multimodal", ct: ContentTypeImage, want: true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            if got := tt.ct.IsMultiModal(); got != tt.want {
                t.Errorf("IsMultiModal() = %v, want %v", got, tt.want)
            }
        })
    }
}

func TestTask_Assign(t *testing.T) {
    task := &Task{Status: StatusTodo}
    err := task.Assign("user123")
    assert.NoError(t, err)
    assert.Equal(t, "user123", task.AssigneeID)
}
```

**Example (Java)**:
```java
@Test
public void testTaskAssign() {
    Task task = new Task();
    task.setStatus(TaskStatus.TODO);

    task.assign("user123");

    assertEquals("user123", task.getAssigneeId());
    assertEquals(TaskStatus.IN_PROGRESS, task.getStatus());
}
```

**What to Test**:
- Business methods on entities
- Value object validation
- State transitions
- Aggregate consistency rules

---

#### 2. Domain Service Tests (Unit Tests with Mocks)

**Purpose**: Test business logic orchestration
**Dependencies**: Repository interfaces
**Mock Strategy**: Mock all repository interfaces

**Example (Go with gomock)**:
```go
func TestDatasetServiceImpl_CreateDataset(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    // Create mocks
    mockRepo := mock_repo.NewMockIDatasetAPI(ctrl)
    mockConfig := confmocks.NewMockIConfig(ctrl)

    // Initialize service with mocks
    service := &DatasetServiceImpl{
        repo:          mockRepo,
        storageConfig: mockConfig.GetDatasetItemStorage,
        specConfig:    mockConfig.GetDatasetSpec,
    }

    tests := []struct {
        name     string
        dataset  *entity.Dataset
        fields   []*entity.FieldSchema
        mockSetup func()
        wantErr  bool
    }{
        {
            name: "successful creation",
            dataset: &entity.Dataset{
                Name: "test-dataset",
                SpaceID: 123,
            },
            fields: []*entity.FieldSchema{
                {Name: "input", Key: "input", Status: entity.FieldStatusAvailable},
            },
            mockSetup: func() {
                mockConfig.EXPECT().GetDatasetFeature().Return(&conf.DatasetFeature{
                    Feature: &entity.DatasetFeatures{},
                })
                mockConfig.EXPECT().GetDatasetSpec().Return(&conf.DatasetSpec{
                    Spec: &entity.DatasetSpec{},
                })
                mockRepo.EXPECT().
                    CreateDatasetAndSchema(gomock.Any(), gomock.Any(), gomock.Any()).
                    Return(nil)
            },
            wantErr: false,
        },
        {
            name: "validation failure",
            dataset: &entity.Dataset{Name: ""},
            fields: []*entity.FieldSchema{},
            mockSetup: func() {
                mockConfig.EXPECT().GetDatasetFeature().Return(&conf.DatasetFeature{})
            },
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            tt.mockSetup()
            err := service.CreateDataset(context.Background(), tt.dataset, tt.fields)
            assert.Equal(t, tt.wantErr, err != nil)
        })
    }
}
```

**Example (Java with Mockito)**:
```java
@ExtendWith(MockitoExtension.class)
class TaskServiceTest {
    @Mock
    private ITaskRepository taskRepo;

    @Mock
    private IUserRepository userRepo;

    @InjectMocks
    private TaskServiceImpl taskService;

    @Test
    void testAssignTask_Success() {
        // Arrange
        Long taskId = 1L;
        String userId = "user123";
        Task task = new Task();
        task.setId(taskId);
        task.setStatus(TaskStatus.TODO);

        when(taskRepo.getById(taskId)).thenReturn(task);
        when(userRepo.exists(userId)).thenReturn(true);
        when(taskRepo.update(any(Task.class))).thenReturn(task);

        // Act
        Task result = taskService.assignTask(taskId, userId);

        // Assert
        assertEquals(userId, result.getAssigneeId());
        verify(taskRepo).update(task);
    }
}
```

**What to Test**:
- Business rule validation
- Multi-repository coordination
- Error handling
- Edge cases and boundary conditions

---

#### 3. Repository Implementation Tests (Unit Tests with DAO Mocks)

**Purpose**: Test data access coordination and caching
**Dependencies**: DAOs (MySQL, Redis, etc.)
**Mock Strategy**: Mock all DAO interfaces

**Example (Go)**:
```go
func TestDatasetRepo_SetItemCount(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockRedisDAO := redismocks.NewMockDatasetDAO(ctrl)
    repo := &DatasetRepo{datasetRedisDAO: mockRedisDAO}

    tests := []struct {
        name      string
        datasetID int64
        count     int64
        mockSetup func()
        wantErr   bool
    }{
        {
            name:      "success",
            datasetID: 1,
            count:     100,
            mockSetup: func() {
                mockRedisDAO.EXPECT().
                    SetItemCount(gomock.Any(), int64(1), int64(100)).
                    Return(nil)
            },
            wantErr: false,
        },
        {
            name:      "redis error",
            datasetID: 2,
            count:     50,
            mockSetup: func() {
                mockRedisDAO.EXPECT().
                    SetItemCount(gomock.Any(), int64(2), int64(50)).
                    Return(errors.New("redis error"))
            },
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            tt.mockSetup()
            err := repo.SetItemCount(context.Background(), tt.datasetID, tt.count)
            assert.Equal(t, tt.wantErr, err != nil)
        })
    }
}

func TestDatasetRepo_GetWithCache(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockMySQLDAO := mysqlmocks.NewMockDatasetDAO(ctrl)
    mockRedisDAO := redismocks.NewMockDatasetDAO(ctrl)

    repo := &DatasetRepo{
        datasetMySQLDAO: mockMySQLDAO,
        datasetRedisDAO: mockRedisDAO,
    }

    t.Run("cache hit", func(t *testing.T) {
        expectedDataset := &model.Dataset{ID: 1, Name: "test"}

        // Expect cache check
        mockRedisDAO.EXPECT().
            Get(gomock.Any(), int64(1)).
            Return(expectedDataset, nil)

        // Should NOT query database
        mockMySQLDAO.EXPECT().GetByID(gomock.Any(), gomock.Any()).Times(0)

        result, err := repo.GetDataset(context.Background(), 1)
        assert.NoError(t, err)
        assert.Equal(t, expectedDataset, result)
    })

    t.Run("cache miss", func(t *testing.T) {
        expectedDataset := &model.Dataset{ID: 1, Name: "test"}

        // Cache miss
        mockRedisDAO.EXPECT().
            Get(gomock.Any(), int64(1)).
            Return(nil, redis.Nil)

        // Query database
        mockMySQLDAO.EXPECT().
            GetByID(gomock.Any(), int64(1)).
            Return(expectedDataset, nil)

        // Update cache
        mockRedisDAO.EXPECT().
            Set(gomock.Any(), int64(1), expectedDataset, gomock.Any()).
            Return(nil)

        result, err := repo.GetDataset(context.Background(), 1)
        assert.NoError(t, err)
        assert.Equal(t, expectedDataset, result)
    })
}
```

**What to Test**:
- Cache hit/miss scenarios
- PO ↔ DO conversion
- Transaction handling
- Batch operations
- Error propagation

---

#### 4. DTO Convertor Tests (Pure Unit Tests)

**Purpose**: Test data transformation
**Dependencies**: NONE
**Mock Strategy**: No mocking needed

**Example (Go)**:
```go
func TestPromptDTO2DO(t *testing.T) {
    tests := []struct {
        name    string
        dto     *PromptDTO
        want    *entity.Prompt
        wantErr bool
    }{
        {
            name: "complete conversion",
            dto: &PromptDTO{
                ID:          "123",
                SpaceID:     "456",
                DisplayName: "Test Prompt",
                Description: "Test Description",
            },
            want: &entity.Prompt{
                ID:      123,
                SpaceID: 456,
                PromptBasic: entity.PromptBasic{
                    DisplayName: "Test Prompt",
                    Description: "Test Description",
                },
            },
            wantErr: false,
        },
        {
            name: "missing required field",
            dto: &PromptDTO{
                ID: "123",
                // Missing SpaceID
            },
            want:    nil,
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := PromptDTO2DO(tt.dto)
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
                assert.Equal(t, tt.want, got)
            }
        })
    }
}

func TestPromptDO2DTO_Bidirectional(t *testing.T) {
    original := &entity.Prompt{
        ID:      123,
        SpaceID: 456,
        PromptBasic: entity.PromptBasic{
            DisplayName: "Test",
        },
    }

    // Convert to DTO
    dto := PromptDO2DTO(original)

    // Convert back to DO
    result, err := PromptDTO2DO(dto)

    assert.NoError(t, err)
    assert.Equal(t, original, result)
}
```

**What to Test**:
- Bidirectional conversion accuracy
- Null/nil handling
- Default value assignment
- Validation during conversion

---

#### 5. Application Service Tests (Unit Tests with Domain Mocks)

**Purpose**: Test use case orchestration
**Dependencies**: Domain services, repositories, providers
**Mock Strategy**: Mock domain services and external providers

**Example (Go)**:
```go
func TestPromptManageApp_CreatePrompt(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockPromptService := mock_service.NewMockIPromptService(ctrl)
    mockAuthProvider := mock_provider.NewMockIAuthProvider(ctrl)
    mockAuditProvider := mock_provider.NewMockIAuditProvider(ctrl)

    app := &PromptManageApplicationImpl{
        promptService:  mockPromptService,
        authProvider:   mockAuthProvider,
        auditProvider:  mockAuditProvider,
    }

    tests := []struct {
        name      string
        request   *CreatePromptRequest
        mockSetup func()
        wantErr   bool
    }{
        {
            name: "successful creation",
            request: &CreatePromptRequest{
                SpaceID:     "123",
                DisplayName: "Test Prompt",
            },
            mockSetup: func() {
                // Mock auth check
                mockAuthProvider.EXPECT().
                    CheckSpacePermission(gomock.Any(), "123", "prompt:create").
                    Return(nil)

                // Mock domain service
                mockPromptService.EXPECT().
                    CreatePrompt(gomock.Any(), gomock.Any()).
                    Return(int64(456), nil)

                // Mock audit log
                mockAuditProvider.EXPECT().
                    Log(gomock.Any(), gomock.Any()).
                    Return(nil)
            },
            wantErr: false,
        },
        {
            name: "permission denied",
            request: &CreatePromptRequest{
                SpaceID:     "123",
                DisplayName: "Test",
            },
            mockSetup: func() {
                mockAuthProvider.EXPECT().
                    CheckSpacePermission(gomock.Any(), "123", "prompt:create").
                    Return(errors.New("permission denied"))
            },
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            tt.mockSetup()
            _, err := app.CreatePrompt(context.Background(), tt.request)
            assert.Equal(t, tt.wantErr, err != nil)
        })
    }
}
```

**What to Test**:
- Authentication and authorization
- DTO conversion
- Domain service orchestration
- Audit logging
- Error handling and translation

---

#### 6. API Handler Tests (Unit Tests with Application Mocks)

**Purpose**: Test HTTP request/response handling
**Dependencies**: Application services
**Mock Strategy**: Mock application services

**Example (Go with Hertz)**:
```go
func TestCreatePromptHandler(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockApp := mock_app.NewMockPromptManageService(ctrl)
    handler := &PromptHandler{promptApp: mockApp}

    tests := []struct {
        name           string
        requestBody    string
        mockSetup      func()
        expectedStatus int
    }{
        {
            name: "successful request",
            requestBody: `{
                "space_id": "123",
                "display_name": "Test Prompt"
            }`,
            mockSetup: func() {
                mockApp.EXPECT().
                    CreatePrompt(gomock.Any(), gomock.Any()).
                    Return(&CreatePromptResponse{PromptID: "456"}, nil)
            },
            expectedStatus: 200,
        },
        {
            name:        "invalid JSON",
            requestBody: `{invalid json}`,
            mockSetup:   func() {},
            expectedStatus: 400,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            tt.mockSetup()

            // Create test context
            ctx := &app.RequestContext{}
            ctx.Request.SetBody([]byte(tt.requestBody))

            // Call handler
            handler.HandleCreatePrompt(context.Background(), ctx)

            // Assert response
            assert.Equal(t, tt.expectedStatus, ctx.Response.StatusCode())
        })
    }
}
```

**What to Test**:
- Request parsing and validation
- Response formatting
- HTTP status codes
- Error response format

---

### Integration Tests

Integration tests verify that multiple components work together correctly.

#### Repository Integration Tests

**Purpose**: Test repository with real database
**Strategy**: Use test containers or in-memory databases

**Example (Go with Testcontainers)**:
```go
func TestDatasetRepo_Integration(t *testing.T) {
    // Start MySQL container
    mysqlContainer, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: testcontainers.ContainerRequest{
            Image:        "mysql:8.0",
            ExposedPorts: []string{"3306/tcp"},
            Env: map[string]string{
                "MYSQL_ROOT_PASSWORD": "test",
                "MYSQL_DATABASE":      "testdb",
            },
        },
        Started: true,
    })
    require.NoError(t, err)
    defer mysqlContainer.Terminate(ctx)

    // Get connection string
    host, _ := mysqlContainer.Host(ctx)
    port, _ := mysqlContainer.MappedPort(ctx, "3306")
    dsn := fmt.Sprintf("root:test@tcp(%s:%s)/testdb", host, port.Port())

    // Initialize repository with real DB
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
    require.NoError(t, err)

    repo := NewDatasetRepo(db, nil) // nil for cache in integration test

    // Test actual database operations
    t.Run("create and retrieve dataset", func(t *testing.T) {
        dataset := &entity.Dataset{
            Name:    "test-dataset",
            SpaceID: 123,
        }

        id, err := repo.CreateDataset(ctx, dataset)
        assert.NoError(t, err)
        assert.NotZero(t, id)

        retrieved, err := repo.GetDataset(ctx, id)
        assert.NoError(t, err)
        assert.Equal(t, dataset.Name, retrieved.Name)
    })
}
```

**Example (Java with Testcontainers)**:
```java
@Testcontainers
@SpringBootTest
class TaskRepositoryIntegrationTest {
    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @Autowired
    private TaskRepository taskRepository;

    @Test
    void testCreateAndRetrieveTask() {
        Task task = new Task();
        task.setTitle("Test Task");
        task.setStatus(TaskStatus.TODO);

        Long id = taskRepository.create(task);
        assertNotNull(id);

        Task retrieved = taskRepository.getById(id);
        assertEquals("Test Task", retrieved.getTitle());
    }
}
```

---

### End-to-End Tests

E2E tests verify complete user flows through the entire stack.

**Example (Go)**:
```go
func TestPromptWorkflow_E2E(t *testing.T) {
    // Start test server
    server := setupTestServer(t)
    defer server.Shutdown()

    client := &http.Client{}
    baseURL := server.URL

    t.Run("complete prompt lifecycle", func(t *testing.T) {
        // 1. Create prompt
        createReq := `{
            "space_id": "123",
            "display_name": "E2E Test Prompt",
            "description": "Test description"
        }`
        resp, err := client.Post(baseURL+"/api/v1/prompts", "application/json",
            strings.NewReader(createReq))
        require.NoError(t, err)
        assert.Equal(t, 200, resp.StatusCode)

        var createResp CreatePromptResponse
        json.NewDecoder(resp.Body).Decode(&createResp)
        promptID := createResp.PromptID

        // 2. Get prompt
        resp, err = client.Get(baseURL + "/api/v1/prompts/" + promptID)
        require.NoError(t, err)
        assert.Equal(t, 200, resp.StatusCode)

        // 3. Update prompt
        updateReq := `{"display_name": "Updated Name"}`
        req, _ := http.NewRequest("PUT", baseURL+"/api/v1/prompts/"+promptID,
            strings.NewReader(updateReq))
        resp, err = client.Do(req)
        require.NoError(t, err)
        assert.Equal(t, 200, resp.StatusCode)

        // 4. Delete prompt
        req, _ = http.NewRequest("DELETE", baseURL+"/api/v1/prompts/"+promptID, nil)
        resp, err = client.Do(req)
        require.NoError(t, err)
        assert.Equal(t, 200, resp.StatusCode)
    })
}
```

---

### Testing Tools by Language

#### Go
- **Testing Framework**: Built-in `testing` package
- **Mocking**: `go.uber.org/mock/gomock` (formerly golang/mock)
- **Assertions**: `github.com/stretchr/testify/assert`
- **Test Containers**: `github.com/testcontainers/testcontainers-go`
- **Table-Driven Tests**: Standard Go pattern

**Generate Mocks**:
```bash
# Install mockgen
go install go.uber.org/mock/mockgen@latest

# Generate mocks for interfaces
mockgen -source=domain/repo/prompt_repo.go -destination=domain/repo/mocks/mock_prompt_repo.go
```

#### Java
- **Testing Framework**: JUnit 5
- **Mocking**: Mockito
- **Assertions**: AssertJ
- **Test Containers**: Testcontainers
- **Spring Test**: `@SpringBootTest`, `@WebMvcTest`

#### Python
- **Testing Framework**: pytest
- **Mocking**: unittest.mock or pytest-mock
- **Assertions**: Built-in assert
- **Test Containers**: testcontainers-python
- **Fixtures**: pytest fixtures

#### TypeScript
- **Testing Framework**: Jest or Vitest
- **Mocking**: Jest mocks or Vitest mocks
- **Assertions**: expect (Jest/Vitest)
- **Test Containers**: testcontainers-node
- **DI Testing**: Use framework-specific testing utilities

---

### Test Coverage Guidelines

| Layer | Target Coverage | Priority |
|-------|----------------|----------|
| Domain Entities | 90%+ | High |
| Domain Services | 85%+ | High |
| Repositories | 80%+ | Medium |
| Convertors | 95%+ | High |
| Application Services | 80%+ | Medium |
| API Handlers | 70%+ | Medium |

---

### Best Practices

#### 1. Test Naming Conventions
- **Go**: `TestFunctionName_Scenario`
- **Java**: `testFunctionName_Scenario`
- **Python**: `test_function_name_scenario`

#### 2. Arrange-Act-Assert Pattern
```
// Arrange: Set up test data and mocks
mockRepo.EXPECT().GetByID(1).Return(entity, nil)

// Act: Execute the function under test
result, err := service.GetEntity(ctx, 1)

// Assert: Verify the results
assert.NoError(t, err)
assert.Equal(t, expected, result)
```

#### 3. Table-Driven Tests
Use table-driven tests for multiple scenarios:
```go
tests := []struct {
    name    string
    input   Input
    want    Output
    wantErr bool
}{
    {name: "scenario 1", input: ..., want: ..., wantErr: false},
    {name: "scenario 2", input: ..., want: ..., wantErr: true},
}
```

#### 4. Test Isolation
- Each test should be independent
- Use `t.Run()` for subtests
- Clean up resources in `defer` or `t.Cleanup()`

#### 5. Mock Verification
- Verify that mocks were called as expected
- Use `gomock.Any()` for flexible matching
- Use `Times(n)` to verify call count

#### 6. Test Data Builders
Create helper functions for test data:
```go
func newTestDataset(opts ...func(*entity.Dataset)) *entity.Dataset {
    ds := &entity.Dataset{
        Name:    "default-name",
        SpaceID: 123,
    }
    for _, opt := range opts {
        opt(ds)
    }
    return ds
}

// Usage
dataset := newTestDataset(
    func(d *entity.Dataset) { d.Name = "custom-name" },
)
```

---

### Common Testing Patterns

#### Testing Error Cases
```go
t.Run("repository error", func(t *testing.T) {
    mockRepo.EXPECT().
        GetByID(gomock.Any(), int64(1)).
        Return(nil, errors.New("database error"))

    _, err := service.GetEntity(ctx, 1)
    assert.Error(t, err)
    assert.Contains(t, err.Error(), "database error")
})
```

#### Testing Transactions
```go
t.Run("transaction rollback on error", func(t *testing.T) {
    mockDB.EXPECT().Begin().Return(mockTx, nil)
    mockTx.EXPECT().Rollback().Return(nil)

    mockRepo.EXPECT().
        Create(gomock.Any(), gomock.Any()).
        Return(errors.New("constraint violation"))

    err := service.CreateWithTransaction(ctx, entity)
    assert.Error(t, err)
})
```

#### Testing Caching
```go
t.Run("cache invalidation on update", func(t *testing.T) {
    // Update database
    mockMySQLDAO.EXPECT().
        Update(gomock.Any(), gomock.Any()).
        Return(nil)

    // Invalidate cache
    mockRedisDAO.EXPECT().
        Delete(gomock.Any(), int64(1)).
        Return(nil)

    err := repo.UpdateEntity(ctx, 1, updates)
    assert.NoError(t, err)
})
```

---

### Running Tests

#### Go
```bash
# Run all tests
go test ./...

# Run with verbose output
go test -v ./...

# Run specific package
go test -v ./modules/prompt/domain/service/...

# Run specific test
go test -v ./modules/prompt/... -run TestPromptService_Create

# Run with coverage
go test -cover ./...
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out

# Run with race detection
go test -race ./...

# Run integration tests only (with build tags)
go test -tags=integration ./...
```

#### Java
```bash
# Maven
mvn test
mvn test -Dtest=TaskServiceTest
mvn verify  # includes integration tests

# Gradle
./gradlew test
./gradlew test --tests TaskServiceTest
./gradlew integrationTest
```

#### Python
```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=src --cov-report=html

# Run specific test
pytest tests/test_task_service.py::test_create_task

# Run with markers
pytest -m unit
pytest -m integration
```

---

### Continuous Integration

**Example GitHub Actions Workflow**:
```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: test
          MYSQL_DATABASE: testdb
        ports:
          - 3306:3306

      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v3

      - name: Set up Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.21'

      - name: Run unit tests
        run: go test -v -race -coverprofile=coverage.out ./...

      - name: Run integration tests
        run: go test -v -tags=integration ./...
        env:
          MYSQL_DSN: root:test@tcp(localhost:3306)/testdb
          REDIS_ADDR: localhost:6379

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.out
```

---

### Summary: Testing Checklist

- [ ] Domain entities have unit tests for all business methods
- [ ] Domain services use mocked repositories
- [ ] Repository implementations use mocked DAOs
- [ ] Convertors have bidirectional conversion tests
- [ ] Application services use mocked domain services
- [ ] API handlers use mocked application services
- [ ] Integration tests use test containers
- [ ] E2E tests cover critical user flows
- [ ] Test coverage meets targets (80%+ overall)
- [ ] CI/CD pipeline runs all tests automatically
- [ ] Mocks are generated and up-to-date
- [ ] Test data builders simplify test setup

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
