### plugin

# Text-to-SQL DataSource Plugin Documentation

## Overview

A Godspeed plugin that enables natural language to SQL conversion and multi-database query execution with built-in caching support. This plugin leverages Gemini models to convert natural language queries into SQL statements and executes them across various database types.

## Features

- Natural language to SQL conversion using Google Gemini
- Multi-database support
  - PostgreSQL
  - MySQL
  - MongoDB
  - Oracle
- Automatic query caching with Redis
- Query validation
- Schema management
- Error handling and logging
- Connection pooling
- Resource cleanup

## Installation

### Prerequisites

```bash
# Required dependencies
npm install @godspeedsystems/core @google/generative-ai pg mysql2 mongodb oracledb redis
```
## Configuration

### Environment Setup
Create or update your `.env` file with the following:

```env
# Gemini API Configuration
GEMINI_API_KEY=your_gemini_api_key

# PostgreSQL Configuration
PG_USER=your_postgres_user
PG_HOST=localhost
PG_DB=your_database
PG_PASSWORD=your_password
PG_PORT=5432

# MySQL Configuration
MYSQL_HOST=localhost
MYSQL_USER=your_mysql_user
MYSQL_PASSWORD=your_password
MYSQL_DB=your_database

# MongoDB Configuration
MONGODB_URL=mongodb://localhost:27017
MONGODB_DB=your_database

# Oracle Configuration
ORACLE_USER=your_oracle_user
ORACLE_PASSWORD=your_password
ORACLE_CONNECT_STRING=localhost:1521/XEPDB1

# Redis Configuration
REDIS_URL=redis://localhost:6379
```

### DataSource Configuration
Create a YAML configuration file `src/datasources/text-to-sql.yaml`:

```yaml
type: text-to-sql
config:
  gemini:
    apiKey: ${GEMINI_API_KEY}
  databases:
    postgres:
      enabled: true
      config:
        user: ${PG_USER}
        host: ${PG_HOST}
        database: ${PG_DB}
        password: ${PG_PASSWORD}
        port: ${PG_PORT}
    mysql:
      enabled: true
      config:
        host: ${MYSQL_HOST}
        user: ${MYSQL_USER}
        password: ${MYSQL_PASSWORD}
        database: ${MYSQL_DB}
    mongodb:
      enabled: true
      config:
        url: ${MONGODB_URL}
        database: ${MONGODB_DB}
    oracle:
      enabled: true
      config:
        user: ${ORACLE_USER}
        password: ${ORACLE_PASSWORD}
        connectString: ${ORACLE_CONNECT_STRING}
  cache:
    enabled: true
    ttl: 3600
```

## Example Usage

### Basic Query Event Configuration
Create a new file `src/events/query.yaml`:

```yaml
'http.post./query':
  fn: execute-query
  body:
    content:
      application/json:
        schema:
          type: object
          properties:
            query:
              type: string
            dbType:
              type: string
              default: postgres
          required: ['query']
```

### Query Execution Workflow
Define your function `src/functions/execute-query.yaml`:

```yaml
id: execute-query
summary: Execute natural language query
tasks:
  - id: convert-and-execute
    fn: datasource.text-to-sql.execute
    args:
      query: <% inputs.body.query %>
      dbType: <% inputs.body.dbType || 'postgres' %>
```

### Example Requests

1. **Simple Query Request:**
```bash
curl -X POST http://localhost:3000/query \
  -H "Content-Type: application/json" \
  -d '{"query": "Find all active users"}'
```

2. **Complex Query Request:**
```bash
curl -X POST http://localhost:3000/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Show total sales by category for the last quarter",
    "dbType": "mysql"
  }'
```

### Example Responses

```json
{
  "sql": "SELECT category, SUM(amount) AS total_sales FROM sales WHERE date >= DATE_SUB(CURDATE(), INTERVAL 3 MONTH)",
  "data": [
    {
      "category": "Electronics",
      "total_sales": 105000
    }
  ],
  "metadata": {
    "rowCount": 1,
    "executionTime": 48
  }
}
```

## Error Handling

```json
{
  "error": "QUERY_ERROR",
  "message": "Failed to execute query",
  "details": "Syntax error at or near 'total'"
}
```


```

## Limitations

1. Natural language processing depends on Google Gemini
2. Redis required for caching functionality
3. Database-specific features may vary

## Troubleshooting

### Common Issues

1. Connection Failures

   - Verify database credentials
   - Check network connectivity
   - Ensure services are running

2. Query Generation Issues

   - Validate Google gemini API key
   - Check query format
   - Review database schema

3. Cache Problems
   - Verify Redis connection
   - Check cache configuration
   - Monitor memory usage
  







