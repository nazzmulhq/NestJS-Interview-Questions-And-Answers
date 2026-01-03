# Node.js Advanced Interview Questions - Part 2 (Section 22-30)

## 22. Debugger এবং Inspector

### Built-in Debugger

```javascript
// Start with: node inspect app.js

function calculate(a, b) {
	debugger; // Breakpoint
	const sum = a + b;
	const product = a * b;
	return { sum, product };
}

const result = calculate(5, 10);
console.log(result);

// Debugger commands:
// cont, c - Continue execution
// next, n - Step next
// step, s - Step in
// out, o - Step out
// pause - Pause running code
// watch('expr') - Watch expression
// setBreakpoint(), sb() - Set breakpoint
```

### Inspector Protocol

```javascript
const inspector = require("inspector");
const fs = require("fs");

// Start inspector session
const session = new inspector.Session();
session.connect();

// CPU Profiling
function startProfiling() {
	return new Promise((resolve) => {
		session.post("Profiler.enable", () => {
			session.post("Profiler.start", resolve);
		});
	});
}

function stopProfiling() {
	return new Promise((resolve) => {
		session.post("Profiler.stop", (err, { profile }) => {
			if (err) throw err;
			resolve(profile);
		});
	});
}

async function profileFunction(fn) {
	await startProfiling();
	await fn();
	const profile = await stopProfiling();

	// Save profile
	fs.writeFileSync("profile.cpuprofile", JSON.stringify(profile));
	console.log("Profile saved to profile.cpuprofile");

	session.disconnect();
}

// Heap Snapshot
function takeHeapSnapshot() {
	return new Promise((resolve) => {
		session.post("HeapProfiler.enable", () => {
			session.post("HeapProfiler.takeHeapSnapshot", null, (err, snapshot) => {
				fs.writeFileSync("heap.heapsnapshot", JSON.stringify(snapshot));
				resolve();
			});
		});
	});
}

// Coverage
async function collectCoverage(fn) {
	session.post("Profiler.enable");
	session.post("Profiler.startPreciseCoverage", {
		callCount: true,
		detailed: true,
	});

	await fn();

	return new Promise((resolve) => {
		session.post("Profiler.takePreciseCoverage", (err, { result }) => {
			session.post("Profiler.stopPreciseCoverage");
			resolve(result);
		});
	});
}
```

### Remote Debugging

```javascript
// Start app with: node --inspect=0.0.0.0:9229 app.js

const { promisify } = require("util");
const { execFile } = require("child_process");
const execFileAsync = promisify(execFile);

async function debugRemote() {
	// Connect to remote debugger
	const response = await fetch("http://localhost:9229/json");
	const targets = await response.json();

	const debuggerUrl = targets[0].webSocketDebuggerUrl;
	console.log("Debugger URL:", debuggerUrl);

	// Use WebSocket to connect
	const WebSocket = require("ws");
	const ws = new WebSocket(debuggerUrl);

	ws.on("open", () => {
		// Enable debugger
		ws.send(
			JSON.stringify({
				id: 1,
				method: "Debugger.enable",
			}),
		);

		// Set breakpoint
		ws.send(
			JSON.stringify({
				id: 2,
				method: "Debugger.setBreakpointByUrl",
				params: {
					lineNumber: 10,
					url: "file:///app.js",
				},
			}),
		);
	});

	ws.on("message", (data) => {
		const message = JSON.parse(data);
		console.log("Debug event:", message);
	});
}
```

### Debug Utilities

```javascript
// Debug logging with namespaces
const debug = require("debug");

const logServer = debug("app:server");
const logDatabase = debug("app:database");
const logAuth = debug("app:auth");

logServer("Server starting on port %d", 3000);
logDatabase("Connected to database");
logAuth("User %s authenticated", "john");

// Enable debug: DEBUG=app:* node app.js

// Custom debug utility
class Debug {
	constructor(namespace) {
		this.namespace = namespace;
		this.enabled = process.env.DEBUG?.includes(namespace);
	}

	log(...args) {
		if (this.enabled) {
			console.log(`[${this.namespace}]`, ...args);
		}
	}

	error(...args) {
		if (this.enabled) {
			console.error(`[${this.namespace}] ERROR:`, ...args);
		}
	}

	time(label) {
		if (this.enabled) {
			console.time(`[${this.namespace}] ${label}`);
		}
	}

	timeEnd(label) {
		if (this.enabled) {
			console.timeEnd(`[${this.namespace}] ${label}`);
		}
	}
}

// ব্যবহার
const debug = new Debug("myapp:service");
debug.log("Processing request");
debug.time("query");
// ... do work
debug.timeEnd("query");
```

