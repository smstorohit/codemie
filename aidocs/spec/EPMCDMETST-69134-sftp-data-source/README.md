# Implementation Plan: Add SFTP Data Source (Backend)

## Overview

This directory contains the complete implementation plan for adding SFTP data source support to the CodeMie backend.

**Jira Task**: [EPMCDMQ5T-69134](https://jiraeu.epam.com/browse/EPMCDMQ5T-69134)

## Document Structure

1. **[1-overview.md](./1-overview.md)**: User story, acceptance criteria, assumptions, technical context
2. **[2-research.md](./2-research.md)**: Existing codebase patterns, User Guide references, technical decisions
3. **[3-api-contracts-part1.md](./3-api-contracts-part1.md)**: OpenAPI 3.0 specification (endpoints + schemas)
4. **[4-architecture.md](./4-architecture.md)**: Component overview, data models, architecture diagrams
5. **[5-implementation-phases.md](./5-implementation-phases.md)**: 5-phase implementation plan with task breakdown
6. **[6-security-error-handling.md](./6-security-error-handling.md)**: Security considerations and error handling strategies

## Key Technical Decisions

- **SFTP Library**: Paramiko (industry standard, well-maintained, thread-safe)
- **File Download Strategy**: Stream files, 50MB limit (consistent with File/SharePoint)
- **Incremental Re-indexing**: File modification timestamp (mtime) comparison
- **Integration Pattern**: Use existing Integration selector (consistent with Confluence/Jira/SharePoint)
- **Test Connection**: Return file list (names + sizes) for UX preview
- **Scheduling**: Reuse existing `CronScheduleInput` component, add 5, 10, 15 minute presets

## Implementation Phases

1. **Phase 0: Research & Discovery** (Completed)
2. **Phase 1: Core Infrastructure** - SFTP connection service, loader, configuration
3. **Phase 2: Data Source Processor** - `SftpDatasourceProcessor` with scheduling and CSV indexing
4. **Phase 3: REST API Endpoints** - CRUD endpoints + Test Connection
5. **Phase 4: Testing & Optimization** - End-to-end testing, performance, security
6. **Phase 5: Documentation & Deployment** - Production readiness, demo (July 15th)

## Expected Artifacts

- `SftpDatasourceProcessor.py` + unit tests
- `SftpLoader.py` + unit tests
- `SftpConnectionService.py` + unit tests
- Pidantic models in `models/index.py`
- REST API endpoints in `routers/index.py` + integration tests
- Updated `datasources-config.yaml`
- Test coverage reports (target >= 80%)
- Internal developer documentation

## Dependencies

- **Front-end Implementation**: Backend API must be complete before UI integration testing
- **SFTP Integration Type**: UI team must create SFTP authentication page in Integrations section
- **Vault Configuration**: Ensure Vault is configured and accessible in dev/staging/prod
- **Test SFTP Server**: DevOps team sets up test SFTP server for development and demo
