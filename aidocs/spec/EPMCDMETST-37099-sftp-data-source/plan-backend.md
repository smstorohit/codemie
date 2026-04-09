# Implementation Plan: Add SFTP Data Source to CodeMie Platform (Backend)

## Overview
This plan covers the backend implementation for adding SFTP (Secure File Transfer Protocol) data source support to the CodeMie platform. Users will be able to configure SFTP connections, automatically poll remote directories every 15 minutes, and index CSV files for use with CodeMie assistants.

**Repository**: codemie (https://github.com/smstorohit/codemie)

## User Story
**As a** CodeMie administrator,  
**I want to** configure an SFTP data source in the CodeMie setup wizard,  
**so that** I can securely pull and index CSV files from legacy systems for use with CodeMie assistants.

### Acceptance Criteria
1. SFTP connection manager with username/password authentication
2. Credential storage in CodeMie's encrypted vault (AWS KMS/Azure Key Vault/GCP KMS)
3. Fixed 15-minute polling interval for automatic file retrieval
4. CSV-only file support with existing indexing pipeline integration
5. Test connection validation endpoint
6. REST API endpoints for SFTP data source CRUD operations
7. Audit logging for all SFTP transfer activities
8. Cron scheduling support for automatic reindexing

### Assumptions
- SFTP servers support standard SSH key exchange and password authentication (no MFA or key-based auth in v1)
- CSV files follow standard formatting (comma-delimited, with optional headers)
- SFTP servers are accessible from CodeMie's deployment environment
- Polling interval is fixed at 15 minutes (not user-configurable in v1)
- File deduplication handled by checking file names and timestamps
- Existing CodeMie credential vault and RBAC sufficient for SFTP credential storage

## Research Findings

### Existing Codebase Patterns
- **Similar Features**: SharePointDatasourceProcessor, ConfluenceDatasourceProcessor, JiraDatasourceProcessor
- **Reusable Components**:
  - `BaseDatasourceProcessor` - Abstract base class for all datasources
  - `BaseDatasourceLoader` - Abstract loader for fetching remote data
  - `EncryptionFactory` - Multi-cloud KMS credential encryption
  - `SchedulerSettingsService` - Cron-based reindexing scheduler
  - FastAPI router patterns in `src/codemie/rest_api/routers/index.py`
- **Established Patterns**:
  - Datasource lifecycle: `_init_index()`, `_init_loader()`, `process()`, `_cleanup_data()`
  - Connection validation: `check_*_connection()` classmethod pattern
  - Polling support: `cron_expression` parameter with scheduler integration

### CodeMie User Guide References
- **Relevant Documentation**: Data source configuration patterns, credential vault storage, project-level integrations
- **User Flows**: Setup wizard → Configure credentials → Test connection → Save → Auto-indexing
- **Platform Conventions**: Project-scoped datasources, RBAC credential access, 15-minute minimum polling intervals

## Technical Context
- **Repository**: codemie
- **Tech Stack**: Python 3.11+, FastAPI, SQLModel, Elasticsearch, Paramiko (SSH/SFTP client)
- **Dependencies**: 
  - `paramiko` - Python SSH/SFTP client library
  - Existing: `langchain`, `elasticsearch`, `pydantic`, `sqlmodel`
- **Integrations**: 
  - CodeMie credential vault (KMS integration)
  - Elasticsearch (indexing pipeline)
  - Scheduler service (cron-based reindexing)

## API Contracts

```yaml
openapi: 3.0.3
info:
  title: SFTP Data Source API
  version: 1.0.0
  description: API endpoints for managing SFTP data sources in CodeMie

paths:
  /v1/index/knowledge_base/sftp:
    post:
      summary: Create SFTP data source
      tags:
        - Indexing
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/IndexKnowledgeBaseSFTPRequest'
      responses:
        '201':
          description: SFTP data source created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BaseResponse'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '422':
          $ref: '#/components/responses/ValidationError'

  /v1/index/{index_id}/sftp:
    put:
      summary: Update SFTP data source
      tags:
        - Indexing
      security:
        - BearerAuth: []
      parameters:
        - name: index_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateKnowledgeBaseSFTPRequest'
      responses:
        '200':
          description: SFTP data source updated successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BaseResponse'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '404':
          $ref: '#/components/responses/NotFound'

  /v1/index/datasource/health-check:
    post:
      summary: Test SFTP connection
      tags:
        - Indexing
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/DatasourceHealthCheckRequest'
      responses:
        '200':
          description: Connection test result
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/DatasourceHealthCheckResponse'
        '400':
          $ref: '#/components/responses/BadRequest'

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    IndexKnowledgeBaseSFTPRequest:
      type: object
      required:
        - name
        - project_name
        - description
        - hostname
        - username
        - password
        - directory_path
      properties:
        name:
          type: string
          minLength: 4
          maxLength: 50
          pattern: '^[a-zA-Z0-9][\w-]*$'
          description: Unique datasource name
        project_name:
          type: string
          description: Project to associate datasource with
        description:
          type: string
          minLength: 1
          maxLength: 500
          description: Datasource description (used by LLM for context)
        project_space_visible:
          type: boolean
          default: false
          description: Share datasource with entire project
        hostname:
          type: string
          description: SFTP server hostname or IP
        port:
          type: integer
          default: 22
          minimum: 1
          maximum: 65535
          description: SFTP server port
        username:
          type: string
          description: SFTP authentication username
        password:
          type: string
          format: password
          description: SFTP authentication password (encrypted at rest)
        directory_path:
          type: string
          description: Remote directory path to pull files from
        file_pattern:
          type: string
          default: "*.csv"
          description: Glob pattern for file filtering (CSV only)
        setting_id:
          type: string
          format: uuid
          description: Optional integration credential ID (for shared credentials)
        embedding_model:
          type: string
          description: Embedding model override (defaults to platform default)
        cron_expression:
          type: string
          description: Cron expression for automatic reindexing (default every 15 min)
        guardrail_assignments:
          type: array
          items:
            $ref: '#/components/schemas/GuardrailAssignmentItem'

    UpdateKnowledgeBaseSFTPRequest:
      type: object
      required:
        - name
        - project_name
      properties:
        name:
          type: string
        project_name:
          type: string
        description:
          type: string
          maxLength: 500
        project_space_visible:
          type: boolean
        hostname:
          type: string
        port:
          type: integer
        username:
          type: string
        password:
          type: string
          format: password
        directory_path:
          type: string
        file_pattern:
          type: string
        setting_id:
          type: string
        new_project_name:
          type: string
          description: New project association (triggers reindexing)
        guardrail_assignments:
          type: array
          items:
            $ref: '#/components/schemas/GuardrailAssignmentItem'
        cron_expression:
          type: string

    DatasourceHealthCheckRequest:
      type: object
      required:
        - project_name
        - index_type
      properties:
        project_name:
          type: string
        index_type:
          type: string
          enum: [knowledge_base_sftp]
        hostname:
          type: string
        port:
          type: integer
        username:
          type: string
        password:
          type: string
        directory_path:
          type: string
        setting_id:
          type: string

    DatasourceHealthCheckResponse:
      type: object
      properties:
        implemented:
          type: boolean
          default: true
        documents_count:
          type: integer
          nullable: true
        error:
          $ref: '#/components/schemas/ErrorMessage'

    ErrorMessage:
      type: object
      properties:
        message:
          type: string
        details:
          type: string
        help:
          type: string
        field_error:
          type: string

    BaseResponse:
      type: object
      properties:
        message:
          type: string

    GuardrailAssignmentItem:
      type: object
      properties:
        guardrail_id:
          type: string
          format: uuid
        mode:
          type: string
          enum: [input, output, both]
        source:
          type: string
          enum: [input, output]

  responses:
    BadRequest:
      description: Invalid request parameters
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorMessage'
    Unauthorized:
      description: Authentication required
    Forbidden:
      description: Insufficient permissions
    NotFound:
      description: Resource not found
    ValidationError:
      description: Request validation failed
      content:
        application/json:
          schema:
            type: object
            properties:
              error:
                type: object
                properties:
                  message:
                    type: string
```

## Architecture & Design

### Component Overview
1. **SFTP Loader** (`src/codemie/datasource/loader/sftp_loader.py`): BaseDatasourceLoader implementation for SFTP file retrieval using Paramiko
2. **SFTP Processor** (`src/codemie/datasource/sftp/sftp_datasource_processor.py`): BaseDatasourceProcessor implementation coordinating SFTP indexing lifecycle
3. **API Endpoints** (`src/codemie/rest_api/routers/index.py`): FastAPI routes for SFTP datasource CRUD operations
4. **Request/Response Models** (`src/codemie/rest_api/models/index.py`): Pydantic models for API validation
5. **Settings Integration** (`src/codemie/rest_api/models/settings.py`): SFTP credentials storage model

### Data Model
```python
# SFTPIndexInfo model (embedded in IndexInfo)
class SFTPIndexInfo(BaseModel):
    hostname: str  # e.g., "sftp.example.com"
    port: int = 22
    username: str
    password_encrypted: str  # Encrypted via EncryptionFactory
    directory_path: str  # e.g., "/data/exports"
    file_pattern: str = "*.csv"  # Glob pattern for file matching
```

### Security Considerations
- **Authentication**: Username/password stored encrypted using CodeMie vault (AWS KMS/Azure Key Vault/GCP KMS)
- **Data Encryption**: SFTP protocol provides transport layer encryption (SSH)
- **Credential Storage**: Follow existing pattern from SharePointDatasourceProcessor (`_encrypt_sftp_password`, `_decrypt_sftp_password`)
- **Input Validation**: Pydantic models validate all API inputs; hostname DNS validation
- **RBAC**: Existing CodeMie permission model (project admins, datasource owners)

### Error Handling
- **Connection Failures**: Log and surface to UI with actionable error messages (hostname unreachable, authentication failed, directory not found)
- **File Retrieval Errors**: Retry with exponential backoff (existing `@retry` decorator pattern)
- **Indexing Failures**: Existing `DatasourceMonitoringCallback` captures errors in IndexInfo model
- **Polling Failures**: Scheduler watchdog auto-resumes stale jobs

## Implementation Phases

### Phase 0: Research & Discovery ✅
- [x] Review existing SharePointDatasourceProcessor implementation *(completed during planning)*
- [x] Analyze CodeMie User Guide for datasource patterns *(completed during planning)*
- [x] Review authentication/authorization flow *(completed during planning)*
- [ ] Install and test Paramiko SFTP client library locally

### Phase 1: Dependencies & Data Models
**Goal**: Add Paramiko dependency and define SFTP data models

**Tasks**:
1. Add `paramiko` to `pyproject.toml` dependencies
2. Define `SFTPIndexInfo` model in `src/codemie/rest_api/models/index.py`
3. Define `SFTPCredentials` model in `src/codemie/rest_api/models/settings.py` (if using integration-based credentials)
4. Define API request models: `IndexKnowledgeBaseSFTPRequest`, `UpdateKnowledgeBaseSFTPRequest`
5. Update `IndexInfo` model to include `sftp` field (similar to `sharepoint` field)

**Files Created/Modified**:
- `pyproject.toml` (add paramiko ~3.5.0)
- `src/codemie/rest_api/models/index.py` (add SFTPIndexInfo, request models)
- `src/codemie/rest_api/models/settings.py` (add SFTPCredentials if using integration pattern)

**Acceptance Criteria**:
- Paramiko installed and importable
- Data models pass Pydantic validation tests
- Alembic migration created for new `sftp` JSONB field in `index_info` table

---

### Phase 2: SFTP Loader Implementation
**Goal**: Create BaseDatasourceLoader implementation for SFTP file retrieval

**Tasks**:
1. Create `src/codemie/datasource/loader/sftp_loader.py`
2. Implement `SFTPLoader(BaseDatasourceLoader)`:
   - Constructor: Accept hostname, port, username, password (decrypted), directory_path, file_pattern
   - `lazy_load()`: Connect to SFTP server, list files matching pattern, yield Documents
   - `fetch_remote_stats()`: Return `{DOCUMENTS_COUNT_KEY: file_count}` for UI progress tracking
   - `validate_connection()`: Test connection (used by health check endpoint)
3. Add Paramiko SFTP client wrapper with automatic disconnect on errors
4. Implement glob pattern matching for `*.csv` file filtering
5. Add modified_since support for incremental reindexing (check file modification timestamps)
6. Write unit tests for SFTPLoader

**Files Created/Modified**:
- `src/codemie/datasource/loader/sftp_loader.py` (new)
- `tests/codemie/datasource/loader/test_sftp_loader.py` (new)

**Acceptance Criteria**:
- SFTPLoader connects to SFTP server successfully
- `lazy_load()` yields Document objects with correct metadata (source, file_path)
- `fetch_remote_stats()` returns accurate file count
- Connection validation catches common errors (auth failure, directory not found)
- Unit tests achieve >80% coverage

---

### Phase 3: SFTP Processor Implementation
**Goal**: Create BaseDatasourceProcessor implementation coordinating SFTP indexing

**Tasks**:
1. Create `src/codemie/datasource/sftp/sftp_datasource_processor.py`
2. Implement `SFTPDatasourceProcessor(BaseDatasourceProcessor)`:
   - `_init_index()`: Create/update IndexInfo with SFTPIndexInfo config
   - `_init_loader()`: Return SFTPLoader instance with decrypted credentials
   - `_cleanup_data_for_incremental_reindex()`: Delete stale chunks from Elasticsearch (similar to SharePoint pattern)
   - `_process_chunk()`: Transform SFTP file chunks into Documents
   - `_on_process_end()`: Clear sensitive state (decrypted passwords)
   - `check_sftp_connection()` classmethod: Validate connection (used by API health check)
3. Add credential encryption/decryption helpers:
   - `_encrypt_sftp_password(password: str) -> str` using EncryptionFactory
   - `_decrypt_sftp_password(encrypted: str) -> str`
4. Integrate with SchedulerSettingsService for cron-based reindexing
5. Write unit tests for SFTPDatasourceProcessor

**Files Created/Modified**:
- `src/codemie/datasource/sftp/sftp_datasource_processor.py` (new)
- `src/codemie/datasource/sftp/__init__.py` (new)
- `tests/codemie/datasource/sftp/test_sftp_datasource_processor.py` (new)

**Acceptance Criteria**:
- Processor successfully indexes CSV files from SFTP server
- Incremental reindexing removes stale chunks and adds new files
- Passwords encrypted at rest using CodeMie vault
- Cron scheduling creates/updates scheduler settings
- Unit tests achieve >80% coverage

---

### Phase 4: REST API Endpoints
**Goal**: Add FastAPI endpoints for SFTP datasource CRUD operations

**Tasks**:
1. Update `src/codemie/rest_api/routers/index.py`:
   - `POST /v1/index/knowledge_base/sftp`: Create SFTP datasource
   - `PUT /v1/index/{index_id}/sftp`: Update SFTP datasource
   - Update `POST /v1/index/datasource/health-check`: Add SFTP connection validation
2. Add background task wrappers:
   - `index_sftp_datasource_in_background()`: Async task for initial indexing
   - `reindex_sftp_datasource_in_background()`: Async task for full reindex
3. Add authentication/authorization checks (existing `Ability(user).can()` pattern)
4. Add error handling with ExtendedHTTPException (user-friendly error messages)
5. Write integration tests for API endpoints

**Files Created/Modified**:
- `src/codemie/rest_api/routers/index.py` (add SFTP endpoints)
- `tests/codemie/rest_api/routers/test_index_sftp.py` (new, integration tests)

**Acceptance Criteria**:
- POST endpoint creates SFTP datasource and starts indexing in background
- PUT endpoint updates config and triggers reindex
- Health check endpoint validates connection before saving
- API returns proper HTTP status codes (201, 200, 400, 401, 403, 404, 422)
- Integration tests cover success and error scenarios

---

### Phase 5: Settings Integration (Optional)
**Goal**: Add SFTP to settings/integrations UI config

**Tasks**:
1. Update `src/codemie/service/constants.py`: Add `FullDatasourceTypes.SFTP = "knowledge_base_sftp"`
2. If using integration-based credentials (optional pattern):
   - Update settings config to include SFTP credential type
   - Add SFTP credentials to CredentialTypes enum
   - Test integration creation flow

**Files Created/Modified**:
- `src/codemie/service/constants.py` (add SFTP constant)
- (Optional) Settings-related files if using integration pattern

**Acceptance Criteria**:
- SFTP datasource type appears in platform constants
- (If using integration) SFTP credentials can be created as project/user integrations

---

### Phase 6: Testing & Documentation
**Goal**: Comprehensive testing and documentation

**Tasks**:
1. **Unit Tests**:
   - SFTPLoader: Connection, file listing, error handling
   - SFTPDatasourceProcessor: Indexing lifecycle, encryption, scheduling
2. **Integration Tests**:
   - API endpoints: CRUD operations, health checks
   - End-to-end: Create datasource → Index files → Query Elasticsearch
3. **Manual Testing**:
   - Test with real SFTP server (mock-sftp-server for CI)
   - Test polling interval (15 minutes)
   - Test credential encryption/decryption across KMS providers
4. **Documentation**:
   - Update CodeMie User Guide with SFTP setup instructions
   - Add API documentation (OpenAPI spec embedded above)
   - Create troubleshooting guide (common SFTP errors)
5. **Code Review**:
   - Follow CodeMie security patterns checklist
   - Validate against REST API patterns guide

**Files Created/Modified**:
- `tests/codemie/datasource/sftp/*` (unit tests)
- `tests/codemie/rest_api/routers/test_index_sftp.py` (integration tests)
- `docs/user-guide/sftp-datasource.md` (new)
- `docs/troubleshooting/sftp-errors.md` (new)

**Acceptance Criteria**:
- Unit test coverage >80% for new code
- Integration tests pass in CI pipeline
- End-to-end test successfully indexes CSV from SFTP server
- User Guide includes SFTP setup walkthrough with screenshots
- Troubleshooting guide covers top 5 common errors

## Expected Artifacts
- `plan-backend.md` (this file)
- OpenAPI 3.0 specification (embedded above)
- SFTP Loader (`sftp_loader.py`)
- SFTP Processor (`sftp_datasource_processor.py`)
- API endpoints (updated `index.py` router)
- Request/response models (updated `index.py` models)
- Unit and integration tests
- Updated CodeMie User Guide sections

## Dependencies
- **External Library**: `paramiko ~3.5.0` (Python SSH/SFTP client)
- **Platform Services**: EncryptionFactory, SchedulerSettingsService, ElasticsearchClient
- **Cross-Repository**: Frontend UI implementation (codemie-ui repository)

## Notes
- **Demo Timeline**: Backend implementation estimated at 5-7 days (includes testing); demo scheduled for July 15th
- **Paramiko Security**: Paramiko is actively maintained and follows SSH security best practices
- **Performance**: 15-minute polling interval prevents SFTP server overload; file deduplication avoids re-downloading unchanged files
- **Future Enhancements** (out of scope for v1):
  - SSH key-based authentication (in addition to username/password)
  - User-configurable polling intervals (5 min - 24 hrs)
  - Support for additional file formats (JSON, XML, etc.)
  - IP whitelisting configuration in UI
