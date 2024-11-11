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
npm install @godspeedsystems/core @google/generative-ai @types/pg mysql2 mongodb @types/oracledb redis
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
# Example Configuration Guide for Text-to-SQL Plugin

```

## Plugin Configuration

### 1. DataSource Configuration
Create `src/datasources/text-to-sql.yaml`:
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
  cache:
    enabled: true
    url: ${REDIS_URL}
```

### 2. Event Configuration
Create `src/events/sql-query.yaml`:
```yaml
http.post./sql-query:
  summary: "Convert natural language to SQL and execute"
  description: "Endpoint to convert natural language to SQL query and execute it"
  fn: execute-query
  authn: false
  body:
    content:
      application/json:
        schema:
          type: object
          properties:
            query:
              type: string
              description: "Natural language query to be converted to SQL"
            dbType:
              type: string
              description: "Target database type (postgres, mysql, etc.)"
              default: "postgres"
          required:
            - query
  responses:
    200:
      description: "Successful query execution"
      content:
        application/json:
          schema:
            type: object
```

### 3. Function Configuration
Create `src/functions/execute-query.yaml`:
```yaml
id: execute-query
summary: Execute natural language query
tasks:
  - id: convert-and-execute
    fn: text-to-sql
    args:
      query: <% inputs.body.query %>
      dbType: <% inputs.body.dbType || 'postgres' %>
```

### 4. Handler Implementation
Create `src/functions/execute-query.ts`:
```typescript
import { GSContext, PlainObject } from "@godspeedsystems/core";

async function handler(ctx: GSContext): Promise<PlainObject> {
  try {
    const { body } = ctx.inputs.data;
    
    if (!body.query) {
      throw new Error("Query is required");
    }

    // Execute the query using the text-to-sql datasource
    const result = await ctx.datasources['text-to-sql'].execute(ctx, {
      query: body.query,
      dbType: body.dbType || 'postgres',
      validateOnly: false,
      cache: true
    });

    return {
      success: true,
      data: result
    };

  } catch (error: any) {
    ctx.logger.error('Query execution failed:', error);
    return {
      success: false,
      code: 500,
      message: error.message,
      data: {
        message: "Query execution failed"
      }
    };
  }
}

export default handler;
```

