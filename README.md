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

### Environment Variables

Create a `.env` file with the following variables:

```env
# Gemini Configuration
GEMINI_API_KEY=your_gemini_api_key_here

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

### Plugin Configuration

Create `text-to-sql.json` in your project:

```json
{
  "gemini": {
    "apiKey": "${process.env.GEMINI_API_KEY}"
  },
  "databases": {
    "postgres": {
      "enabled": true,
      "config": {
        "user": "${PG_USER}",
        "host": "${PG_HOST}",
        "database": "${PG_DB}",
        "password": "${PG_PASSWORD}",
        "port": "${PG_PORT}"
      }
    }
  },
  "redis": {
    "url": "${REDIS_URL}"
  }
}
```

## Usage

### Basic Query Execution

```typescript
const result = await dataSource.execute(ctx, {
  query: "Find all active users",
  dbType: "postgres",
});
```

### Query Validation

```typescript
const validation = await dataSource.execute(ctx, {
  query: "Show me sales data",
  validateOnly: true,
});
```

### Cache Control

```typescript
const result = await dataSource.execute(ctx, {
  query: "List all products",
  cache: false, // Disable caching
});
```

## API Reference

### DataSource Class Methods

#### initClient()

Initializes database connections and caching.

```typescript
protected async initClient(): Promise<object>
```

#### execute()

Executes queries and manages results.

```typescript
async execute(
  ctx: GSContext,
  args: {
    query: string;          // Natural language query
    dbType?: string;        // Database type (default: 'postgres')
    validateOnly?: boolean; // Validate without executing
    cache?: boolean;       // Enable/disable caching
  }
): Promise<QueryResult>
```

#### cleanup()

Closes all connections properly.

```typescript
async cleanup(): Promise<void>
```

### Interfaces

#### QueryResult

```typescript
interface QueryResult {
  data: any[];
  metadata?: {
    rowCount?: number;
    columns?: string[];
  };
}
```

#### DatabaseConfig

```typescript
interface DatabaseConfig {
  type: "postgres" | "mysql" | "mongodb" | "oracle";
  config: any;
}
```

## Error Handling

### Common Errors

```typescript
// Database Connection Error
{
  code: 'CONNECTION_ERROR',
  message: 'Failed to connect to database',
  details: error.message
}

// Query Execution Error
{
  code: 'QUERY_ERROR',
  message: 'Failed to execute query',
  details: error.message
}
```

## Best Practices

### Query Writing

1. Be specific with natural language queries
2. Include relevant context
3. Use proper English

### Caching Strategy

1. Enable caching for read-heavy operations
2. Disable for real-time data requirements
3. Configure appropriate TTL values

### Connection Management

1. Properly initialize connections
2. Monitor connection pools
3. Implement cleanup procedures

## Logging

The plugin uses Godspeed's built-in logger:

```typescript
logger.info("Operation successful");
logger.error("Operation failed:", error);
logger.debug("Debug information:", data);
```

## Examples

### Complex Queries

```typescript
// Analytical Query
const analysis = await dataSource.execute(ctx, {
  query: "Calculate monthly sales growth by category for last quarter",
  dbType: "postgres",
});

// Filtered Query
const filtered = await dataSource.execute(ctx, {
  query: "Find orders worth more than $1000 from California",
  dbType: "mysql",
});
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

## License

This project is licensed under the MIT License.
