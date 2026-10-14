## Architecture & Design

### Component Overview

1. **SftpDatasourceProcessor** (`src/codemie/datasource/sftp/sftp_datasource_processor.py`):
   - Extends `BaseDatasourceProcessor`
   - Implements `process`, `reprocess`, `resume`, `incremental_reindex` methods
   - Manages scheduler creation/update via `_create_or_update_scheduler`
   - Integrates with `SftpLoader` for file fetching

2. **SftpLoader** (`src/codemie/datasource/loader/sftp_loader.py`):
   - Extends `BaseDatasourceLoader` (LangChain loader)
   - Uses Paramiko for SFTP connections
   - Implements `lazy_load` (yields `Document` objects) and `fetch_remote_stats`
   - Supports file pattern matching (glob), mtime-based filtering, 50MB file size limit
   - Delegates CSV parsing to LangChain's `CSVLoader`

3. **SftpConnectionService** (`src/codemie/service/sftp/sftp_connection_service.py`):
   - Handles Paramiko SSH/SFTP connection lifecycle
   - Provides `test_connection` method (returns file list + validation results)
   - Integrates with `VaultEncryptionService` to retrieve decrypted credentials
   - Implements connection pooling and retry logic with exponential backoff

4. **REST API Endpoints** (`src/codemie/rest_api/routers/index.py`):
   - `POST /index/knowledge_base/sftp`: Create SFTP data source
   - `POST /index/knowledge_base/sftp/test-connection`: Test SFTP connection
   - `GET /PUT/DELETE /index/knowledge_base/sftp/{id}`: Manage SFTP data sources

5. **Pidantic Models** (`src/codemie/rest_api/models/index.py`):
   - `CreateIndexKnowledgeBaseSftp` (extends `CreateIndexKnowledgeBase`)
   - `UpdateIndexKnowledgeBaseSftp`
   - Add fields: `directory_path`, `file_pattern`, `csv_separator`, `csv_start_row`, `csv_rows_per_document`

6. **Configuration** (`config/datasources/datasources-config.yaml`):
   ```yaml
   sftp:
     loader:
       connection_timeout_s: 30
       max_file_size_mb: 50
       loader_batch_size: 10
       max_retries: 3
       retry_delay_s: 5
   ```

### Data Model

**SFTP Data Source Entity**:

- `id`: UUID
- `name`: String (4-25 chars, lowercase, `-` and `_` only)
- `description`: String (optional, max 500 chars)
- `project_id`: UUID (FK)
- `integration_id`: UUID (FK to SFTP integration)
- `directory_path`: String (remote SFTP path)
- `file_pattern`: String (default: `*`)
- `cron_expression`: String (default: `*/15 * * * *`)
- `shared_with_project_team`: Boolean
- `csv_separator`: String (default: `,`)
- `csv_start_row`: Integer (default: 0)
- `csv_rows_per_document`: Integer (default: 100)
- `status`: Enum (`active`, `indexing`, `failed`)
- `created_at`: DateTime
- `updated_at`: DateTime
- `last_indexing_date`: DateTime

**SFTP Integration Entity**: (created by user via Integrations section)

- `id`: UUID
- `name`: String
- `type`: `sftp` (static)
- `hostname`: String
- `port`: Integer (default: 22)
- `username`: String (encrypted in Vault)
- `password`: String (encrypted in Vault)
- `user_id` or `project_id`: Ownership