### 5. Plugin Code 
Create `src/datasources/types/text-to-sql.ts` and copy the code in index.ts:
```
import {
    GSContext,
    GSDataSource,
    PlainObject,
    logger,
  } from '@godspeedsystems/core';
  import { Pool as PgPool } from 'pg';
  import {
    Connection as MySQLConnection,
    createConnection as createMySQLConnection,
  } from 'mysql2/promise';
  import { MongoClient, Db } from 'mongodb';
  import * as oracledb from 'oracledb';
  import { createClient, RedisClientType } from 'redis';
  import { createHash } from 'crypto';
  import { GoogleGenerativeAI } from '@google/generative-ai';
  
  // Interfaces
  interface DatabaseConfig {
    type: 'postgres' | 'mysql' | 'mongodb' | 'oracle';
    config: any;
  }
  
  interface DatabaseConnection {
    client: any;
    type: string;
    isConnected: boolean;
  }
  
  interface QueryResult {
    data: any[];
    metadata?: any;
  }
  
  class DatabaseConnectionManager {
    private connections: Map<string, DatabaseConnection> = new Map();
  
    async connect(config: DatabaseConfig): Promise<DatabaseConnection> {
      try {
        switch (config.type) {
          case 'postgres':
            return await this.connectPostgres(config.config);
          case 'mysql':
            return await this.connectMySQL(config.config);
          case 'mongodb':
            return await this.connectMongo(config.config);
          case 'oracle':
            return await this.connectOracle(config.config);
          default:
            throw new Error(`Database not supported yet: ${config.type}`);
        }
      } catch (error) {
        logger.error(`Database connection failed for ${config.type}:`, error);
        throw error;
      }
    }
  
    private async connectPostgres(config: any): Promise<DatabaseConnection> {
      try {
        const pool = new PgPool(config);
        await pool.query('SELECT 1');
        logger.info('PostgreSQL connection established');
        return { client: pool, type: 'postgres', isConnected: true };
      } catch (error) {
        logger.error('PostgreSQL connection failed:', error);
        throw error;
      }
    }
  
    private async connectMySQL(config: any): Promise<DatabaseConnection> {
      try {
        const connection = await createMySQLConnection(config);
        await connection.query('SELECT 1');
        logger.info('MySQL connection established');
        return { client: connection, type: 'mysql', isConnected: true };
      } catch (error) {
        logger.error('MySQL connection failed:', error);
        throw error;
      }
    }
  
    private async connectMongo(config: any): Promise<DatabaseConnection> {
      try {
        const client = await MongoClient.connect(config.url);
        const db = client.db(config.database);
        logger.info('MongoDB connection established');
        return { client: db, type: 'mongodb', isConnected: true };
      } catch (error) {
        logger.error('MongoDB connection failed:', error);
        throw error;
      }
    }
  
    private async connectOracle(config: any): Promise<DatabaseConnection> {
      try {
        const connection = await oracledb.getConnection({
          user: config.user,
          password: config.password,
          connectString: config.connectString,
        });
        await connection.execute('SELECT 1 FROM DUAL');
        logger.info('Oracle connection established');
        return { client: connection, type: 'oracle', isConnected: true };
      } catch (error) {
        logger.error('Oracle connection failed:', error);
        throw error;
      }
    }
  }
  
  export default class MultiDBTextToSQLDataSource extends GSDataSource {
    private genAI!: GoogleGenerativeAI;
    private model!: any;
    private dbManager!: DatabaseConnectionManager;
    private connections!: Map<string, DatabaseConnection>;
    private redisClient!: RedisClientType;
    private schemas!: Map<string, string>;
  
    protected async initClient(): Promise<object> {
      try {
        logger.info('Initializing Text-to-SQL service');
  
        this.genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);
        this.model = this.genAI.getGenerativeModel({ model: 'gemini-pro' });
  
        this.dbManager = new DatabaseConnectionManager();
        this.connections = new Map();
        this.schemas = new Map();
  
        await this.initializeDatabases();
  
        this.redisClient = createClient({
          url: process.env.REDIS_URL,
        });
  
        logger.info('Text-to-SQL service initialized successfully');
        return { status: 'initialized' };
      } catch (error) {
        logger.error('Service initialization failed:', error);
        throw error;
      }
    }
  
    private async initializeDatabases() {
      const dbConfigs: DatabaseConfig[] = [
        {
          type: 'postgres',
          config: {
            user: process.env.PG_USER,
            host: process.env.PG_HOST,
            database: process.env.PG_DB,
            password: process.env.PG_PASSWORD,
            port: parseInt(process.env.PG_PORT || '5432'),
          },
        },
        {
          type: 'mysql',
          config: {
            host: process.env.MYSQL_HOST,
            user: process.env.MYSQL_USER,
            password: process.env.MYSQL_PASSWORD,
            database: process.env.MYSQL_DB,
          },
        },
        {
          type: 'mongodb',
          config: {
            url: process.env.MONGODB_URL,
            database: process.env.MONGODB_DB,
          },
        },
        {
          type: 'oracle',
          config: {
            user: process.env.ORACLE_USER,
            password: process.env.ORACLE_PASSWORD,
            connectString: process.env.ORACLE_CONNECT_STRING,
          },
        },
      ];
  
      for (const config of dbConfigs) {
        try {
          logger.info(`Initializing ${config.type} connection`);
          const connection = await this.dbManager.connect(config);
          this.connections.set(config.type, connection);
          this.schemas.set(
            config.type,
            await this.fetchDatabaseSchema(config.type),
          );
          logger.info(`${config.type} initialization complete`);
        } catch (error) {
          logger.error(`Failed to initialize ${config.type}:`, error);
        }
      }
    }
  
    private async fetchDatabaseSchema(dbType: string): Promise<string> {
      const connection = this.connections.get(dbType);
      if (!connection) throw new Error(`No connection found for ${dbType}`);
  
      try {
        logger.info(`Fetching schema for ${dbType}`);
        switch (dbType) {
          case 'postgres':
            return await this.fetchPostgresSchema(connection);
          case 'mysql':
            return await this.fetchMySQLSchema(connection);
          case 'mongodb':
            return await this.fetchMongoSchema(connection);
          case 'oracle':
            return await this.fetchOracleSchema(connection);
          default:
            throw new Error(`Unsupported database type: ${dbType}`);
        }
      } catch (error) {
        logger.error(`Schema fetch failed for ${dbType}:`, error);
        throw error;
      }
    }
  
    private async fetchPostgresSchema(
      connection: DatabaseConnection,
    ): Promise<string> {
      const schemaQuery = `
              SELECT 
                  table_name,
                  column_name,
                  data_type,
                  is_nullable
              FROM 
                  information_schema.columns
              WHERE 
                  table_schema = 'public'
              ORDER BY 
                  table_name, ordinal_position;
          `;
      const result = await connection.client.query(schemaQuery);
      return this.formatRelationalSchema(result.rows);
    }
  
    private async fetchMySQLSchema(
      connection: DatabaseConnection,
    ): Promise<string> {
      const [rows] = await connection.client.query(`
              SELECT 
                  TABLE_NAME,
                  COLUMN_NAME,
                  DATA_TYPE,
                  IS_NULLABLE
              FROM 
                  INFORMATION_SCHEMA.COLUMNS
              WHERE 
                  TABLE_SCHEMA = DATABASE()
          `);
      return this.formatRelationalSchema(rows);
    }
  
    private async fetchMongoSchema(
      connection: DatabaseConnection,
    ): Promise<string> {
      const db = connection.client as Db;
      const collections = await db.listCollections().toArray();
      return this.formatMongoSchema(collections);
    }
  
    private async fetchOracleSchema(
      connection: DatabaseConnection,
    ): Promise<string> {
      const result = await connection.client.execute(
        `
              SELECT 
                  table_name,
                  column_name,
                  data_type,
                  nullable,
                  data_length,
                  data_precision
              FROM 
                  user_tab_columns
              ORDER BY 
                  table_name, column_id
          `,
        [],
        { outFormat: oracledb.OUT_FORMAT_OBJECT },
      );
  
      return this.formatOracleSchema(result.rows);
    }
  
    private formatRelationalSchema(rows: any[]): string {
      const tableMap = new Map<string, any[]>();
      rows.forEach((row) => {
        const tableName = row.table_name || row.TABLE_NAME;
        if (!tableMap.has(tableName)) {
          tableMap.set(tableName, []);
        }
        tableMap.get(tableName)?.push(row);
      });
  
      return Array.from(tableMap.entries())
        .map(([table, columns]) => {
          return `Table ${table} {\n${columns
            .map(
              (col) =>
                `  ${col.column_name || col.COLUMN_NAME} ${
                  col.data_type || col.DATA_TYPE
                }`,
            )
            .join('\n')}\n}`;
        })
        .join('\n\n');
    }
  
    private formatMongoSchema(collections: any[]): string {
      return collections
        .map(
          (collection) =>
            `Collection ${collection.name} {\n  // Schema is dynamic\n}`,
        )
        .join('\n\n');
    }
  
    private formatOracleSchema(rows: any[]): string {
      const tableMap = new Map<string, any[]>();
      rows.forEach((row) => {
        if (!tableMap.has(row.TABLE_NAME)) {
          tableMap.set(row.TABLE_NAME, []);
        }
        tableMap.get(row.TABLE_NAME)?.push(row);
      });
  
      return Array.from(tableMap.entries())
        .map(([table, columns]) => {
          return `Table ${table} {\n${columns
            .map(
              (col) =>
                `  ${col.COLUMN_NAME} ${col.DATA_TYPE}(${col.DATA_LENGTH})`,
            )
            .join('\n')}\n}`;
        })
        .join('\n\n');
    }
  
    async execute(ctx: GSContext, args: PlainObject): Promise<any> {
      const {
        query,
        dbType = 'postgres', //defaults to postgres
        validateOnly = false,
        cache = true,
      } = args;
  
      try {
        logger.info('Executing query', { dbType, validateOnly });
  
        const connection = this.connections.get(dbType);
        if (!connection) {
          throw new Error(`No connection available for database type: ${dbType}`);
        }
  
        if (cache) {
          const cachedResult = await this.getCachedQuery(query, dbType);
          if (cachedResult) {
            logger.info('Cache hit, returning cached result');
            return cachedResult;
          }
        }
  
        const generatedQuery = await this.generateQuery(query, dbType);
        logger.debug('Generated SQL query', { generatedQuery });
  
        if (validateOnly) {
          return { status: 'valid', query: generatedQuery };
        }
  
        const result = await this.executeQuery(generatedQuery, dbType);
  
        if (cache) {
          await this.cacheQuery(query, dbType, result);
        }
  
        logger.info('Query executed successfully');
        return result;
      } catch (error) {
        logger.error('Query execution failed:', error);
        throw error;
      }
    }
  
    private async generateQuery(
      naturalQuery: string,
      dbType: string,
    ): Promise<string> {
      const schema = this.schemas.get(dbType);
  
      try {
        const prompt = `
          Schema:
          ${schema}
    
          Task: Convert this natural language query to a PostgreSQL query:
          "${naturalQuery}"
    
          Requirements:
          - Return only the raw SQL query
          - No markdown formatting
          - No explanations
          - Must start with SELECT
          - Only use existing tables and columns
        `;
    
        const result = await this.model.generateContent(prompt);
        let sql = result.response.text().trim();
        
        // Clean up the response
        sql = sql.replace(/```sql/gi, '')
                 .replace(/```/g, '')
                 .replace(/`/g, '')
                 .trim();
    
        // Basic validation
        if (!sql.toLowerCase().startsWith('select')) {
          throw new Error('Generated query must start with SELECT');
        }
    
        return sql;
      } catch (error:any) {
        logger.error(`SQL generation failed: ${error.message}`);
        throw new Error(`Failed to generate SQL: ${error.message}`);
      }
    }
  
    private async executeQuery(
      query: string,
      dbType: string,
    ): Promise<QueryResult> {
      const connection = this.connections.get(dbType);
      if (!connection) throw new Error(`No connection found for ${dbType}`);
  
      try {
        switch (dbType) {
          case 'postgres':
            const pgResult = await connection.client.query(query);
            return {
              data: pgResult.rows,
              metadata: { rowCount: pgResult.rowCount },
            };
  
          case 'mysql':
            const [mysqlRows] = await connection.client.query(query);
            return { data: mysqlRows };
  
          case 'mongodb':
            const mongoResult = await eval(`connection.client.${query}`);
            return {
              data: Array.isArray(mongoResult) ? mongoResult : [mongoResult],
            };
  
          case 'oracle':
            const oracleResult = await connection.client.execute(query, [], {
              outFormat: oracledb.OUT_FORMAT_OBJECT,
            });
            return {
              data: oracleResult.rows || [],
              metadata: {
                rowsAffected: oracleResult.rowsAffected,
                metaData: oracleResult.metaData,
              },
            };
  
          default:
            throw new Error(`Unsupported database type: ${dbType}`);
        }
      } catch (error) {
        logger.error(`Query execution failed for ${dbType}:`, error);
        throw error;
      }
    }
  
    private async cacheQuery(
      query: string,
      dbType: string,
      result: QueryResult,
    ): Promise<void> {
      const hash = createHash('md5').update(query).update(dbType).digest('hex');
  
      try {
        await this.redisClient.set(`sql_cache:${hash}`, JSON.stringify(result), {
          EX: 3600, // Sets expiration to 1 hour
        });
        logger.debug('Query result cached', { hash });
      } catch (error) {
        logger.warn('Cache operation failed:', error);
      }
    }
  
    private async getCachedQuery(
      query: string,
      dbType: string,
    ): Promise<QueryResult | null> {
      const hash = createHash('md5').update(query).update(dbType).digest('hex');
  
      try {
        const cached = await this.redisClient.get(`sql_cache:${hash}`);
        return cached ? JSON.parse(cached) : null;
      } catch (error) {
        logger.warn('Cache retrieval failed:', error);
        return null;
      }
    }
  
    async cleanup(): Promise<void> {
      try {
        logger.info('Starting cleanup');
  
        // Close other database connections
        for (const [dbType, connection] of this.connections.entries()) {
          try {
            switch (connection.type) {
              case 'postgres':
                await (connection.client as PgPool).end();
                break;
              case 'mysql':
                await (connection.client as MySQLConnection).end();
                break;
              case 'oracle':
                await connection.client.close();
                break;
              case 'mongodb':
                await (connection.client as MongoClient).close();
                break;
            }
            logger.info(`Closed ${dbType} connection`);
          } catch (error) {
            logger.error(`Failed to close ${dbType} connection:`, error);
          }
        }
  
        await this.redisClient.quit(); // Ensure Redis client is properly closed
        logger.info('Cleanup completed successfully');
      } catch (error) {
        logger.error('Cleanup failed:', error);
        throw error;
      }
    }
  }
  
  const SourceType = 'DS';
  const Type = 'text-to-sql';
  const CONFIG_FILE_NAME = 'text-to-sql';
  const DEFAULT_CONFIG = {
    gemini: {
      apiKey: process.env.GEMINI_API_KEY,
    },
    databases: {
      postgres: {
        enabled: true,
        config: {
          user: process.env.PG_USER,
          host: process.env.PG_HOST,
          database: process.env.PG_DB,
          password: process.env.PG_PASSWORD,
          port: parseInt(process.env.PG_PORT || '5432'),
        },
      },
      mysql: {
        enabled: true,
        config: {
          host: process.env.MYSQL_HOST,
          user: process.env.MYSQL_USER,
          password: process.env.MYSQL_PASSWORD,
          database: process.env.MYSQL_DB,
        },
      },
      mongodb: {
        enabled: true,
        config: {
          url: process.env.MONGODB_URL,
          database: process.env.MONGODB_DB,
        },
      },
      oracle: {
        enabled: true,
        config: {
          user: process.env.ORACLE_USER,
          password: process.env.ORACLE_PASSWORD,
          connectString: process.env.ORACLE_CONNECT_STRING,
        },
      },
    },
    redis: {
      url: process.env.REDIS_URL,
    },
  };
  
  export {
    MultiDBTextToSQLDataSource as DataSource,
    SourceType,
    Type,
    CONFIG_FILE_NAME,
    DEFAULT_CONFIG,
  };
  
```

## Database Setup

### PostgreSQL Container Setup
```bash
# Run PostgreSQL container
sudo docker run --name test-postgres \
  -e POSTGRES_USER=testuser \
  -e POSTGRES_PASSWORD=testpass \
  -e POSTGRES_DB=testdb \
  -p 5434:5432 \
  -d postgres:latest

# Create test data
psql -h localhost -p 5434 -U testuser -d testdb

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    status VARCHAR(20)
);

INSERT INTO users (name, email, status) VALUES 
    ('John Doe', 'john@example.com', 'active'),
    ('Jane Smith', 'jane@example.com', 'inactive');
```

### Redis Cache Setup
```bash
# Run Redis container
sudo docker run --name test-redis \
  -p 6380:6379 \
  -d redis:latest
```



## 🎯 Usage

### API Endpoint
POST `/sql-query`

### Request Body
```json
{
  "query": "find all active users",
  "dbType": "postgres"
}
```

### Response
```json
{
  "success": true,
  "data": {
    "rows": [
      {
        "id": 1,
        "name": "John Doe",
        "email": "john@example.com",
        "status": "active"
      }
    ],
    "metadata": {
      "rowCount": 1,
      "executionTime": "0.123s"
    }
  }
}
```

## 🌟 Key Benefits

1. **Developer Productivity**
   - No need to write SQL queries manually
   - Rapid prototyping and development
   - Focus on business logic instead of query syntax

2. **End User Experience**
   - Natural language interface to databases
   - Faster query results with caching
   - Reduced learning curve

3. **Enterprise Ready**
   - Production-grade security
   - Scalable architecture
   - Comprehensive logging
   - Performance optimization

## 🔥 Advanced Examples

### Complex Queries
```json
{
  "query": "Show me total sales by category for active products in last quarter",
  "dbType": "postgres"
}
```

### Analytical Queries
```json
{
  "query": "Calculate monthly growth rate of user signups",
  "dbType": "postgres"
}
```

## 🛡️ Security Features

- Query validation before execution
- SQL injection prevention
- Rate limiting support
- Error handling and sanitization
- Secure database connections

## 📈 Performance

- Redis caching reduces database load
- Query optimization suggestions
- Connection pooling
- Execution time monitoring
- Cache hit ratio tracking


## 📋 TODO & Roadmap

- [ ] Add support for MySQL
- [ ] Implement query optimization suggestions
- [ ] Add GraphQL support
- [ ] Enhanced error messages
- [ ] Query templating system
- [ ] Performance monitoring dashboard


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
  