---

## 23. Performance Profiling

### CPU Profiling

```javascript
const { Session } = require("inspector");
const fs = require("fs");

class CPUProfiler {
	constructor() {
		this.session = new Session();
		this.session.connect();
	}

	start() {
		return new Promise((resolve, reject) => {
			this.session.post("Profiler.enable", (err) => {
				if (err) return reject(err);
				this.session.post("Profiler.start", (err) => {
					if (err) return reject(err);
					resolve();
				});
			});
		});
	}

	stop() {
		return new Promise((resolve, reject) => {
			this.session.post("Profiler.stop", (err, { profile }) => {
				if (err) return reject(err);
				resolve(profile);
			});
		});
	}

	async profile(fn, filename = "profile.cpuprofile") {
		await this.start();

		try {
			await fn();
		} finally {
			const profile = await this.stop();
			fs.writeFileSync(filename, JSON.stringify(profile));
			console.log(`Profile saved to ${filename}`);
			this.session.disconnect();
		}
	}
}

// ব্যবহার
const profiler = new CPUProfiler();

await profiler.profile(async () => {
	// Code to profile
	for (let i = 0; i < 1000000; i++) {
		Math.sqrt(i);
	}
});

// View in Chrome DevTools: chrome://inspect
```

### Memory Profiling

```javascript
const v8 = require("v8");
const fs = require("fs");

class MemoryProfiler {
	static takeSnapshot(filename = `heap-${Date.now()}.heapsnapshot`) {
		const snapshot = v8.writeHeapSnapshot(filename);
		console.log(`Heap snapshot saved to ${snapshot}`);
		return snapshot;
	}

	static getHeapStats() {
		const stats = v8.getHeapStatistics();
		return {
			totalHeapSize: (stats.total_heap_size / 1024 / 1024).toFixed(2) + " MB",
			usedHeapSize: (stats.used_heap_size / 1024 / 1024).toFixed(2) + " MB",
			heapSizeLimit: (stats.heap_size_limit / 1024 / 1024).toFixed(2) + " MB",
			mallocedMemory: (stats.malloced_memory / 1024 / 1024).toFixed(2) + " MB",
			peakMallocedMemory: (stats.peak_malloced_memory / 1024 / 1024).toFixed(2) + " MB",
		};
	}

	static trackMemoryUsage(interval = 5000) {
		const measurements = [];

		const timer = setInterval(() => {
			const usage = process.memoryUsage();
			measurements.push({
				timestamp: Date.now(),
				rss: usage.rss,
				heapTotal: usage.heapTotal,
				heapUsed: usage.heapUsed,
				external: usage.external,
			});

			console.log("Memory:", {
				rss: (usage.rss / 1024 / 1024).toFixed(2) + " MB",
				heapUsed: (usage.heapUsed / 1024 / 1024).toFixed(2) + " MB",
			});
		}, interval);

		return {
			stop: () => {
				clearInterval(timer);
				return measurements;
			},
		};
	}

	static async detectMemoryLeak(fn, iterations = 100) {
		const samples = [];

		for (let i = 0; i < iterations; i++) {
			await fn();

			if (i % 10 === 0) {
				global.gc && global.gc(); // Run with --expose-gc
				const usage = process.memoryUsage();
				samples.push(usage.heapUsed);
			}
		}

		// Analyze trend
		const firstHalf = samples.slice(0, samples.length / 2);
		const secondHalf = samples.slice(samples.length / 2);

		const avgFirst = firstHalf.reduce((a, b) => a + b) / firstHalf.length;
		const avgSecond = secondHalf.reduce((a, b) => a + b) / secondHalf.length;

		const increase = ((avgSecond - avgFirst) / avgFirst) * 100;

		return {
			samples,
			avgFirst: (avgFirst / 1024 / 1024).toFixed(2) + " MB",
			avgSecond: (avgSecond / 1024 / 1024).toFixed(2) + " MB",
			increase: increase.toFixed(2) + "%",
			possibleLeak: increase > 10, // More than 10% increase
		};
	}
}

// ব্যবহার
console.log("Heap stats:", MemoryProfiler.getHeapStats());

// Track memory over time
const tracker = MemoryProfiler.trackMemoryUsage(1000);
setTimeout(() => {
	const measurements = tracker.stop();
	console.log("Collected", measurements.length, "measurements");
}, 10000);

// Detect memory leak
const leakTest = await MemoryProfiler.detectMemoryLeak(async () => {
	// Potentially leaky code
	const data = new Array(10000).fill("x");
});

console.log("Leak test:", leakTest);
```

