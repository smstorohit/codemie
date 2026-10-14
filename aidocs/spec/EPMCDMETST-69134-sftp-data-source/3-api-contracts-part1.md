## API Contracts

```yaml
openapi: 3.0.3
info:
  title: CodeMie SFTP Data Source API
  version: 1.0.0
  description: |
    API endpoints for managing SFTP data sources in CodeMie. 
    Follows existing `/index/knowledge_base/{type}` patterns.

paths:
  /index/knowledge_base/sftp:
    post:
      summary: Create a new SFTP data source
      description: |
        Creates a new SFTP data source with secure credential storage in Vault.
        Requires a valid integration_id (SFTP integration).
        Automatically creates a scheduler for polling based on cron_expression.
      operationId: createSftpDatasource
      tags:
        - Data Sources
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateSftpDatasourceRequest'
      responses:
        '201':
          description: SFTP data source created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/DatasourceResponse'
        '400':
          description: Bad request (invalid name, integration_id, or cron_expression)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Project or integration not found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '500':
          description: Internal server error (Vault failure, scheduler error)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

  /index/knowledge_base/sftp/test-connection:
    post:
      summary: Test SFTP connection and return file list preview
      description: |
        Validates SFTP credentials and connectivity without creating a data source.
        Returns a preview of files matching the file pattern in the specified directory.
        Used by the UI "Test Connection" button.
      operationId: testSftpConnection
      tags:
        - Data Sources
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/TestSftpConnectionRequest'
      responses:
        '200':
          description: Connection successful, file list returned
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/TestConnectionResponse'
        '400':
          description: Bad request (missing/required fields)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '401':
          description: Authentication failed (invalid username/password)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Directory not found on SFTP server
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '503':
          description: SFTP server unreachable (network timeout)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
```
