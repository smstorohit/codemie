# Implementation Plan: Add SFTP Data Source (Backend - codemie)

## Overview

This plan covers the backend implementation for adding SFTP (Secure File Transfer Protocol) data source support to the CodeMie platform. This feature enables users to connect to SFTP servers, automatically fetch CSV files from specified directories, and index them into the CodeMie vector database for AI-assisted queries and analysis.

The implementation will follow existing CodeMie data source patterns (File, Confluence, Jira, SharePoint) and leverage the established architecture for credential management, polling, and indexing.

## User Story

**As a** CodeMie administrator or project team member,  
**I want to** configure and connect to SFTP servers as a data source within the CodeMie setup wizard,  
**so that** I can automatically fetch and index CSV files from legacy systems for use in AI-assisted queries and analysis.

### Acceptance Criteria

1. **SFTP Data Source Type Available**: The "Add Data Source" wizard includes a new "SFTP" option.
2. **Configuration Form Fields**: Support for hostname, port (default 22), username, password, directory path, file pattern, and polling interval.
3. **Credential Security**: SFTP credentials stored in encrypted vault with RBAC.
4. **Connection Validation**: API Test Connection endpoint validates credentials and returns file list preview.
5. **Automated File Fetching**: Polls SFTP directory at configured interval, fetches CSV files, and auto-indexes.
6. **Audit Logging**: All SFTP operations logged with timestamps and user context.
7. **Large Data Volume Handling**: Handles 500GB initial + 2-3GB daily updates without performance degradation.

### Assumptions

- Only SFTP protocol supported (no plain FTP or FTPS)
- Only CSV file format supported for initial release
- 50MB per-file limit (consistent with File data source)
- SFTP server accessible from CodeMie platform
- IP whitelisting handled externally by client's IT team
- Files auto-indexed without user selection
- Duplicate detection via filename + mtime
- Sufficient storage capacity for data volumes

## Technical Context

- **Repository**: codemie (backend)
- **Tech Stack**: Python 3.11+, FastAPI, LangChain, Paramiko, HashiCorp Vault, PostgreSQL
- **Dependencies**:
  - Paramiko (SSH/SFTP client library)
  - HashiCorp Vault (credential storage)
  - LangChain CSV loader (reuse existing parser)
  - Cron scheduler system (existing trigger infrastructure)
- **Integrations**:
  - Index Service (auto-indexing CSV files)
  - Encryption Service (Vault integration)
  - Datasource Monitoring Callback (audit logging)
  - Platform Indexing Service (cross-project sync)
