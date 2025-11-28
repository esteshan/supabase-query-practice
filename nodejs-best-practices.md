# Node.js Best Practices

> **Purpose**: This document serves as a reference for AI-assisted code reviews to ensure Node.js applications follow established best practices. It should be consulted periodically when reviewing or writing Node.js code.
>
> **Official References**:
> - [Node.js Introduction](https://nodejs.org/en/learn/getting-started/introduction-to-nodejs)
> - [Node.js API Documentation](https://nodejs.org/docs/latest/api/)
> - [Node.js Security Best Practices](https://nodejs.org/en/learn/getting-started/security-best-practices)

---

## Table of Contents

1. [Core Principles](#1-core-principles)
2. [Asynchronous Programming](#2-asynchronous-programming)
3. [Event Loop & Non-Blocking I/O](#3-event-loop--non-blocking-io)
4. [Error Handling](#4-error-handling)
5. [Security](#5-security)
6. [Module System](#6-module-system)
7. [Project Structure](#7-project-structure)
8. [Environment & Configuration](#8-environment--configuration)
9. [HTTP & Networking](#9-http--networking)
10. [File System Operations](#10-file-system-operations)
11. [Streams & Buffers](#11-streams--buffers)
12. [Performance Optimization](#12-performance-optimization)
13. [Logging & Monitoring](#13-logging--monitoring)
14. [Testing](#14-testing)
15. [Dependency Management](#15-dependency-management)
16. [Process Management](#16-process-management)
17. [Documentation](#17-documentation)

---

## 1. Core Principles

### 1.1 Node.js Runtime Understanding

- **Single-Threaded Event Loop**: Node.js runs on a single thread using an event-driven, non-blocking I/O model
- **V8 Engine**: Node.js runs the V8 JavaScript engine outside of the browser
- **Non-Blocking by Default**: Libraries in Node.js are generally written using non-blocking paradigms

### 1.2 Fundamental Rules

```javascript
// ✅ DO: Use const for variables that won't be reassigned
const server = createServer();

// ✅ DO: Use let only when reassignment is needed
let connectionCount = 0;

// ❌ DON'T: Use var (function-scoped, hoisted)
var outdatedVariable = 'avoid';
```

### 1.3 ECMAScript Standards

- Use modern ECMAScript features (ES2020+)
- Leverage optional chaining (`?.`) and nullish coalescing (`??`)
- Use template literals instead of string concatenation

```javascript
// ✅ DO: Modern syntax
const userName = user?.profile?.name ?? 'Anonymous';
const message = `Hello, ${userName}!`;

// ❌ DON'T: Legacy patterns
const userName = user && user.profile && user.profile.name || 'Anonymous';
const message = 'Hello, ' + userName + '!';
```

---

## 2. Asynchronous Programming

### 2.1 Prefer async/await Over Callbacks

```javascript
// ✅ DO: Use async/await for cleaner code
async function fetchUserData(userId) {
  try {
    const user = await db.findUser(userId);
    const orders = await db.findOrders(user.id);
    return { user, orders };
  } catch (error) {
    throw new Error(`Failed to fetch user data: ${error.message}`);
  }
}

// ❌ DON'T: Callback hell
function fetchUserData(userId, callback) {
  db.findUser(userId, (err, user) => {
    if (err) return callback(err);
    db.findOrders(user.id, (err, orders) => {
      if (err) return callback(err);
      callback(null, { user, orders });
    });
  });
}
```

### 2.2 Parallel Execution with Promise.all

```javascript
// ✅ DO: Run independent operations in parallel
async function fetchDashboardData(userId) {
  const [user, notifications, analytics] = await Promise.all([
    fetchUser(userId),
    fetchNotifications(userId),
    fetchAnalytics(userId),
  ]);
  return { user, notifications, analytics };
}

// ❌ DON'T: Sequential execution when parallel is possible
async function fetchDashboardData(userId) {
  const user = await fetchUser(userId);
  const notifications = await fetchNotifications(userId);
  const analytics = await fetchAnalytics(userId);
  return { user, notifications, analytics };
}
```

### 2.3 Handle Promise Rejections

```javascript
// ✅ DO: Always handle rejections
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
  // Application-specific logging, throwing an error, or other logic
});

// ✅ DO: Use Promise.allSettled when you need all results regardless of failures
const results = await Promise.allSettled([
  fetchCriticalData(),
  fetchOptionalData(),
]);
```

### 2.4 Don't Await in Return Statements

```javascript
// ✅ DO: Return the promise directly
async function getData() {
  return fetchFromDatabase();
}

// ❌ DON'T: Unnecessary await in return
async function getData() {
  return await fetchFromDatabase();
}
```

---

## 3. Event Loop & Non-Blocking I/O

### 3.1 Never Block the Event Loop

```javascript
// ❌ DON'T: Synchronous operations that block
const data = fs.readFileSync('/large-file.txt');
const hash = crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512');

// ✅ DO: Use asynchronous alternatives
const data = await fs.promises.readFile('/large-file.txt');
const hash = await new Promise((resolve, reject) => {
  crypto.pbkdf2(password, salt, 100000, 64, 'sha512', (err, key) => {
    if (err) reject(err);
    else resolve(key);
  });
});
```

### 3.2 Offload CPU-Intensive Tasks

```javascript
// ✅ DO: Use Worker Threads for CPU-intensive operations
import { Worker, isMainThread, parentPort } from 'node:worker_threads';

if (isMainThread) {
  const worker = new Worker('./cpu-intensive-task.js');
  worker.on('message', (result) => console.log(result));
  worker.postMessage({ data: largeDataset });
} else {
  parentPort.on('message', (data) => {
    const result = performHeavyComputation(data);
    parentPort.postMessage(result);
  });
}
```

### 3.3 Understand setImmediate vs process.nextTick

```javascript
// process.nextTick - executes before the event loop continues
// setImmediate - executes on the next iteration of the event loop

// ✅ DO: Use setImmediate for I/O callbacks
setImmediate(() => {
  // Runs after I/O events
});

// ✅ DO: Use process.nextTick sparingly for immediate execution
process.nextTick(() => {
  // Runs before any I/O events
});
```

---

## 4. Error Handling

### 4.1 Use Error-First Callbacks (Legacy Pattern)

```javascript
// When working with callback-based APIs
function readConfig(callback) {
  fs.readFile('config.json', (err, data) => {
    if (err) {
      return callback(err);
    }
    callback(null, JSON.parse(data));
  });
}
```

### 4.2 Centralized Error Handling

```javascript
// ✅ DO: Create custom error classes
class AppError extends Error {
  constructor(message, statusCode, isOperational = true) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = isOperational;
    Error.captureStackTrace(this, this.constructor);
  }
}

class NotFoundError extends AppError {
  constructor(resource) {
    super(`${resource} not found`, 404);
  }
}

class ValidationError extends AppError {
  constructor(message) {
    super(message, 400);
  }
}
```

### 4.3 Global Error Handlers

```javascript
// ✅ DO: Handle uncaught exceptions and unhandled rejections
process.on('uncaughtException', (error) => {
  console.error('Uncaught Exception:', error);
  // Perform cleanup and exit
  process.exit(1);
});

process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection:', reason);
  // Log and continue or exit based on severity
});
```

### 4.4 Early Returns and Guard Clauses

```javascript
// ✅ DO: Use early returns for validation
async function processOrder(order) {
  if (!order) {
    throw new ValidationError('Order is required');
  }
  
  if (!order.items?.length) {
    throw new ValidationError('Order must have items');
  }
  
  if (order.status !== 'pending') {
    throw new ValidationError('Only pending orders can be processed');
  }
  
  // Happy path continues here
  return await executeOrder(order);
}
```

---

## 5. Security

### 5.1 Avoid Experimental Features in Production

```javascript
// ❌ DON'T: Use experimental features in production
// Experimental features may be unstable and subject to breaking changes

// ✅ DO: Check Node.js documentation for feature stability
// Use only stable APIs in production environments
```

### 5.2 Input Validation

```javascript
// ✅ DO: Always validate and sanitize user input
import { z } from 'zod';

const UserInputSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  age: z.number().int().positive().max(150),
});

function processUserInput(input) {
  const result = UserInputSchema.safeParse(input);
  if (!result.success) {
    throw new ValidationError(result.error.message);
  }
  return result.data;
}
```

### 5.3 Prevent Injection Attacks

```javascript
// ✅ DO: Use parameterized queries
const user = await db.query(
  'SELECT * FROM users WHERE id = $1',
  [userId]
);

// ❌ DON'T: String concatenation in queries
const user = await db.query(
  `SELECT * FROM users WHERE id = '${userId}'`
);
```

### 5.4 Secure HTTP Headers

```javascript
// ✅ DO: Set security headers
import helmet from 'helmet';

app.use(helmet());

// Or manually set critical headers
app.use((req, res, next) => {
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
  next();
});
```

### 5.5 Rate Limiting

```javascript
// ✅ DO: Implement rate limiting to prevent abuse
import rateLimit from 'express-rate-limit';

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP to 100 requests per windowMs
  message: 'Too many requests, please try again later.',
});

app.use('/api/', limiter);
```

### 5.6 Secure Environment Variables

```javascript
// ✅ DO: Never hardcode secrets
const apiKey = process.env.API_KEY;

// ✅ DO: Validate environment variables at startup
function validateEnv() {
  const required = ['DATABASE_URL', 'API_KEY', 'JWT_SECRET'];
  const missing = required.filter((key) => !process.env[key]);
  
  if (missing.length > 0) {
    throw new Error(`Missing required environment variables: ${missing.join(', ')}`);
  }
}
```

---

## 6. Module System

### 6.1 Prefer ES Modules Over CommonJS

```javascript
// ✅ DO: Use ES Modules (ESM)
import { createServer } from 'node:http';
import { readFile } from 'node:fs/promises';
import express from 'express';

export function myFunction() {}
export default class MyClass {}

// ❌ DON'T: Use CommonJS in new projects (unless required)
const http = require('http');
const fs = require('fs');
module.exports = myFunction;
```

### 6.2 Use Node Protocol for Built-in Modules

```javascript
// ✅ DO: Use 'node:' protocol prefix
import fs from 'node:fs';
import path from 'node:path';
import { createServer } from 'node:http';

// ❌ DON'T: Import without protocol (ambiguous)
import fs from 'fs';
```

### 6.3 Organize Imports

```javascript
// ✅ DO: Group and order imports
// 1. Built-in modules
import { createServer } from 'node:http';
import path from 'node:path';

// 2. External dependencies
import express from 'express';
import { z } from 'zod';

// 3. Internal modules
import { config } from './config.js';
import { UserService } from './services/user.js';

// 4. Types (if using TypeScript)
import type { Request, Response } from 'express';
```

---

## 7. Project Structure

### 7.1 Recommended Directory Structure

```
/project-root
├── src/
│   ├── config/           # Configuration files
│   │   └── index.js
│   ├── controllers/      # Route handlers
│   ├── middleware/       # Express middleware
│   ├── models/           # Data models
│   ├── routes/           # Route definitions
│   ├── services/         # Business logic
│   ├── utils/            # Utility functions
│   └── index.js          # Application entry point
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── scripts/              # Build/deploy scripts
├── .env.example          # Environment template
├── .gitignore
├── package.json
├── package-lock.json
└── README.md
```

### 7.2 Separation of Concerns

```javascript
// ✅ DO: Separate server setup from application logic

// server.js - HTTP server setup
import { createServer } from 'node:http';
import app from './app.js';

const server = createServer(app);
server.listen(process.env.PORT || 3000);

// app.js - Application configuration
import express from 'express';
import routes from './routes/index.js';

const app = express();
app.use(express.json());
app.use('/api', routes);
export default app;
```

---

## 8. Environment & Configuration

### 8.1 Environment Variables

```javascript
// ✅ DO: Load and validate environment variables early
import 'dotenv/config';

const config = {
  port: parseInt(process.env.PORT || '3000', 10),
  nodeEnv: process.env.NODE_ENV || 'development',
  database: {
    url: process.env.DATABASE_URL,
    poolSize: parseInt(process.env.DB_POOL_SIZE || '10', 10),
  },
  isDevelopment: process.env.NODE_ENV === 'development',
  isProduction: process.env.NODE_ENV === 'production',
};

export default Object.freeze(config);
```

### 8.2 Configuration Schema Validation

```javascript
// ✅ DO: Validate configuration at startup
import { z } from 'zod';

const ConfigSchema = z.object({
  PORT: z.string().regex(/^\d+$/).transform(Number),
  NODE_ENV: z.enum(['development', 'production', 'test']),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
});

export function loadConfig() {
  const result = ConfigSchema.safeParse(process.env);
  if (!result.success) {
    console.error('Invalid configuration:', result.error.format());
    process.exit(1);
  }
  return result.data;
}
```

---

## 9. HTTP & Networking

### 9.1 Basic HTTP Server

```javascript
import { createServer } from 'node:http';

const server = createServer((req, res) => {
  // Set appropriate headers
  res.setHeader('Content-Type', 'application/json');
  
  // Handle request
  if (req.method === 'GET' && req.url === '/health') {
    res.statusCode = 200;
    res.end(JSON.stringify({ status: 'healthy' }));
    return;
  }
  
  res.statusCode = 404;
  res.end(JSON.stringify({ error: 'Not found' }));
});

server.listen(3000, '127.0.0.1', () => {
  console.log('Server running at http://127.0.0.1:3000/');
});
```

### 9.2 Graceful Shutdown

```javascript
// ✅ DO: Implement graceful shutdown
const server = app.listen(port);

const shutdown = async (signal) => {
  console.log(`${signal} received, shutting down gracefully`);
  
  server.close(() => {
    console.log('HTTP server closed');
  });
  
  // Close database connections
  await db.close();
  
  // Close other resources
  process.exit(0);
};

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

### 9.3 Request Timeout Handling

```javascript
// ✅ DO: Set appropriate timeouts
server.setTimeout(30000); // 30 seconds

// Or per-request timeout
app.use((req, res, next) => {
  req.setTimeout(30000, () => {
    res.status(408).json({ error: 'Request timeout' });
  });
  next();
});
```

---

## 10. File System Operations

### 10.1 Use Promises API

```javascript
import { readFile, writeFile, mkdir } from 'node:fs/promises';
import { existsSync } from 'node:fs';
import path from 'node:path';

// ✅ DO: Use async file operations
async function saveUserData(userId, data) {
  const dirPath = path.join(process.cwd(), 'data', 'users');
  
  if (!existsSync(dirPath)) {
    await mkdir(dirPath, { recursive: true });
  }
  
  const filePath = path.join(dirPath, `${userId}.json`);
  await writeFile(filePath, JSON.stringify(data, null, 2), 'utf8');
}
```

### 10.2 Path Handling

```javascript
import path from 'node:path';
import { fileURLToPath } from 'node:url';

// ✅ DO: Use path module for cross-platform compatibility
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

const configPath = path.join(__dirname, '..', 'config', 'app.json');
const normalizedPath = path.normalize(userInputPath);

// ❌ DON'T: Concatenate paths with strings
const badPath = __dirname + '/config/app.json';
```

---

## 11. Streams & Buffers

### 11.1 Use Streams for Large Data

```javascript
import { createReadStream, createWriteStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';
import { createGzip } from 'node:zlib';

// ✅ DO: Use streams for large files
async function compressFile(input, output) {
  await pipeline(
    createReadStream(input),
    createGzip(),
    createWriteStream(output)
  );
}

// ✅ DO: Stream responses for large data
app.get('/download/:fileId', async (req, res) => {
  const filePath = await getFilePath(req.params.fileId);
  const stream = createReadStream(filePath);
  
  res.setHeader('Content-Type', 'application/octet-stream');
  stream.pipe(res);
});
```

### 11.2 Handle Backpressure

```javascript
// ✅ DO: Respect backpressure in streams
import { Readable, Writable } from 'node:stream';

const readable = getReadableStream();
const writable = getWritableStream();

readable.on('data', (chunk) => {
  const canContinue = writable.write(chunk);
  if (!canContinue) {
    readable.pause();
    writable.once('drain', () => readable.resume());
  }
});
```

---

## 12. Performance Optimization

### 12.1 Clustering for Multi-Core Systems

```javascript
import cluster from 'node:cluster';
import { availableParallelism } from 'node:os';
import process from 'node:process';

const numCPUs = availableParallelism();

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} is running`);
  
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
  
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died`);
    cluster.fork(); // Replace dead workers
  });
} else {
  // Workers can share any TCP connection
  startServer();
  console.log(`Worker ${process.pid} started`);
}
```

### 12.2 Caching Strategies

```javascript
// ✅ DO: Implement appropriate caching
const cache = new Map();
const CACHE_TTL = 60 * 1000; // 1 minute

async function getCachedData(key, fetchFn) {
  const cached = cache.get(key);
  
  if (cached && Date.now() - cached.timestamp < CACHE_TTL) {
    return cached.data;
  }
  
  const data = await fetchFn();
  cache.set(key, { data, timestamp: Date.now() });
  return data;
}
```

### 12.3 Memory Management

```javascript
// ✅ DO: Monitor memory usage
setInterval(() => {
  const used = process.memoryUsage();
  console.log({
    heapUsed: Math.round(used.heapUsed / 1024 / 1024) + ' MB',
    heapTotal: Math.round(used.heapTotal / 1024 / 1024) + ' MB',
    external: Math.round(used.external / 1024 / 1024) + ' MB',
  });
}, 30000);

// ✅ DO: Clear references when done
function processLargeData(data) {
  let results = performOperations(data);
  // Clear reference after use
  data = null;
  return results;
}
```

---

## 13. Logging & Monitoring

### 13.1 Structured Logging

```javascript
// ✅ DO: Use structured logging
import winston from 'winston';

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'api-service' },
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.simple(),
  }));
}

export default logger;
```

### 13.2 Request Logging

```javascript
// ✅ DO: Log HTTP requests with relevant details
import morgan from 'morgan';

// Development
app.use(morgan('dev'));

// Production - custom format
app.use(morgan(':method :url :status :res[content-length] - :response-time ms'));
```

### 13.3 Avoid Console in Production

```javascript
// ✅ DO: Use proper logger instead of console
logger.info('Server started', { port: 3000 });
logger.error('Database connection failed', { error: err.message });

// ❌ DON'T: Use console.log in production
console.log('Server started on port 3000');
```

---

## 14. Testing

### 14.1 Unit Testing

```javascript
// ✅ DO: Write testable code and comprehensive tests
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { UserService } from './user-service.js';

describe('UserService', () => {
  let userService;
  let mockRepository;
  
  beforeEach(() => {
    mockRepository = {
      findById: vi.fn(),
      save: vi.fn(),
    };
    userService = new UserService(mockRepository);
  });
  
  it('should return user when found', async () => {
    const mockUser = { id: '1', name: 'Test User' };
    mockRepository.findById.mockResolvedValue(mockUser);
    
    const result = await userService.getUser('1');
    
    expect(result).toEqual(mockUser);
    expect(mockRepository.findById).toHaveBeenCalledWith('1');
  });
  
  it('should throw when user not found', async () => {
    mockRepository.findById.mockResolvedValue(null);
    
    await expect(userService.getUser('999')).rejects.toThrow('User not found');
  });
});
```

### 14.2 Integration Testing

```javascript
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import request from 'supertest';
import app from './app.js';

describe('API Integration Tests', () => {
  let server;
  
  beforeAll(() => {
    server = app.listen(0);
  });
  
  afterAll(() => {
    server.close();
  });
  
  it('GET /api/health returns 200', async () => {
    const response = await request(app).get('/api/health');
    expect(response.status).toBe(200);
    expect(response.body).toHaveProperty('status', 'healthy');
  });
});
```

### 14.3 Use Node.js Built-in Test Runner

```javascript
// ✅ DO: Consider using Node.js built-in test runner (v18+)
import { test, describe, it } from 'node:test';
import assert from 'node:assert';

describe('Math operations', () => {
  it('should add numbers correctly', () => {
    assert.strictEqual(1 + 1, 2);
  });
});
```

---

## 15. Dependency Management

### 15.1 Lock File Best Practices

```bash
# ✅ DO: Always commit lock files
git add package-lock.json

# ✅ DO: Use npm ci in CI/CD environments
npm ci

# ❌ DON'T: Use npm install in CI (can update lock file)
npm install
```

### 15.2 Security Auditing

```bash
# ✅ DO: Regularly audit dependencies
npm audit

# ✅ DO: Fix vulnerabilities
npm audit fix

# ✅ DO: Check for outdated packages
npm outdated
```

### 15.3 Dependency Best Practices

```javascript
// ✅ DO: Pin major versions for stability
{
  "dependencies": {
    "express": "^4.18.0",  // Allows minor/patch updates
    "critical-lib": "4.18.0"  // Exact version for critical deps
  }
}

// ✅ DO: Separate dev dependencies
{
  "devDependencies": {
    "vitest": "^1.0.0",
    "eslint": "^9.0.0"
  }
}
```

---

## 16. Process Management

### 16.1 Handle Process Signals

```javascript
// ✅ DO: Handle termination signals
const signals = ['SIGTERM', 'SIGINT', 'SIGUSR2'];

signals.forEach((signal) => {
  process.on(signal, async () => {
    console.log(`Received ${signal}, shutting down...`);
    
    // Close server
    await new Promise((resolve) => server.close(resolve));
    
    // Close database connections
    await database.disconnect();
    
    // Exit cleanly
    process.exit(0);
  });
});
```

### 16.2 Child Processes

```javascript
import { spawn, exec } from 'node:child_process';
import { promisify } from 'node:util';

const execAsync = promisify(exec);

// ✅ DO: Use spawn for long-running processes
const child = spawn('node', ['worker.js'], {
  stdio: 'inherit',
  detached: false,
});

// ✅ DO: Use exec for simple commands
const { stdout, stderr } = await execAsync('git status');
```

---

## 17. Documentation

### 17.1 JSDoc Comments

```javascript
/**
 * Fetches user data from the database by ID.
 * @param {string} userId - The unique identifier of the user.
 * @param {Object} [options] - Optional parameters.
 * @param {boolean} [options.includeProfile=false] - Whether to include profile data.
 * @returns {Promise<User>} The user object.
 * @throws {NotFoundError} When user is not found.
 * @example
 * const user = await getUserById('123', { includeProfile: true });
 */
async function getUserById(userId, options = {}) {
  // Implementation
}
```

### 17.2 README Requirements

Every Node.js project should include a README with:

- Project description
- Prerequisites and dependencies
- Installation instructions
- Configuration (environment variables)
- Usage examples
- API documentation (if applicable)
- Testing instructions
- Deployment instructions

---

## Quick Reference Checklist

### Before Committing Code

- [ ] No synchronous I/O operations in request handlers
- [ ] All async functions have proper error handling
- [ ] User input is validated and sanitized
- [ ] No sensitive data in logs or error messages
- [ ] Environment variables are used for configuration
- [ ] Dependencies are up to date and audited
- [ ] Tests pass and coverage is adequate
- [ ] Code follows ESLint rules
- [ ] No console.log statements (use proper logger)
- [ ] Graceful shutdown is implemented

### Security Checklist

- [ ] Input validation with schema validation (Zod)
- [ ] Parameterized queries (no SQL injection)
- [ ] Security headers configured (Helmet)
- [ ] Rate limiting implemented
- [ ] HTTPS enforced in production
- [ ] No secrets in codebase
- [ ] Dependencies audited for vulnerabilities
- [ ] No experimental features in production

---

> **Last Updated**: November 2024
> 
> **Node.js Version**: v20.x LTS / v22.x
> 
> **References**:
> - [Node.js Official Documentation](https://nodejs.org/docs/latest/api/)
> - [Node.js Best Practices](https://nodejs.org/en/learn/getting-started/introduction-to-nodejs)
> - [Node.js Security Best Practices](https://nodejs.org/en/learn/getting-started/security-best-practices)