### Benchmarking

```javascript
const { performance } = require("perf_hooks");

class Benchmark {
	constructor(name) {
		this.name = name;
		this.results = [];
	}

	async run(fn, iterations = 1000) {
		const times = [];

		// Warmup
		for (let i = 0; i < 10; i++) {
			await fn();
		}

		// Measure
		for (let i = 0; i < iterations; i++) {
			const start = performance.now();
			await fn();
			const end = performance.now();
			times.push(end - start);
		}

		const sorted = times.sort((a, b) => a - b);
		const sum = times.reduce((a, b) => a + b, 0);

		this.results.push({
			name: this.name,
			iterations,
			mean: sum / times.length,
			median: sorted[Math.floor(sorted.length / 2)],
			min: sorted[0],
			max: sorted[sorted.length - 1],
			p95: sorted[Math.floor(sorted.length * 0.95)],
			p99: sorted[Math.floor(sorted.length * 0.99)],
		});

		return this.results[this.results.length - 1];
	}

	compare(benchmarks) {
		const results = benchmarks.map((b) => b.results[0]);
		results.sort((a, b) => a.mean - b.mean);

		console.log("\nBenchmark Results:");
		console.log("=".repeat(80));

		results.forEach((result, index) => {
			const relative =
				index === 0 ? "baseline" : `${(result.mean / results[0].mean).toFixed(2)}x slower`;

			console.log(`${result.name}:`);
			console.log(`  Mean: ${result.mean.toFixed(4)}ms (${relative})`);
			console.log(`  Median: ${result.median.toFixed(4)}ms`);
			console.log(`  Min: ${result.min.toFixed(4)}ms, Max: ${result.max.toFixed(4)}ms`);
			console.log(`  P95: ${result.p95.toFixed(4)}ms, P99: ${result.p99.toFixed(4)}ms`);
			console.log();
		});
	}
}

// ব্যবহার
async function benchmarkExample() {
	const b1 = new Benchmark("Array.push");
	await b1.run(() => {
		const arr = [];
		for (let i = 0; i < 1000; i++) {
			arr.push(i);
		}
	});

	const b2 = new Benchmark("Array.concat");
	await b2.run(() => {
		let arr = [];
		for (let i = 0; i < 1000; i++) {
			arr = arr.concat(i);
		}
	});

	const b3 = new Benchmark("Array spread");
	await b3.run(() => {
		let arr = [];
		for (let i = 0; i < 1000; i++) {
			arr = [...arr, i];
		}
	});

	Benchmark.prototype.compare([b1, b2, b3]);
}

benchmarkExample();
```

---

## 24. Environment Variables

### Environment Variables Management

