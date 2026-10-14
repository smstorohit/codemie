## Research Findings

### Existing Codebase Patterns

- **BaseDatasourceProcessor**: Abstract base class with `process`, `reprocess`, `resume`, `incremental_reindex` methods
- **BaseDatasourceLoader**: LangChain loader with `lazy_load` and `fetch_remote_stats` methods
- **FileDatasourceProcessor**: Handles CSV files with separator, start row, rows_per_document configuration
- **VaultEncryptionService**: HashiCorp Vault integration for secure credential storage with RBAC
- **Cron Scheduling**: `cron_expression` field in processors, `_create_or_update_scheduler` method
- **Datasources Config**: `datasources-config.yaml` with loader-specific settings (timeouts, batch sizes)
- **REST API Patterns**: `/index/knowledge_base/{type}` endpoints in `routers/index.py`
- **Model Structure**: `CreateIndexKnowledgeBase` with `project_id`, `name`, `description`, `integration_id`, `cron_expression`

### CodeMie User Guide References

- **Data Source Naming Restrictions**: Lowercase, 4-25 chars, only `-` and `_` symbols
- **Standard Fields**: Project, Name, Description, Shared with Project Team, Data Source Type
- **Integration Requirement**: Data sources require user/project integrations for authentication
- **Indexing Workflow**: Create → Automatic indexing ‒ View/Edit/Delete actions
- **CSV Configuration**: Separator, start row configuration already supported

### Technical Decisions (Confirmed)

1. **SFTP Library**: Paramiko (industry standard, well-maintained, thread-safe)
2. **File Download Strategy**: Stream files, enforce 50MB limit (consistent with File/SharePoint)
3. **Incremental Re-indexing**: File modification timestamp (mtime) comparison (aligns with SharePoint pattern)
4. **Integration Pattern**: Use existing Integration selector (consistent with Confluence/Jira/SharePoint)
5. **Test Connection**: Return file list (names + sizes) for UX preview
6. **Scheduling**: Reuse existing `CronScheduleInput` component, add 5, 10, 15 minute presets
