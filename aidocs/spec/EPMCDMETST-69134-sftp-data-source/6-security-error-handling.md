## Security & Error Handling

### Security Considerations

1. **Credential Storage**:
   - SFTP credentials (username, password) stored in HashiCorp Vault
   - Role-based access control (RBAC) enforced at project level
   - Credentials never exposed in API responses or logs

2. **Transport Security**:
   - SFTP (SSH-based) ensures encrypted data transfer
   - Paramiko supports host key verification (configurable)

3. **Input Validation**:
   - Pidantic validation for all API inputs (name pattern, port range, cron expression)
   - Sanitize `directory_path` and `file_pattern` to prevent path traversal
   - Rate limiting on Test Connection endpoint (10 requests/minute)

4. **Audit Logging**:
   - All SFTP operations logged via `DatasourceMonitoringCallback`
   - Log events: connection success/failure, files fetched (names + sizes), authentication errors
   - Include `project_id`, `user_id`, `datasource_id`, `timestamp`

5. **IP Whitelisting**:
   - Out of scope for CodeMie (configured externally by client's IT team)
   - Document in user guide

### Error Handling

1. **Connection Errors**:
   - `Paramiko.ssh_exception.SSHException`: Network timeout or host unreachable ‒ Return 503 with clear message
   - `Paramiko.ssh_exception.AuthenticationException`: Invalid credentials → Return 401 with troubleshooting hints
   - `FileNotFoundError`: Directory doesn't exist ‒ Return 404 with suggestions

2. **File Processing Errors**:
   - File >50MB: Skip, log warning, continue with next file
   - CSV parsing error: Log error, skip file, continue
   - Network interruption during download: Retry with exponential backoff (3 retries)

3. **Scheduler Errors**:
   - Invalid cron expression: Return 400 with validation error during create/update
   - Scheduler creation failure: Roll back data source creation, return 500

4. **Vault Errors**:
   - Vault unavailable: Return 500 with generic error (don't expose Vault details)
   - Credential not found: Return 404 with "integration not found" message

5. **Logging**:
   - Use `logging` module with structured logs (JSON format)
   - Error levels: `ILDOO` (connection success, files fetched), `WARN` (file skipped), `ERROR` (connection failure)
   - Never log sensitive data (passwords, tokens, file content)