```javascript
// .env file
/*
NODE_ENV=production
PORT=3000
DB_HOST=localhost
DB_PORT=5432
DB_NAME=mydb
DB_USER=admin
DB_PASSWORD=secret
JWT_SECRET=my-secret-key
API_KEY=abc123
*/

// Load environment variables
require("dotenv").config();

// Type-safe environment
class Environment {
	static get(key, defaultValue) {
		const value = process.env[key];
		if (value === undefined && defaultValue === undefined) {
			throw new Error(`Environment variable ${key} is required`);
		}
		return value || defaultValue;
	}

	static getNumber(key, defaultValue) {
		const value = this.get(key, defaultValue?.toString());
		const num = parseInt(value, 10);
		if (isNaN(num)) {
			throw new Error(`Environment variable ${key} must be a number`);
		}
		return num;
	}

	static getBoolean(key, defaultValue = false) {
		const value = this.get(key, defaultValue.toString());
		return value === "true" || value === "1";
	}

	static getArray(key, separator = ",", defaultValue = []) {
		const value = this.get(key, defaultValue.join(separator));
		return value.split(separator).map((v) => v.trim());
	}

	static validate(schema) {
		const errors = [];

		for (const [key, validator] of Object.entries(schema)) {
			try {
				validator(process.env[key]);
			} catch (error) {
				errors.push(`${key}: ${error.message}`);
			}
		}

		if (errors.length > 0) {
			throw new Error(`Environment validation failed:\n${errors.join("\n")}`);
		}
	}
}

// Configuration class
class Config {
	static load() {
		// Validate required variables
		Environment.validate({
			NODE_ENV: (val) => {
				if (!["development", "production", "test"].includes(val)) {
					throw new Error("Must be development, production, or test");
				}
			},
			PORT: (val) => {
				const port = parseInt(val);
				if (isNaN(port) || port < 1 || port > 65535) {
					throw new Error("Must be a valid port number");
				}
			},
			DB_HOST: (val) => {
				if (!val) throw new Error("Database host is required");
			},
		});

		return {
			env: Environment.get("NODE_ENV"),
			port: Environment.getNumber("PORT", 3000),
			database: {
				host: Environment.get("DB_HOST"),
				port: Environment.getNumber("DB_PORT", 5432),
				name: Environment.get("DB_NAME"),
				user: Environment.get("DB_USER"),
				password: Environment.get("DB_PASSWORD"),
			},
			jwt: {
				secret: Environment.get("JWT_SECRET"),
				expiresIn: Environment.get("JWT_EXPIRES_IN", "1h"),
			},
			api: {
				key: Environment.get("API_KEY"),
				timeout: Environment.getNumber("API_TIMEOUT", 5000),
			},
			features: {
				enableCache: Environment.getBoolean("ENABLE_CACHE", true),
				enableMetrics: Environment.getBoolean("ENABLE_METRICS", false),
			},
			allowedOrigins: Environment.getArray("ALLOWED_ORIGINS", ",", ["http://localhost:3000"]),
		};
	}

	static get isDevelopment() {
		return process.env.NODE_ENV === "development";
	}

	static get isProduction() {
		return process.env.NODE_ENV === "production";
	}

	static get isTest() {
		return process.env.NODE_ENV === "test";
	}
}

// ব্যবহার
try {
	const config = Config.load();
	console.log("Configuration loaded:", config);
} catch (error) {
	console.error("Configuration error:", error.message);
	process.exit(1);
}
```

---

## 25. Error Handling Strategies

### Custom Error Classes

```javascript
// Base error class
class AppError extends Error {
	constructor(message, statusCode = 500, isOperational = true) {
		super(message);
		this.statusCode = statusCode;
		this.isOperational = isOperational;
		this.timestamp = new Date().toISOString();

		Error.captureStackTrace(this, this.constructor);
	}

	toJSON() {
		return {
			error: {
				message: this.message,
				statusCode: this.statusCode,
				timestamp: this.timestamp,
			},
		};
	}
}

// Specific error types
class ValidationError extends AppError {
	constructor(message, errors = []) {
		super(message, 400);
		this.name = "ValidationError";
		this.errors = errors;
	}

	toJSON() {
		return {
			error: {
				message: this.message,
				statusCode: this.statusCode,
				errors: this.errors,
			},
		};
	}
}

class NotFoundError extends AppError {
	constructor(resource) {
		super(`${resource} not found`, 404);
		this.name = "NotFoundError";
		this.resource = resource;
	}
}

class UnauthorizedError extends AppError {
	constructor(message = "Unauthorized") {
		super(message, 401);
		this.name = "UnauthorizedError";
	}
}

class DatabaseError extends AppError {
	constructor(message, originalError) {
		super(message, 500, false);
		this.name = "DatabaseError";
		this.originalError = originalError;
	}
}
```

### Error Handler Middleware

```javascript
// Express error handler
function errorHandler(err, req, res, next) {
	// Log error
	logger.error({
		message: err.message,
		stack: err.stack,
		url: req.url,
		method: req.method,
		ip: req.ip,
		user: req.user?.id,
	});

	// Operational errors
	if (err.isOperational) {
		return res.status(err.statusCode).json(err.toJSON());
	}

	// Programming errors
	if (process.env.NODE_ENV === "development") {
		return res.status(500).json({
			error: {
				message: err.message,
				stack: err.stack,
			},
		});
	}

	// Production - don't leak error details
	return res.status(500).json({
		error: {
			message: "Internal server error",
		},
	});
}

// Async error wrapper
function asyncHandler(fn) {
	return (req, res, next) => {
		Promise.resolve(fn(req, res, next)).catch(next);
	};
}
```

