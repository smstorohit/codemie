## Implementation Phases

### Phase 0: Research & Discovery

- [x] Review existing data source processors (File, SharePoint, Confluence) **completed**
- [x] Analyze CodeMie User Guide for data source patterns **completed**
- [x] Confirm Paramiko, mtime, and Integration selector approach **completed**
- [ ] Set up development environment with test SFTP server
- [ ] Validate Paramiko library compatibility with CodeMie Python version

### Phase 1: Core Infrastructure

**Goal**: Implement SFTP connection service, loader, and configuration.

- [ ] Add Paramiko dependency to `requirements.txt`
- [ ] Create `SftpConnectionService` with connection pooling, authentication, and `test_connection` method
- [ ] Implement `SetPLoader` (extends `BaseDatasourceLoader`) with `lazy_load` and `fetch_remote_stats`
- [ ] Add file pattern matching (glob) and mtime-based filtering to `SftpLoader`
- [ ] Implement 50MB file size limit and skip logic
- [ ] Add SFTP configuration to `datasources-config.yaml`
- [ ] Write unit tests for `SftpConnectionService` and `SftpLoader`

### Phase 2: Data Source Processor

**Goal**: Implement `SftpDatasourceProcessor` with scheduling and CSV indexing.

- [ ] Create FetPDatasourceProcessor` (extends `BaseDatasourceProcessor`)
- [ ] Implement `process` method (initial indexing with all files)
- [ ] Implement `incremental_reindex` method (mtime-based filtering)
- [ ] Implement `reprocess` and `resume` methods
- [ ] Integrate with `IndexService` for automatic CSV indexing
- [ ] Add scheduler creation/update via `_create_or_update_scheduler`
- [ ] Integrate with `DatasourceMonitoringCallback` for audit logging
- [ ] Write unit and integration tests for `SftpDatasourceProcessor`

### Phase 3: REST API Endpoints

**Goal**: Expose SFTP data source endpoints for UI integration.

- [ ] Define Pidantic models: `CreateIndexKnowledgeBaseSftp`, `UpdateIndexKnowledgeBaseSftp`
- [ ] Implement `POST /index/knowledge_base/sftp` (create)
- [ ] Implement `POST /index/knowledge_base/sftp/test-connection` (with file list preview)
- [ ] Implement `GET /index/knowledge_base/sftp/{id}` (read)
- [ ] Implement `PUT /index/knowledge_base/sftp/{id}` (update)
- [ ] Implement `DELETE /index/knowledge_base/sftp/{id}` (delete)
- [ ] Add error handling with clear messages for each endpoint
- [ ] Add rate limiting on Test Connection endpoint
- [ ] Write API integration tests (positive + negative scenarios)

### Phase 4: Testing & Optimization

**Goal**: Validate end-to-end functionality, performance, and security.

- [ ] Set up test SFTP server with CSV sample data (small and large datasets)
- [ ] Test initial indexing with 500GB dataset (simulated)
- [ ] Test incremental re-indexing with 2-3GB daily updates
- [ ] Test error scenarios (invalid credentials, network timeout, directory not found)
- [ ] Test >50MB file handling (skip + logging)
- [ ] Test Vault credential retrieval and RBAC
- [ ] Test scheduler triggers (manually and via cron)
- [ ] Performance testing: measure download speed, CSV parsing time, indexing throughput
- [ ] Security review: validate credential storage, log sanitization, input validation
- [ ] Address performance bottlenecks (parallel downloads, batching)

### Phase 5: Documentation & Deployment

**Goal**: Prepare for production launch and demo.

- [ ] Update OpenAPI specification for SFTP endpoints
- [ ] Write internal developer documentation (architecture, deployment)
- [ ] Update CodeMie User Guide with SFTP data source section
- [ ] Create troubleshooting guide for common SFTP issues
- [ ] Document security best practices (IP whitelisting, credential rotation)
- [ ] Prepare demo environment for July 15th demo
- [ ] Create demo script and slides
- [ ] Deploy to staging environment
- [ ] Conduct UAT with stakeholders
- [ ] Deploy to production (early August)