### Circuit Breaker Pattern

```javascript
class CircuitBreaker {
	constructor(options = {}) {
		this.failureThreshold = options.failureThreshold || 5;
		this.resetTimeout = options.resetTimeout || 60000;
		this.state = "CLOSED"; // CLOSED, OPEN, HALF_OPEN
		this.failures = 0;
		this.nextAttempt = Date.now();
	}

	async execute(fn) {
		if (this.state === "OPEN") {
			if (Date.now() < this.nextAttempt) {
				throw new Error("Circuit breaker is OPEN");
			}
			this.state = "HALF_OPEN";
		}

		try {
			const result = await fn();
			this.onSuccess();
			return result;
		} catch (error) {
			this.onFailure();
			throw error;
		}
	}

	onSuccess() {
		this.failures = 0;

		if (this.state === "HALF_OPEN") {
			this.state = "CLOSED";
			console.log("Circuit breaker: HALF_OPEN -> CLOSED");
		}
	}

	onFailure() {
		this.failures++;

		if (this.failures >= this.failureThreshold) {
			this.state = "OPEN";
			this.nextAttempt = Date.now() + this.resetTimeout;
			console.log(`Circuit breaker: OPEN (failures: ${this.failures})`);
		}
	}

	getState() {
		return this.state;
	}
}

// ব্যবহার
const breaker = new CircuitBreaker({
	failureThreshold: 3,
	resetTimeout: 30000,
});

async function callExternalAPI() {
	return await breaker.execute(async () => {
		const response = await fetch("https://api.example.com/data");
		return await response.json();
	});
}
```

---

## 26. Monitoring এবং Logging

### Winston Logger

```javascript
const winston = require("winston");

// Logger configuration
const logger = winston.createLogger({
	level: process.env.LOG_LEVEL || "info",
	format: winston.format.combine(
		winston.format.timestamp(),
		winston.format.errors({ stack: true }),
		winston.format.json(),
	),
	defaultMeta: { service: "my-app" },
	transports: [
		new winston.transports.File({
			filename: "logs/error.log",
			level: "error",
		}),
		new winston.transports.File({
			filename: "logs/combined.log",
		}),
	],
});

// Console logging in development
if (process.env.NODE_ENV !== "production") {
	logger.add(
		new winston.transports.Console({
			format: winston.format.combine(winston.format.colorize(), winston.format.simple()),
		}),
	);
}

// HTTP request logging
function loggerMiddleware(req, res, next) {
	const start = Date.now();

	res.on("finish", () => {
		const duration = Date.now() - start;

		logger.info({
			method: req.method,
			url: req.url,
			status: res.statusCode,
			duration: `${duration}ms`,
			ip: req.ip,
			userAgent: req.get("user-agent"),
		});
	});

	next();
}

// ব্যবহার
logger.info("Server started", { port: 3000 });
logger.error("Database connection failed", { error: err.message });
logger.warn("API rate limit exceeded", { ip: req.ip });
```

### Application Metrics

```javascript
const promClient = require("prom-client");

// Create Registry
const register = new promClient.Registry();

// Add default metrics
promClient.collectDefaultMetrics({ register });

// Custom metrics
const httpRequestDuration = new promClient.Histogram({
	name: "http_request_duration_seconds",
	help: "Duration of HTTP requests in seconds",
	labelNames: ["method", "route", "status"],
	buckets: [0.1, 0.5, 1, 2, 5],
});

const httpRequestTotal = new promClient.Counter({
	name: "http_requests_total",
	help: "Total number of HTTP requests",
	labelNames: ["method", "route", "status"],
});

const activeConnections = new promClient.Gauge({
	name: "active_connections",
	help: "Number of active connections",
});

register.registerMetric(httpRequestDuration);
register.registerMetric(httpRequestTotal);
register.registerMetric(activeConnections);

// Metrics middleware
function metricsMiddleware(req, res, next) {
	const start = Date.now();

	res.on("finish", () => {
		const duration = (Date.now() - start) / 1000;

		httpRequestDuration
			.labels(req.method, req.route?.path || req.path, res.statusCode)
			.observe(duration);

		httpRequestTotal.labels(req.method, req.route?.path || req.path, res.statusCode).inc();
	});

	next();
}

// Metrics endpoint
app.get("/metrics", async (req, res) => {
	res.set("Content-Type", register.contentType);
	res.end(await register.metrics());
});
```

---

## 27. Security Best Practices

### Input Validation and Sanitization

```javascript
const validator = require("validator");

class InputValidator {
	static validateEmail(email) {
		if (!validator.isEmail(email)) {
			throw new ValidationError("Invalid email address");
		}
		return validator.normalizeEmail(email);
	}

	static validatePassword(password) {
		const errors = [];

		if (password.length < 8) {
			errors.push("Password must be at least 8 characters");
		}

		if (!/[A-Z]/.test(password)) {
			errors.push("Password must contain uppercase letter");
		}

		if (!/[a-z]/.test(password)) {
			errors.push("Password must contain lowercase letter");
		}

		if (!/[0-9]/.test(password)) {
			errors.push("Password must contain number");
		}

		if (errors.length > 0) {
			throw new ValidationError("Password validation failed", errors);
		}

		return password;
	}

	static sanitizeString(str, maxLength = 255) {
		return validator.escape(validator.trim(str)).slice(0, maxLength);
	}

	static validateURL(url) {
		if (!validator.isURL(url, { require_protocol: true })) {
			throw new ValidationError("Invalid URL");
		}
		return url;
	}
}
```

### Rate Limiting

```javascript
class RateLimiter {
	constructor(options = {}) {
		this.windowMs = options.windowMs || 60000;
		this.maxRequests = options.maxRequests || 100;
		this.storage = new Map();
	}

	middleware() {
		return (req, res, next) => {
			const key = this.getKey(req);
			const now = Date.now();

			// Get or create bucket
			let bucket = this.storage.get(key);

			if (!bucket) {
				bucket = {
					count: 0,
					resetTime: now + this.windowMs,
				};
				this.storage.set(key, bucket);
			}

			// Reset if window expired
			if (now > bucket.resetTime) {
				bucket.count = 0;
				bucket.resetTime = now + this.windowMs;
			}

			bucket.count++;

			// Set headers
			res.set("X-RateLimit-Limit", this.maxRequests);
			res.set("X-RateLimit-Remaining", Math.max(0, this.maxRequests - bucket.count));
			res.set("X-RateLimit-Reset", new Date(bucket.resetTime).toISOString());

			// Check limit
			if (bucket.count > this.maxRequests) {
				res.status(429).json({
					error: "Too many requests",
				});
				return;
			}

			next();
		};
	}

	getKey(req) {
		return req.ip || req.connection.remoteAddress;
	}

	cleanup() {
		const now = Date.now();
		for (const [key, bucket] of this.storage.entries()) {
			if (now > bucket.resetTime + this.windowMs) {
				this.storage.delete(key);
			}
		}
	}
}

// ব্যবহার
const limiter = new RateLimiter({
	windowMs: 60000, // 1 minute
	maxRequests: 100,
});

app.use("/api", limiter.middleware());

// Cleanup old entries periodically
setInterval(() => limiter.cleanup(), 60000);
```

### SQL Injection Prevention

```javascript
// ❌ খারাপ - SQL Injection vulnerable
function getUserBad(userId) {
	return db.query(`SELECT * FROM users WHERE id = ${userId}`);
}

// ✅ ভালো - Parameterized queries
function getUserGood(userId) {
	return db.query("SELECT * FROM users WHERE id = ?", [userId]);
}

// ✅ ভালো - Query builder (Knex.js)
function getUserKnex(userId) {
	return db("users").where("id", userId).first();
}
```

### XSS Prevention

```javascript
// Output encoding
function escapeHTML(str) {
	return str
		.replace(/&/g, "&amp;")
		.replace(/</g, "&lt;")
		.replace(/>/g, "&gt;")
		.replace(/"/g, "&quot;")
		.replace(/'/g, "&#039;");
}

// Content Security Policy
app.use((req, res, next) => {
	res.setHeader(
		"Content-Security-Policy",
		"default-src 'self'; " +
			"script-src 'self' 'unsafe-inline'; " +
			"style-src 'self' 'unsafe-inline'; " +
			"img-src 'self' data: https:;",
	);
	next();
});
```

---

## 28. Single Executable Applications

### Creating Standalone Executables

```javascript
// package.json
{
  "name": "my-app",
  "version": "1.0.0",
  "bin": {
    "myapp": "./cli.js"
  },
  "scripts": {
    "build": "pkg . --targets node18-linux-x64,node18-win-x64,node18-macos-x64"
  },
  "pkg": {
    "assets": ["config/**/*", "templates/**/*"],
    "outputPath": "dist"
  }
}

// cli.js
#!/usr/bin/env node

const { program } = require("commander");

program
 .name("myapp")
 .description("My CLI application")
 .version("1.0.0");

program
 .command("start")
 .description("Start the application")
 .option("-p, --port <number>", "Port number", "3000")
 .action((options) => {
  console.log(`Starting on port ${options.port}`);
  // Start your app
 });

program
 .command("config")
 .description("Configure application")
 .option("-s, --set <key=value>", "Set configuration")
 .action((options) => {
  // Handle configuration
 });

program.parse();
```

### Node.js SEA (Single Executable Application)

```javascript
// Node 20+ built-in SEA
// 1. Create your app
// app.js
console.log("Hello from SEA!");
console.log("Args:", process.argv);

// 2. Create sea-config.json
{
  "main": "app.js",
  "output": "sea-prep.blob",
  "disableExperimentalSEAWarning": true
}

// 3. Build commands
// Generate blob
node --experimental-sea-config sea-config.json

// Copy node binary
cp $(command -v node) myapp

// Inject blob (Linux/Mac)
npx postject myapp NODE_SEA_BLOB sea-prep.blob \
    --sentinel-fuse NODE_SEA_FUSE_fce680ab2cc467b6e072b8b5df1996b2

// Make executable
chmod +x myapp

// Run
./myapp
```

---

## 29. WASI (WebAssembly System Interface)

### Using WASI in Node.js

```javascript
const { WASI } = require("wasi");
const fs = require("fs");

// Initialize WASI
const wasi = new WASI({
	args: process.argv,
	env: process.env,
	preopens: {
		"/sandbox": "/tmp",
	},
});

// Load WASM module
const wasmBuffer = fs.readFileSync("./module.wasm");

WebAssembly.instantiate(wasmBuffer, wasi.getImportObject()).then((result) => {
	wasi.start(result.instance);
});
```

### Rust to WASM Example

```rust
// lib.rs
#[no_mangle]
pub extern "C" fn fibonacci(n: i32) -> i32 {
    if n <= 1 {
        return n;
    }
    fibonacci(n - 1) + fibonacci(n - 2)
}

// Build: cargo build --target wasm32-wasi --release
```

```javascript
// Using in Node.js
const { WASI } = require("wasi");
const fs = require("fs");

async function runWasm() {
	const wasi = new WASI();

	const wasm = await WebAssembly.compile(
		fs.readFileSync("./target/wasm32-wasi/release/mylib.wasm"),
	);

	const instance = await WebAssembly.instantiate(wasm, wasi.getImportObject());

	// Call exported function
	const result = instance.exports.fibonacci(10);
	console.log("Fibonacci(10) =", result);
}

runWasm();
```

---

## 30. Web Crypto API

### Encryption and Decryption

```javascript
const { webcrypto } = require("crypto");
const { subtle } = webcrypto;

// Generate key
async function generateKey() {
	return await subtle.generateKey(
		{
			name: "AES-GCM",
			length: 256,
		},
		true,
		["encrypt", "decrypt"],
	);
}

// Encrypt data
async function encrypt(key, data) {
	const iv = webcrypto.getRandomValues(new Uint8Array(12));
	const encodedData = new TextEncoder().encode(data);

	const encryptedData = await subtle.encrypt(
		{
			name: "AES-GCM",
			iv,
		},
		key,
		encodedData,
	);

	return {
		encrypted: Buffer.from(encryptedData).toString("base64"),
		iv: Buffer.from(iv).toString("base64"),
	};
}

// Decrypt data
async function decrypt(key, encryptedData, iv) {
	const decryptedData = await subtle.decrypt(
		{
			name: "AES-GCM",
			iv: Buffer.from(iv, "base64"),
		},
		key,
		Buffer.from(encryptedData, "base64"),
	);

	return new TextDecoder().decode(decryptedData);
}

// ব্যবহার
(async () => {
	const key = await generateKey();
	const { encrypted, iv } = await encrypt(key, "Secret message");
	console.log("Encrypted:", encrypted);

	const decrypted = await decrypt(key, encrypted, iv);
	console.log("Decrypted:", decrypted);
})();
```

### Digital Signatures

```javascript
// Generate key pair
async function generateKeyPair() {
	return await subtle.generateKey(
		{
			name: "ECDSA",
			namedCurve: "P-384",
		},
		true,
		["sign", "verify"],
	);
}

// Sign data
async function sign(privateKey, data) {
	const encodedData = new TextEncoder().encode(data);

	const signature = await subtle.sign(
		{
			name: "ECDSA",
			hash: "SHA-384",
		},
		privateKey,
		encodedData,
	);

	return Buffer.from(signature).toString("base64");
}

// Verify signature
async function verify(publicKey, signature, data) {
	const encodedData = new TextEncoder().encode(data);

	return await subtle.verify(
		{
			name: "ECDSA",
			hash: "SHA-384",
		},
		publicKey,
		Buffer.from(signature, "base64"),
		encodedData,
	);
}

// ব্যবহার
(async () => {
	const { privateKey, publicKey } = await generateKeyPair();

	const message = "Important document";
	const signature = await sign(privateKey, message);
	console.log("Signature:", signature);

	const isValid = await verify(publicKey, signature, message);
	console.log("Signature valid:", isValid);
})();
```

### Hashing

```javascript
// SHA-256 hash
async function hash(data) {
	const encodedData = new TextEncoder().encode(data);

	const hashBuffer = await subtle.digest("SHA-256", encodedData);

	return Buffer.from(hashBuffer).toString("hex");
}

// PBKDF2 key derivation
async function deriveKey(password, salt) {
	const encodedPassword = new TextEncoder().encode(password);

	const keyMaterial = await subtle.importKey("raw", encodedPassword, "PBKDF2", false, [
		"deriveBits",
		"deriveKey",
	]);

	return await subtle.deriveKey(
		{
			name: "PBKDF2",
			salt: new TextEncoder().encode(salt),
			iterations: 100000,
			hash: "SHA-256",
		},
		keyMaterial,
		{
			name: "AES-GCM",
			length: 256,
		},
		true,
		["encrypt", "decrypt"],
	);
}

// ব্যবহার
(async () => {
	const hashed = await hash("Hello World");
	console.log("Hash:", hashed);

	const key = await deriveKey("mypassword", "randomsalt");
	console.log("Derived key generated");
})();
```

---

## শেষ কথা

এই সম্পূর্ণ ডকুমেন্টেশনে Node.js এর ৩০টি অ্যাডভান্সড টপিক কভার করা হয়েছে যা ৮+ বছরের অভিজ্ঞতাসম্পন্ন ডেভেলপারদের জন্য গুরুত্বপূর্ণ। প্রতিটি টপিকে রিয়েল-ওয়ার্ল্ড কোড উদাহরণ এবং বেস্ট প্র্যাকটিস অন্তর্ভুক্ত করা হয়েছে।

### প্রধান টপিক সমূহ

1. **Core Concepts**: Event Loop, V8 Engine, Process Management
2. **Concurrency**: Worker Threads, Cluster Mode, Child Processes
3. **I/O**: Streams, File System, Buffer Operations
4. **Networking**: HTTP/2, WebSocket, TCP/UDP
5. **Database**: SQLite, Connection Pooling
6. **Testing**: Test Runner, Mocking, Profiling
7. **Production**: Error Handling, Monitoring, Security
8. **Advanced**: SEA, WASI, Web Crypto API

### অতিরিক্ত রিসোর্স

-   [Node.js Official Documentation](https://nodejs.org/docs/latest/api/)
-   [Node.js Best Practices](https://github.com/goldbergyoni/nodebestpractices)
-   [Node.js Design Patterns](https://www.nodejsdesignpatterns.com/)

সফল হোক আপনার ইন্টারভিউ! 🚀
