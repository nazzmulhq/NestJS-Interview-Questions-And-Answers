# Node.js Advanced Interview Questions (৮+ বছরের অভিজ্ঞতার জন্য)

> Node.js ইন্টারভিউ প্রশ্ন এবং উত্তর - বাংলা সংস্করণ

---

## সূচিপত্র

### Core Concepts

1. [Event Loop এবং Non-blocking I/O](#1-event-loop-এবং-non-blocking-io)
2. [V8 Engine এবং Memory Management](#2-v8-engine-এবং-memory-management)
3. [Process এবং Child Processes](#3-process-এবং-child-processes)
4. [Cluster Mode এবং Scalability](#4-cluster-mode-এবং-scalability)
5. [Stream API বিস্তারিত](#5-stream-api-বিস্তারিত)

### Advanced Topics

1. [Worker Threads](#6-worker-threads)
2. [Buffer এবং Binary Data](#7-buffer-এবং-binary-data)
3. [Crypto এবং Security](#8-crypto-এবং-security)
4. [Performance Hooks](#9-performance-hooks)
5. [Async Hooks](#10-async-hooks)

### Module System

1. [CommonJS vs ES Modules](#11-commonjs-vs-es-modules)
2. [Module Resolution](#12-module-resolution)
3. [TypeScript Integration](#13-typescript-integration)

### Networking

1. [HTTP/HTTPS Server](#14-httphttps-server)
2. [HTTP/2 Implementation](#15-http2-implementation)
3. [WebSocket এবং Real-time Communication](#16-websocket-এবং-real-time-communication)
4. [TCP/UDP Networking](#17-tcpudp-networking)

### Database & Storage

1. [SQLite Integration](#18-sqlite-integration)
2. [File System Advanced](#19-file-system-advanced)
3. [Database Connection Pooling](#20-database-connection-pooling)

### Testing & Debugging

1. [Test Runner](#21-test-runner)
2. [Debugger এবং Inspector](#22-debugger-এবং-inspector)
3. [Performance Profiling](#23-performance-profiling)

### Production & Deployment

1. [Environment Variables](#24-environment-variables)
2. [Error Handling Strategies](#25-error-handling-strategies)
3. [Monitoring এবং Logging](#26-monitoring-এবং-logging)
4. [Security Best Practices](#27-security-best-practices)

### Advanced Features

1. [Single Executable Applications](#28-single-executable-applications)
2. [WASI (WebAssembly System Interface)](#29-wasi-webassembly-system-interface)
3. [Web Crypto API](#30-web-crypto-api)

---

## উত্তর

## 1. Event Loop এবং Non-blocking I/O

### Event Loop কি এবং এটি কিভাবে কাজ করে?

Event Loop হল Node.js এর হৃদয় যা asynchronous operations পরিচালনা করে।

### Event Loop Phases

```javascript
// Event Loop এর 6টি phase:
// 1. Timers - setTimeout, setInterval
// 2. Pending Callbacks - I/O callbacks
// 3. Idle, Prepare - Internal use
// 4. Poll - I/O events
// 5. Check - setImmediate
// 6. Close Callbacks - socket.on('close')

console.log("1. Start");

setTimeout(() => {
	console.log("2. setTimeout");
}, 0);

setImmediate(() => {
	console.log("3. setImmediate");
});

process.nextTick(() => {
	console.log("4. nextTick");
});

Promise.resolve().then(() => {
	console.log("5. Promise");
});

console.log("6. End");

// Output:
// 1. Start
// 6. End
// 4. nextTick
// 5. Promise
// 2. setTimeout
// 3. setImmediate
```

### Microtasks vs Macrotasks

```javascript
// Microtasks (higher priority)
process.nextTick(() => console.log("nextTick"));
Promise.resolve().then(() => console.log("Promise"));
queueMicrotask(() => console.log("queueMicrotask"));

// Macrotasks (lower priority)
setTimeout(() => console.log("setTimeout"), 0);
setImmediate(() => console.log("setImmediate"));
setInterval(() => console.log("setInterval"), 100);
```

### Event Loop Blocking

```javascript
// খারাপ - Event Loop block করে
function blockingOperation() {
	const start = Date.now();
	while (Date.now() - start < 5000) {
		// 5 seconds blocking
	}
}

// ভালো - Non-blocking
async function nonBlockingOperation() {
	await new Promise((resolve) => setTimeout(resolve, 5000));
}

// CPU-intensive কাজের জন্য Worker Threads ব্যবহার করুন
const { Worker } = require("worker_threads");
```

### Event Loop Monitoring

```javascript
const { performance } = require("perf_hooks");

const obs = new PerformanceObserver((items) => {
	items.getEntries().forEach((entry) => {
		console.log("Event Loop Delay:", entry.duration);
	});
});

obs.observe({ entryTypes: ["measure"] });

setInterval(() => {
	performance.mark("loop-start");
	setImmediate(() => {
		performance.mark("loop-end");
		performance.measure("loop", "loop-start", "loop-end");
	});
}, 1000);
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 2. V8 Engine এবং Memory Management

### Heap Memory এবং Garbage Collection

```javascript
// Memory usage চেক করুন
const used = process.memoryUsage();
console.log({
	rss: `${Math.round(used.rss / 1024 / 1024)} MB`, // Resident Set Size
	heapTotal: `${Math.round(used.heapTotal / 1024 / 1024)} MB`,
	heapUsed: `${Math.round(used.heapUsed / 1024 / 1024)} MB`,
	external: `${Math.round(used.external / 1024 / 1024)} MB`,
	arrayBuffers: `${Math.round(used.arrayBuffers / 1024 / 1024)} MB`,
});
```

### Memory Leaks এড়ানো

```javascript
// খারাপ - Memory leak
const users = [];
function addUser(user) {
	users.push(user); // কখনো clear হয় না
}

// ভালো - WeakMap ব্যবহার
const usersCache = new WeakMap();
function cacheUser(key, user) {
	usersCache.set(key, user); // Auto garbage collected
}

// ভালো - LRU Cache
class LRUCache {
	constructor(maxSize) {
		this.maxSize = maxSize;
		this.cache = new Map();
	}

	set(key, value) {
		if (this.cache.has(key)) {
			this.cache.delete(key);
		}
		this.cache.set(key, value);

		if (this.cache.size > this.maxSize) {
			const firstKey = this.cache.keys().next().value;
			this.cache.delete(firstKey);
		}
	}

	get(key) {
		if (!this.cache.has(key)) return null;

		const value = this.cache.get(key);
		this.cache.delete(key);
		this.cache.set(key, value); // Move to end
		return value;
	}
}
```

### V8 Flags এবং Optimization

```bash
# Heap size বৃদ্ধি করুন
node --max-old-space-size=4096 app.js

# Garbage collection log
node --trace-gc app.js

# Optimization রিপোর্ট
node --trace-opt app.js
node --trace-deopt app.js

# Stack size বৃদ্ধি
node --stack-size=2000 app.js
```

### V8 Heap Snapshot

```javascript
const v8 = require("v8");
const fs = require("fs");

// Heap snapshot তৈরি
const snapshot = v8.writeHeapSnapshot();
console.log("Snapshot saved:", snapshot);

// Heap statistics
const heapStats = v8.getHeapStatistics();
console.log("Heap Stats:", {
	totalHeapSize: heapStats.total_heap_size,
	usedHeapSize: heapStats.used_heap_size,
	heapSizeLimit: heapStats.heap_size_limit,
});

// Serialization
const buffer = v8.serialize({ name: "John", age: 30 });
const obj = v8.deserialize(buffer);
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 3. Process এবং Child Processes

### Process Object

```javascript
// Process information
console.log("Process ID:", process.pid);
console.log("Parent Process ID:", process.ppid);
console.log("Node Version:", process.version);
console.log("Platform:", process.platform);
console.log("Architecture:", process.arch);
console.log("Uptime:", process.uptime());
console.log("Current Directory:", process.cwd());
console.log("Memory Usage:", process.memoryUsage());
console.log("CPU Usage:", process.cpuUsage());

// Command line arguments
console.log("Arguments:", process.argv);
console.log("Exec Path:", process.execPath);
console.log("Exec Arguments:", process.execArgv);

// Environment variables
console.log("NODE_ENV:", process.env.NODE_ENV);
```

### Process Events

```javascript
// Uncaught Exception
process.on("uncaughtException", (error, origin) => {
	console.error("Uncaught Exception:", error);
	console.error("Origin:", origin);
	// Log করুন এবং gracefully exit
	process.exit(1);
});

// Unhandled Promise Rejection
process.on("unhandledRejection", (reason, promise) => {
	console.error("Unhandled Rejection at:", promise);
	console.error("Reason:", reason);
	process.exit(1);
});

// Warning
process.on("warning", (warning) => {
	console.warn("Warning:", warning.name);
	console.warn(warning.message);
	console.warn(warning.stack);
});

// Exit
process.on("exit", (code) => {
	console.log("Process exiting with code:", code);
});

// SIGTERM (Graceful shutdown)
process.on("SIGTERM", () => {
	console.log("SIGTERM received");
	server.close(() => {
		console.log("Server closed");
		process.exit(0);
	});
});

// SIGINT (Ctrl+C)
process.on("SIGINT", () => {
	console.log("SIGINT received");
	process.exit(0);
});
```

### Child Processes

```javascript
const { spawn, exec, execFile, fork } = require("child_process");

// 1. spawn - Stream-based
const ls = spawn("ls", ["-lh", "/usr"]);

ls.stdout.on("data", (data) => {
	console.log(`stdout: ${data}`);
});

ls.stderr.on("data", (data) => {
	console.error(`stderr: ${data}`);
});

ls.on("close", (code) => {
	console.log(`Process exited with code ${code}`);
});

// 2. exec - Buffer-based
exec("ls -lh /usr", (error, stdout, stderr) => {
	if (error) {
		console.error(`Error: ${error}`);
		return;
	}
	console.log(`Output: ${stdout}`);
});

// 3. execFile - নির্দিষ্ট ফাইল execute
execFile("node", ["--version"], (error, stdout, stderr) => {
	if (error) throw error;
	console.log("Node Version:", stdout);
});

// 4. fork - Node.js processes
// main.js
const child = fork("./worker.js");

child.on("message", (msg) => {
	console.log("Message from child:", msg);
});

child.send({ cmd: "start", data: "hello" });

// worker.js
process.on("message", (msg) => {
	console.log("Message from parent:", msg);

	// Heavy computation
	const result = computeHeavyTask(msg.data);

	process.send({ result });
});
```

### Advanced Child Process Patterns

```javascript
const { spawn } = require("child_process");

// Process pool
class ProcessPool {
	constructor(size) {
		this.size = size;
		this.pool = [];
		this.queue = [];

		for (let i = 0; i < size; i++) {
			this.pool.push(this.createWorker());
		}
	}

	createWorker() {
		return {
			process: fork("./worker.js"),
			busy: false,
		};
	}

	async execute(task) {
		return new Promise((resolve, reject) => {
			const worker = this.pool.find((w) => !w.busy);

			if (worker) {
				worker.busy = true;
				worker.process.send(task);

				worker.process.once("message", (result) => {
					worker.busy = false;
					this.processQueue();
					resolve(result);
				});
			} else {
				this.queue.push({ task, resolve, reject });
			}
		});
	}

	processQueue() {
		if (this.queue.length === 0) return;

		const worker = this.pool.find((w) => !w.busy);
		if (worker) {
			const { task, resolve, reject } = this.queue.shift();
			this.execute(task).then(resolve).catch(reject);
		}
	}
}

// ব্যবহার
const pool = new ProcessPool(4);
const result = await pool.execute({ cmd: "process", data: largeData });
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 4. Cluster Mode এবং Scalability

### Cluster Module

```javascript
const cluster = require("cluster");
const http = require("http");
const os = require("os");

const numCPUs = os.cpus().length;

if (cluster.isMaster) {
	console.log(`Master ${process.pid} running`);
	console.log(`Forking ${numCPUs} workers...`);

	// Workers তৈরি
	for (let i = 0; i < numCPUs; i++) {
		cluster.fork();
	}

	// Worker restart করুন যদি crash হয়
	cluster.on("exit", (worker, code, signal) => {
		console.log(`Worker ${worker.process.pid} died`);
		console.log("Starting a new worker...");
		cluster.fork();
	});

	// Workers থেকে message
	cluster.on("message", (worker, message) => {
		console.log(`Message from worker ${worker.id}:`, message);
	});
} else {
	// Workers এ HTTP server
	const server = http.createServer((req, res) => {
		res.writeHead(200);
		res.end(`Response from worker ${process.pid}\n`);
	});

	server.listen(8000);
	console.log(`Worker ${process.pid} started`);
}
```

### Load Balancing

```javascript
const cluster = require("cluster");
const express = require("express");

if (cluster.isMaster) {
	const workers = new Map();

	// Worker management
	for (let i = 0; i < os.cpus().length; i++) {
		const worker = cluster.fork();
		workers.set(worker.id, {
			worker,
			load: 0,
			requests: 0,
		});
	}

	// Load balancing logic
	cluster.on("message", (worker, msg) => {
		if (msg.cmd === "notifyRequest") {
			const workerData = workers.get(worker.id);
			workerData.requests++;
		}
	});

	// Health check
	setInterval(() => {
		workers.forEach((data, id) => {
			console.log(`Worker ${id}: ${data.requests} requests`);
			data.requests = 0; // Reset counter
		});
	}, 5000);
} else {
	const app = express();

	app.use((req, res, next) => {
		process.send({ cmd: "notifyRequest" });
		next();
	});

	app.get("/", (req, res) => {
		res.send(`Worker ${process.pid}`);
	});

	app.listen(3000);
}
```

### Zero-downtime Deployment

```javascript
const cluster = require("cluster");

if (cluster.isMaster) {
	let workers = [];

	function spawnWorkers() {
		for (let i = 0; i < numCPUs; i++) {
			const worker = cluster.fork();
			workers.push(worker);
		}
	}

	function restartWorkers() {
		const workersToRestart = [...workers];
		workers = [];

		workersToRestart.forEach((worker, index) => {
			setTimeout(() => {
				console.log(`Restarting worker ${worker.id}`);
				const newWorker = cluster.fork();
				workers.push(newWorker);

				newWorker.on("listening", () => {
					worker.disconnect();
				});
			}, index * 2000); // Staggered restart
		});
	}

	spawnWorkers();

	// Graceful restart signal
	process.on("SIGUSR2", () => {
		console.log("Graceful restart initiated");
		restartWorkers();
	});
}
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 5. Stream API বিস্তারিত

### Stream এর 4 প্রকার

```javascript
const { Readable, Writable, Duplex, Transform } = require("stream");

// 1. Readable Stream
class MyReadable extends Readable {
	constructor(options) {
		super(options);
		this.counter = 0;
	}

	_read() {
		this.counter++;
		if (this.counter > 10) {
			this.push(null); // End of stream
		} else {
			this.push(`Data chunk ${this.counter}\n`);
		}
	}
}

const readable = new MyReadable();
readable.pipe(process.stdout);

// 2. Writable Stream
class MyWritable extends Writable {
	_write(chunk, encoding, callback) {
		console.log("Writing:", chunk.toString());
		callback(); // Signal completion
	}

	_writev(chunks, callback) {
		// Multiple chunks at once
		chunks.forEach(({ chunk }) => {
			console.log("Batch write:", chunk.toString());
		});
		callback();
	}

	_final(callback) {
		console.log("Stream ending");
		callback();
	}
}

// 3. Duplex Stream (Read + Write)
class MyDuplex extends Duplex {
	_read() {
		this.push("data from read side\n");
		this.push(null);
	}

	_write(chunk, encoding, callback) {
		console.log("Write side:", chunk.toString());
		callback();
	}
}

// 4. Transform Stream
class UpperCaseTransform extends Transform {
	_transform(chunk, encoding, callback) {
		this.push(chunk.toString().toUpperCase());
		callback();
	}
}

const transform = new UpperCaseTransform();
process.stdin.pipe(transform).pipe(process.stdout);
```

### File Streaming (Large Files)

```javascript
const fs = require("fs");
const { pipeline } = require("stream/promises");
const zlib = require("zlib");

// Large file copy with compression
async function compressFile(input, output) {
	await pipeline(fs.createReadStream(input), zlib.createGzip(), fs.createWriteStream(output));
	console.log("File compressed successfully");
}

// Large file processing
async function processLargeFile(filename) {
	const readable = fs.createReadStream(filename, {
		encoding: "utf8",
		highWaterMark: 16 * 1024, // 16KB chunks
	});

	for await (const chunk of readable) {
		// Process each chunk
		await processChunk(chunk);
	}
}

// Stream backpressure handling
const readable = fs.createReadStream("large-file.txt");
const writable = fs.createWriteStream("output.txt");

readable.on("data", (chunk) => {
	const canContinue = writable.write(chunk);

	if (!canContinue) {
		readable.pause(); // Handle backpressure
	}
});

writable.on("drain", () => {
	readable.resume(); // Resume reading
});
```

### Advanced Transform Patterns

```javascript
const { Transform } = require("stream");

// CSV Parser Transform
class CSVParser extends Transform {
	constructor(options) {
		super(options);
		this.headers = null;
		this.buffer = "";
	}

	_transform(chunk, encoding, callback) {
		this.buffer += chunk.toString();
		const lines = this.buffer.split("\n");
		this.buffer = lines.pop(); // Keep incomplete line

		if (!this.headers) {
			this.headers = lines.shift().split(",");
		}

		lines.forEach((line) => {
			if (line.trim()) {
				const values = line.split(",");
				const obj = {};
				this.headers.forEach((header, i) => {
					obj[header] = values[i];
				});
				this.push(JSON.stringify(obj) + "\n");
			}
		});

		callback();
	}

	_flush(callback) {
		if (this.buffer) {
			// Process remaining data
			this._transform(this.buffer, "utf8", callback);
		} else {
			callback();
		}
	}
}

// ব্যবহার
fs.createReadStream("data.csv").pipe(new CSVParser()).pipe(process.stdout);
```

### Stream Error Handling

```javascript
const { pipeline } = require("stream");

pipeline(fs.createReadStream("input.txt"), transform, fs.createWriteStream("output.txt"), (err) => {
	if (err) {
		console.error("Pipeline failed:", err);
	} else {
		console.log("Pipeline succeeded");
	}
});

// Promise-based
const { pipeline } = require("stream/promises");

try {
	await pipeline(fs.createReadStream("input.txt"), transform, fs.createWriteStream("output.txt"));
	console.log("Success");
} catch (err) {
	console.error("Failed:", err);
}
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 6. Worker Threads

Worker Threads CPU-intensive কাজের জন্য multi-threading সক্ষম করে।

### Basic Worker Thread

```javascript
// main.js
const { Worker } = require("worker_threads");

function runWorker(workerData) {
	return new Promise((resolve, reject) => {
		const worker = new Worker("./worker.js", {
			workerData,
		});

		worker.on("message", resolve);
		worker.on("error", reject);
		worker.on("exit", (code) => {
			if (code !== 0) {
				reject(new Error(`Worker stopped with exit code ${code}`));
			}
		});
	});
}

// ব্যবহার
async function main() {
	const result = await runWorker({ num: 42 });
	console.log("Result:", result);
}

// worker.js
const { workerData, parentPort } = require("worker_threads");

function fibonacci(n) {
	if (n <= 1) return n;
	return fibonacci(n - 1) + fibonacci(n - 2);
}

const result = fibonacci(workerData.num);
parentPort.postMessage(result);
```

### Worker Pool

```javascript
const { Worker } = require("worker_threads");
const os = require("os");

class WorkerPool {
	constructor(workerScript, poolSize = os.cpus().length) {
		this.workerScript = workerScript;
		this.poolSize = poolSize;
		this.workers = [];
		this.freeWorkers = [];
		this.tasks = [];

		for (let i = 0; i < poolSize; i++) {
			this.addWorker();
		}
	}

	addWorker() {
		const worker = new Worker(this.workerScript);
		this.workers.push(worker);
		this.freeWorkers.push(worker);

		worker.on("message", (result) => {
			this.freeWorkers.push(worker);
			this.processQueue();
		});

		worker.on("error", (err) => {
			console.error("Worker error:", err);
		});

		worker.on("exit", (code) => {
			this.workers = this.workers.filter((w) => w !== worker);
			this.freeWorkers = this.freeWorkers.filter((w) => w !== worker);

			if (code !== 0) {
				console.error(`Worker stopped with exit code ${code}`);
			}
		});
	}

	async exec(data) {
		return new Promise((resolve, reject) => {
			this.tasks.push({ data, resolve, reject });
			this.processQueue();
		});
	}

	processQueue() {
		if (this.tasks.length === 0 || this.freeWorkers.length === 0) {
			return;
		}

		const worker = this.freeWorkers.pop();
		const { data, resolve, reject } = this.tasks.shift();

		worker.once("message", resolve);
		worker.once("error", reject);
		worker.postMessage(data);
	}

	destroy() {
		this.workers.forEach((worker) => worker.terminate());
	}
}

// ব্যবহার
const pool = new WorkerPool("./worker.js", 4);

async function processTasks() {
	const tasks = Array.from({ length: 100 }, (_, i) => i);
	const results = await Promise.all(tasks.map((task) => pool.exec(task)));
	console.log("All tasks completed");
	pool.destroy();
}
```

### SharedArrayBuffer এবং Atomics

```javascript
// main.js
const { Worker } = require("worker_threads");

const sharedBuffer = new SharedArrayBuffer(4);
const sharedArray = new Int32Array(sharedBuffer);

const worker = new Worker("./atomic-worker.js", {
	workerData: { sharedBuffer },
});

// Atomic operations
Atomics.store(sharedArray, 0, 100);
console.log("Initial value:", Atomics.load(sharedArray, 0));

worker.on("message", () => {
	console.log("Final value:", Atomics.load(sharedArray, 0));
});

// atomic-worker.js
const { workerData, parentPort } = require("worker_threads");
const { sharedBuffer } = workerData;
const sharedArray = new Int32Array(sharedBuffer);

// Atomic increment
for (let i = 0; i < 100; i++) {
	Atomics.add(sharedArray, 0, 1);
}

// Wait/notify pattern
Atomics.wait(sharedArray, 0, 0); // Wait for value change
Atomics.notify(sharedArray, 0); // Notify waiting threads

parentPort.postMessage("done");
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 7. Buffer এবং Binary Data

### Buffer Operations

```javascript
// Buffer তৈরি
const buf1 = Buffer.alloc(10); // Zero-filled
const buf2 = Buffer.allocUnsafe(10); // Uninitialized (faster)
const buf3 = Buffer.from([1, 2, 3, 4, 5]);
const buf4 = Buffer.from("Hello", "utf8");
const buf5 = Buffer.from("48656c6c6f", "hex");

// Buffer operations
const buffer = Buffer.from("Hello World");

console.log("Length:", buffer.length);
console.log("toString:", buffer.toString());
console.log("toJSON:", buffer.toJSON());
console.log("Hex:", buffer.toString("hex"));
console.log("Base64:", buffer.toString("base64"));

// Buffer manipulation
buffer.write("Hi", 0, 2); // Write at offset
const slice = buffer.slice(0, 5); // Slice (shares memory)
const copy = Buffer.from(buffer); // Copy (new memory)

// Buffer comparison
const buf1 = Buffer.from("ABC");
const buf2 = Buffer.from("ABC");
const buf3 = Buffer.from("BCD");

console.log(buf1.equals(buf2)); // true
console.log(Buffer.compare(buf1, buf3)); // -1 (buf1 < buf3)

// Buffer concatenation
const combined = Buffer.concat([buf1, buf2, buf3]);
```

### Binary Protocol Implementation

```javascript
// Custom binary protocol
class BinaryProtocol {
	// Encode message
	static encode(type, payload) {
		const typeBuffer = Buffer.alloc(1);
		typeBuffer.writeUInt8(type, 0);

		const lengthBuffer = Buffer.alloc(4);
		const payloadBuffer = Buffer.from(JSON.stringify(payload));
		lengthBuffer.writeUInt32BE(payloadBuffer.length, 0);

		return Buffer.concat([typeBuffer, lengthBuffer, payloadBuffer]);
	}

	// Decode message
	static decode(buffer) {
		const type = buffer.readUInt8(0);
		const length = buffer.readUInt32BE(1);
		const payload = buffer.slice(5, 5 + length);

		return {
			type,
			payload: JSON.parse(payload.toString()),
		};
	}
}

// ব্যবহার
const message = BinaryProtocol.encode(1, { user: "John", action: "login" });
const decoded = BinaryProtocol.decode(message);
console.log(decoded);
```

### File Buffer Operations

```javascript
const fs = require("fs").promises;

async function readBinaryFile(filename) {
	const buffer = await fs.readFile(filename);

	// Read different data types
	const int8 = buffer.readInt8(0);
	const uint16 = buffer.readUInt16BE(1);
	const int32 = buffer.readInt32LE(3);
	const float = buffer.readFloatBE(7);
	const double = buffer.readDoubleBE(11);

	return { int8, uint16, int32, float, double };
}

async function writeBinaryFile(filename, data) {
	const buffer = Buffer.alloc(19);

	buffer.writeInt8(data.int8, 0);
	buffer.writeUInt16BE(data.uint16, 1);
	buffer.writeInt32LE(data.int32, 3);
	buffer.writeFloatBE(data.float, 7);
	buffer.writeDoubleBE(data.double, 11);

	await fs.writeFile(filename, buffer);
}
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 8. Crypto এবং Security

### Hashing

```javascript
const crypto = require("crypto");

// Password hashing (bcrypt style)
function hashPassword(password) {
	const salt = crypto.randomBytes(16).toString("hex");
	const hash = crypto.pbkdf2Sync(password, salt, 100000, 64, "sha512").toString("hex");
	return `${salt}:${hash}`;
}

function verifyPassword(password, stored) {
	const [salt, originalHash] = stored.split(":");
	const hash = crypto.pbkdf2Sync(password, salt, 100000, 64, "sha512").toString("hex");
	return hash === originalHash;
}

// Async version
async function hashPasswordAsync(password) {
	return new Promise((resolve, reject) => {
		const salt = crypto.randomBytes(16).toString("hex");

		crypto.pbkdf2(password, salt, 100000, 64, "sha512", (err, derivedKey) => {
			if (err) reject(err);
			resolve(`${salt}:${derivedKey.toString("hex")}`);
		});
	});
}

// SHA-256 Hash
function sha256(data) {
	return crypto.createHash("sha256").update(data).digest("hex");
}

// HMAC
function createHMAC(data, secret) {
	return crypto.createHmac("sha256", secret).update(data).digest("hex");
}

function verifyHMAC(data, secret, hmac) {
	const calculated = createHMAC(data, secret);
	return crypto.timingSafeEqual(Buffer.from(calculated), Buffer.from(hmac));
}
```

### Encryption/Decryption

```javascript
// AES-256-GCM Encryption
function encrypt(text, key) {
	const iv = crypto.randomBytes(16);
	const cipher = crypto.createCipheriv("aes-256-gcm", key, iv);

	let encrypted = cipher.update(text, "utf8", "hex");
	encrypted += cipher.final("hex");

	const authTag = cipher.getAuthTag();

	return {
		encrypted,
		iv: iv.toString("hex"),
		authTag: authTag.toString("hex"),
	};
}

function decrypt(encrypted, key, iv, authTag) {
	const decipher = crypto.createDecipheriv("aes-256-gcm", key, Buffer.from(iv, "hex"));

	decipher.setAuthTag(Buffer.from(authTag, "hex"));

	let decrypted = decipher.update(encrypted, "hex", "utf8");
	decrypted += decipher.final("utf8");

	return decrypted;
}

// RSA Asymmetric Encryption
const { publicKey, privateKey } = crypto.generateKeyPairSync("rsa", {
	modulusLength: 2048,
	publicKeyEncoding: {
		type: "spki",
		format: "pem",
	},
	privateKeyEncoding: {
		type: "pkcs8",
		format: "pem",
	},
});

function rsaEncrypt(data, publicKey) {
	return crypto.publicEncrypt(
		{
			key: publicKey,
			padding: crypto.constants.RSA_PKCS1_OAEP_PADDING,
			oaepHash: "sha256",
		},
		Buffer.from(data),
	);
}

function rsaDecrypt(encrypted, privateKey) {
	return crypto.privateDecrypt(
		{
			key: privateKey,
			padding: crypto.constants.RSA_PKCS1_OAEP_PADDING,
			oaepHash: "sha256",
		},
		encrypted,
	);
}
```

### Digital Signatures

```javascript
// Digital signature তৈরি
function sign(data, privateKey) {
	const sign = crypto.createSign("SHA256");
	sign.update(data);
	sign.end();
	return sign.sign(privateKey, "hex");
}

// Signature verify
function verify(data, signature, publicKey) {
	const verify = crypto.createVerify("SHA256");
	verify.update(data);
	verify.end();
	return verify.verify(publicKey, signature, "hex");
}

// ব্যবহার
const message = "Important message";
const signature = sign(message, privateKey);
const isValid = verify(message, signature, publicKey);
console.log("Signature valid:", isValid);
```

### JWT Implementation (from scratch)

```javascript
const crypto = require("crypto");

class JWT {
	static encode(payload, secret, expiresIn = "1h") {
		const header = {
			alg: "HS256",
			typ: "JWT",
		};

		const now = Math.floor(Date.now() / 1000);
		const exp = now + this.parseExpiry(expiresIn);

		const claims = {
			...payload,
			iat: now,
			exp,
		};

		const headerBase64 = Buffer.from(JSON.stringify(header)).toString("base64url");
		const payloadBase64 = Buffer.from(JSON.stringify(claims)).toString("base64url");

		const signature = crypto
			.createHmac("sha256", secret)
			.update(`${headerBase64}.${payloadBase64}`)
			.digest("base64url");

		return `${headerBase64}.${payloadBase64}.${signature}`;
	}

	static decode(token, secret) {
		const [headerBase64, payloadBase64, signature] = token.split(".");

		// Verify signature
		const expectedSignature = crypto
			.createHmac("sha256", secret)
			.update(`${headerBase64}.${payloadBase64}`)
			.digest("base64url");

		if (signature !== expectedSignature) {
			throw new Error("Invalid signature");
		}

		const payload = JSON.parse(Buffer.from(payloadBase64, "base64url").toString());

		// Check expiry
		if (payload.exp && payload.exp < Math.floor(Date.now() / 1000)) {
			throw new Error("Token expired");
		}

		return payload;
	}

	static parseExpiry(str) {
		const units = { s: 1, m: 60, h: 3600, d: 86400 };
		const match = str.match(/^(\d+)([smhd])$/);
		return match ? parseInt(match[1]) * units[match[2]] : 3600;
	}
}

// ব্যবহার
const token = JWT.encode({ userId: 123, role: "admin" }, "secret-key", "2h");
console.log("Token:", token);

const decoded = JWT.decode(token, "secret-key");
console.log("Decoded:", decoded);
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 9. Performance Hooks

### Performance Measurement

```javascript
const { performance, PerformanceObserver } = require("perf_hooks");

// Basic measurement
performance.mark("start");

// ... some operation ...
for (let i = 0; i < 1000000; i++) {
	Math.sqrt(i);
}

performance.mark("end");
performance.measure("operation", "start", "end");

const measure = performance.getEntriesByName("operation")[0];
console.log(`Duration: ${measure.duration}ms`);

// Performance Observer
const obs = new PerformanceObserver((items) => {
	items.getEntries().forEach((entry) => {
		console.log(`${entry.name}: ${entry.duration}ms`);
	});
});

obs.observe({ entryTypes: ["measure", "function"] });

// Function performance
async function slowFunction() {
	await new Promise((resolve) => setTimeout(resolve, 100));
	return "done";
}

performance.timerify(slowFunction)();
```

### HTTP Request Timing

```javascript
const http = require("http");
const { PerformanceObserver } = require("perf_hooks");

const obs = new PerformanceObserver((items) => {
	items.getEntries().forEach((entry) => {
		console.log(`${entry.name}: ${entry.duration}ms`);
	});
});

obs.observe({ entryTypes: ["http"] });

const server = http.createServer((req, res) => {
	// Request automatically measured
	res.end("Hello");
});

server.listen(3000);
```

### Custom Performance Metrics

```javascript
class PerformanceTracker {
	constructor() {
		this.metrics = new Map();
	}

	start(name) {
		performance.mark(`${name}-start`);
	}

	end(name) {
		performance.mark(`${name}-end`);
		performance.measure(name, `${name}-start`, `${name}-end`);

		const measure = performance.getEntriesByName(name)[0];
		this.metrics.set(name, {
			duration: measure.duration,
			timestamp: Date.now(),
		});

		return measure.duration;
	}

	getMetrics() {
		return Object.fromEntries(this.metrics);
	}

	clear() {
		performance.clearMarks();
		performance.clearMeasures();
		this.metrics.clear();
	}
}

// ব্যবহার
const tracker = new PerformanceTracker();

tracker.start("database-query");
await db.query("SELECT * FROM users");
tracker.end("database-query");

tracker.start("api-call");
await fetch("https://api.example.com");
tracker.end("api-call");

console.log("Metrics:", tracker.getMetrics());
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 10. Async Hooks

Async Hooks asynchronous operations track করতে দেয়।

### Basic Async Hooks

```javascript
const async_hooks = require("async_hooks");
const fs = require("fs");

// Async context tracking
const asyncHook = async_hooks.createHook({
	init(asyncId, type, triggerAsyncId, resource) {
		fs.writeSync(1, `Init: ${type}(${asyncId}), trigger: ${triggerAsyncId}\n`);
	},
	before(asyncId) {
		fs.writeSync(1, `Before: ${asyncId}\n`);
	},
	after(asyncId) {
		fs.writeSync(1, `After: ${asyncId}\n`);
	},
	destroy(asyncId) {
		fs.writeSync(1, `Destroy: ${asyncId}\n`);
	},
});

asyncHook.enable();

// Test
setTimeout(() => {
	console.log("Timeout callback");
}, 100);
```

### Request Context Tracking

```javascript
const { AsyncLocalStorage } = require("async_hooks");
const express = require("express");

const requestContext = new AsyncLocalStorage();

const app = express();

// Middleware - Context set করুন
app.use((req, res, next) => {
	const requestId = crypto.randomUUID();
	const userId = req.headers["x-user-id"];

	requestContext.run({ requestId, userId, startTime: Date.now() }, () => {
		next();
	});
});

// যেকোনো জায়গা থেকে context access
function logger(message) {
	const context = requestContext.getStore();
	if (context) {
		console.log(`[${context.requestId}] [User: ${context.userId}] ${message}`);
	} else {
		console.log(message);
	}
}

app.get("/api/users", async (req, res) => {
	logger("Fetching users");

	// Nested async operations এ context automatically propagate হয়
	const users = await db.query("SELECT * FROM users");
	logger(`Found ${users.length} users`);

	res.json(users);
});

// Response time logging
app.use((req, res, next) => {
	res.on("finish", () => {
		const context = requestContext.getStore();
		if (context) {
			const duration = Date.now() - context.startTime;
			logger(`Request completed in ${duration}ms`);
		}
	});
	next();
});
```

### Async Context Propagation

```javascript
const { AsyncResource, executionAsyncId } = require("async_hooks");

class AsyncTaskManager {
	constructor() {
		this.tasks = new Map();
	}

	scheduleTask(fn, ...args) {
		const asyncId = executionAsyncId();
		const resource = new AsyncResource("TaskManager");

		const taskId = crypto.randomUUID();

		this.tasks.set(taskId, {
			fn,
			args,
			asyncId,
			createdAt: Date.now(),
		});

		setTimeout(() => {
			resource.runInAsyncScope(() => {
				try {
					fn(...args);
				} finally {
					this.tasks.delete(taskId);
				}
			});
		}, 0);

		return taskId;
	}

	cancelTask(taskId) {
		this.tasks.delete(taskId);
	}
}

// ব্যবহার
const manager = new AsyncTaskManager();
const taskId = manager.scheduleTask(() => {
	console.log("Task executed");
});
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 11. CommonJS vs ES Modules

### CommonJS (CJS)

```javascript
// module.exports
// math.js
function add(a, b) {
	return a + b;
}

function subtract(a, b) {
	return a - b;
}

module.exports = { add, subtract };

// অথবা
exports.add = add;
exports.subtract = subtract;

// require
const math = require("./math");
const { add } = require("./math");
```

### ES Modules (ESM)

```javascript
// export
// math.mjs
export function add(a, b) {
	return a + b;
}

export function subtract(a, b) {
	return a - b;
}

export default class Calculator {
	add(a, b) {
		return a + b;
	}
}

// import
import Calculator, { add, subtract } from "./math.mjs";
import * as math from "./math.mjs";
```

### package.json Configuration

```json
{
	"type": "module",
	"exports": {
		".": {
			"import": "./index.mjs",
			"require": "./index.cjs"
		},
		"./utils": {
			"import": "./utils.mjs",
			"require": "./utils.cjs"
		}
	}
}
```

### Dynamic Import

```javascript
// ESM এ dynamic import
async function loadModule(moduleName) {
	if (moduleName === "heavy") {
		const module = await import("./heavy-module.mjs");
		return module.default;
	}
}

// CommonJS এ
function loadModuleCJS(moduleName) {
	if (moduleName === "heavy") {
		return require("./heavy-module");
	}
}

// Conditional loading
const isDevelopment = process.env.NODE_ENV === "development";

const logger = isDevelopment ? await import("./dev-logger.mjs") : await import("./prod-logger.mjs");
```

### Interoperability

```javascript
// ESM থেকে CJS import
import { createRequire } from "module";
const require = createRequire(import.meta.url);
const cjsModule = require("./cjs-module");

// CJS থেকে ESM load (async)
(async () => {
	const esmModule = await import("./esm-module.mjs");
})();
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 12. Module Resolution

### Module Search Algorithm

```javascript
// require.resolve - module path খুঁজুন
const path = require.resolve("express");
console.log("Express path:", path);

// require.resolve.paths - search paths
const paths = require.resolve.paths("express");
console.log("Search paths:", paths);

// Cache access
console.log("Cached modules:", Object.keys(require.cache));

// Cache clear (সাবধানে!)
delete require.cache[require.resolve("./my-module")];

// Fresh module load
function requireFresh(module) {
	delete require.cache[require.resolve(module)];
	return require(module);
}
```

### Custom Module Loader

```javascript
const Module = require("module");
const path = require("path");

// Original require
const originalRequire = Module.prototype.require;

// Custom require wrapper
Module.prototype.require = function (id) {
	console.log(`Loading module: ${id}`);

	// Custom logic
	if (id.startsWith("@custom/")) {
		const customPath = path.join(__dirname, "custom-modules", id.replace("@custom/", ""));
		return originalRequire.call(this, customPath);
	}

	return originalRequire.call(this, id);
};

// Module alias
const moduleAliases = {
	"@utils": path.join(__dirname, "src/utils"),
	"@config": path.join(__dirname, "src/config"),
	"@models": path.join(__dirname, "src/models"),
};

Module.prototype.require = function (id) {
	for (const [alias, realPath] of Object.entries(moduleAliases)) {
		if (id.startsWith(alias)) {
			const newId = id.replace(alias, realPath);
			return originalRequire.call(this, newId);
		}
	}
	return originalRequire.call(this, id);
};
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 13. TypeScript Integration

### tsconfig.json (Node.js)

```json
{
	"compilerOptions": {
		"target": "ES2022",
		"module": "commonjs",
		"lib": ["ES2022"],
		"outDir": "./dist",
		"rootDir": "./src",
		"strict": true,
		"esModuleInterop": true,
		"skipLibCheck": true,
		"forceConsistentCasingInFileNames": true,
		"moduleResolution": "node",
		"resolveJsonModule": true,
		"declaration": true,
		"declarationMap": true,
		"sourceMap": true,
		"types": ["node"]
	},
	"include": ["src/**/*"],
	"exclude": ["node_modules", "dist"]
}
```

### Type-safe Node.js Code

```typescript
import { EventEmitter } from "events";
import { IncomingMessage, ServerResponse } from "http";
import { Readable } from "stream";

// Typed EventEmitter
interface UserEvents {
	"user:created": (user: User) => void;
	"user:updated": (user: User) => void;
	"user:deleted": (userId: string) => void;
}

class TypedEventEmitter<T extends Record<string, any>> extends EventEmitter {
	emit<K extends keyof T>(event: K, ...args: Parameters<T[K]>): boolean {
		return super.emit(event as string, ...args);
	}

	on<K extends keyof T>(event: K, listener: T[K]): this {
		return super.on(event as string, listener);
	}

	once<K extends keyof T>(event: K, listener: T[K]): this {
		return super.once(event as string, listener);
	}
}

// ব্যবহার
const userEmitter = new TypedEventEmitter<UserEvents>();

userEmitter.on("user:created", (user) => {
	console.log("User created:", user.name); // Type-safe!
});

userEmitter.emit("user:created", { id: "1", name: "John" });
```

### Typed HTTP Server

```typescript
import * as http from "http";

interface RequestHandler {
	(req: IncomingMessage, res: ServerResponse): void | Promise<void>;
}

interface Route {
	method: string;
	path: string;
	handler: RequestHandler;
}

class TypedServer {
	private routes: Route[] = [];
	private server: http.Server;

	constructor() {
		this.server = http.createServer(this.handleRequest.bind(this));
	}

	route(method: string, path: string, handler: RequestHandler): this {
		this.routes.push({ method, path, handler });
		return this;
	}

	get(path: string, handler: RequestHandler): this {
		return this.route("GET", path, handler);
	}

	post(path: string, handler: RequestHandler): this {
		return this.route("POST", path, handler);
	}

	private async handleRequest(req: IncomingMessage, res: ServerResponse) {
		const route = this.routes.find((r) => r.method === req.method && r.path === req.url);

		if (route) {
			try {
				await route.handler(req, res);
			} catch (error) {
				res.writeHead(500);
				res.end("Internal Server Error");
			}
		} else {
			res.writeHead(404);
			res.end("Not Found");
		}
	}

	listen(port: number): void {
		this.server.listen(port, () => {
			console.log(`Server listening on port ${port}`);
		});
	}
}

// ব্যবহার
const app = new TypedServer();

app.get("/users", async (req, res) => {
	res.writeHead(200, { "Content-Type": "application/json" });
	res.end(JSON.stringify({ users: [] }));
});

app.listen(3000);
```

### Advanced TypeScript Patterns

```typescript
// Utility types for Node.js
type AsyncFunction<T = void> = (...args: any[]) => Promise<T>;
type Callback<T = void> = (err: Error | null, result?: T) => void;

// Promisify utility
function promisify<T>(fn: Function): AsyncFunction<T> {
	return function (...args: any[]): Promise<T> {
		return new Promise((resolve, reject) => {
			fn(...args, (err: Error | null, result: T) => {
				if (err) reject(err);
				else resolve(result);
			});
		});
	};
}

// Typed process.env
interface ProcessEnv {
	NODE_ENV: "development" | "production" | "test";
	PORT: string;
	DATABASE_URL: string;
	API_KEY: string;
}

declare global {
	namespace NodeJS {
		interface ProcessEnv extends ProcessEnv {}
	}
}

// ব্যবহার
const port = parseInt(process.env.PORT, 10);
const isDev = process.env.NODE_ENV === "development";
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 14. HTTP/HTTPS Server

### Advanced HTTP Server

```javascript
const http = require("http");
const url = require("url");

class HTTPServer {
	constructor() {
		this.routes = new Map();
		this.middlewares = [];
	}

	use(middleware) {
		this.middlewares.push(middleware);
	}

	route(method, path, handler) {
		const key = `${method}:${path}`;
		this.routes.set(key, handler);
	}

	async executeMiddlewares(req, res) {
		for (const middleware of this.middlewares) {
			await new Promise((resolve, reject) => {
				middleware(req, res, (err) => {
					if (err) reject(err);
					else resolve();
				});
			});
		}
	}

	async handleRequest(req, res) {
		try {
			// Parse URL
			const parsedUrl = url.parse(req.url, true);
			req.pathname = parsedUrl.pathname;
			req.query = parsedUrl.query;

			// Execute middlewares
			await this.executeMiddlewares(req, res);

			// Find route
			const key = `${req.method}:${req.pathname}`;
			const handler = this.routes.get(key);

			if (handler) {
				await handler(req, res);
			} else {
				res.writeHead(404);
				res.end("Not Found");
			}
		} catch (error) {
			console.error("Server error:", error);
			res.writeHead(500);
			res.end("Internal Server Error");
		}
	}

	listen(port, callback) {
		const server = http.createServer(this.handleRequest.bind(this));
		server.listen(port, callback);
		return server;
	}
}

// ব্যবহার
const app = new HTTPServer();

// Middlewares
app.use((req, res, next) => {
	console.log(`${req.method} ${req.url}`);
	req.timestamp = Date.now();
	next();
});

app.use((req, res, next) => {
	res.setHeader("X-Powered-By", "Node.js");
	next();
});

// Routes
app.route("GET", "/api/health", (req, res) => {
	res.writeHead(200, { "Content-Type": "application/json" });
	res.end(JSON.stringify({ status: "ok", uptime: process.uptime() }));
});

app.route("POST", "/api/data", async (req, res) => {
	// Parse body
	const body = await new Promise((resolve) => {
		let data = "";
		req.on("data", (chunk) => (data += chunk));
		req.on("end", () => resolve(JSON.parse(data)));
	});

	res.writeHead(200, { "Content-Type": "application/json" });
	res.end(JSON.stringify({ received: body }));
});

app.listen(3000, () => console.log("Server running on port 3000"));
```

### HTTPS Server with SSL

```javascript
const https = require("https");
const fs = require("fs");

// SSL certificate
const options = {
	key: fs.readFileSync("private-key.pem"),
	cert: fs.readFileSync("certificate.pem"),
	// Optional: CA certificate for client verification
	ca: fs.readFileSync("ca-certificate.pem"),
	// Request client certificate
	requestCert: true,
	rejectUnauthorized: false,
};

const server = https.createServer(options, (req, res) => {
	// Client certificate info
	if (req.client.authorized) {
		console.log("Client certificate valid");
		const cert = req.socket.getPeerCertificate();
		console.log("Subject:", cert.subject);
		console.log("Issuer:", cert.issuer);
	}

	res.writeHead(200);
	res.end("Secure connection established\n");
});

server.listen(8443, () => {
	console.log("HTTPS server running on port 8443");
});

// TLS options
const tlsOptions = {
	minVersion: "TLSv1.2",
	maxVersion: "TLSv1.3",
	ciphers: "ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384",
	honorCipherOrder: true,
};
```

### Request/Response Utilities

```javascript
// Request body parser
async function parseBody(req, maxSize = 1048576) {
	return new Promise((resolve, reject) => {
		let data = "";
		let size = 0;

		req.on("data", (chunk) => {
			size += chunk.length;
			if (size > maxSize) {
				reject(new Error("Request entity too large"));
				return;
			}
			data += chunk;
		});

		req.on("end", () => {
			try {
				const contentType = req.headers["content-type"];
				if (contentType && contentType.includes("application/json")) {
					resolve(JSON.parse(data));
				} else {
					resolve(data);
				}
			} catch (error) {
				reject(error);
			}
		});

		req.on("error", reject);
	});
}

// Response helper
class Response {
	constructor(res) {
		this.res = res;
	}

	json(data, status = 200) {
		this.res.writeHead(status, { "Content-Type": "application/json" });
		this.res.end(JSON.stringify(data));
	}

	html(content, status = 200) {
		this.res.writeHead(status, { "Content-Type": "text/html" });
		this.res.end(content);
	}

	redirect(location, status = 302) {
		this.res.writeHead(status, { Location: location });
		this.res.end();
	}

	error(message, status = 500) {
		this.res.writeHead(status, { "Content-Type": "application/json" });
		this.res.end(JSON.stringify({ error: message }));
	}

	stream(readStream) {
		readStream.pipe(this.res);
	}
}

// ব্যবহার
const server = http.createServer(async (req, res) => {
	const response = new Response(res);

	try {
		if (req.method === "POST") {
			const body = await parseBody(req);
			response.json({ received: body });
		} else {
			response.json({ message: "Hello World" });
		}
	} catch (error) {
		response.error(error.message, 400);
	}
});
```

### HTTP/2 Server Push

```javascript
const http2 = require("http2");
const fs = require("fs");

const server = http2.createSecureServer({
	key: fs.readFileSync("private-key.pem"),
	cert: fs.readFileSync("certificate.pem"),
});

server.on("stream", (stream, headers) => {
	const path = headers[":path"];

	if (path === "/") {
		// Push CSS এবং JS files
		stream.pushStream({ ":path": "/style.css" }, (err, pushStream) => {
			if (err) throw err;
			pushStream.respondWithFile("style.css");
		});

		stream.pushStream({ ":path": "/app.js" }, (err, pushStream) => {
			if (err) throw err;
			pushStream.respondWithFile("app.js");
		});

		// Send HTML
		stream.respondWithFile("index.html", {
			"content-type": "text/html",
		});
	}
});

server.listen(8443);
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 15. HTTP/2 Implementation

### HTTP/2 Server

```javascript
const http2 = require("http2");
const fs = require("fs");

// HTTP/2 server
const server = http2.createSecureServer({
	key: fs.readFileSync("localhost-privkey.pem"),
	cert: fs.readFileSync("localhost-cert.pem"),
});

server.on("error", (err) => console.error(err));

server.on("stream", (stream, headers) => {
	const method = headers[":method"];
	const path = headers[":path"];

	console.log(`${method} ${path}`);

	// Response headers
	stream.respond({
		"content-type": "application/json",
		":status": 200,
	});

	// Send data
	stream.end(
		JSON.stringify({
			message: "HTTP/2 Response",
			headers,
		}),
	);
});

server.listen(8443);
```

### HTTP/2 Client

```javascript
const http2 = require("http2");

const client = http2.connect("https://localhost:8443", {
	rejectUnauthorized: false, // Self-signed certificate এর জন্য
});

client.on("error", (err) => console.error(err));

// Request পাঠান
const req = client.request({
	":path": "/api/data",
	":method": "GET",
});

req.on("response", (headers) => {
	console.log("Status:", headers[":status"]);
	console.log("Headers:", headers);
});

req.setEncoding("utf8");
let data = "";
req.on("data", (chunk) => {
	data += chunk;
});
req.on("end", () => {
	console.log("Response:", data);
	client.close();
});

req.end();
```

### Server Push Example

```javascript
const http2 = require("http2");
const fs = require("fs");
const path = require("path");

const server = http2.createSecureServer({
	key: fs.readFileSync("key.pem"),
	cert: fs.readFileSync("cert.pem"),
});

server.on("stream", (stream, headers) => {
	const reqPath = headers[":path"];

	if (reqPath === "/") {
		// Push resources আগে পাঠান
		const pushResources = [
			{ path: "/style.css", file: "public/style.css", type: "text/css" },
			{ path: "/app.js", file: "public/app.js", type: "application/javascript" },
			{ path: "/logo.png", file: "public/logo.png", type: "image/png" },
		];

		pushResources.forEach((resource) => {
			stream.pushStream({ ":path": resource.path }, (err, pushStream, headers) => {
				if (err) {
					console.error("Push error:", err);
					return;
				}

				pushStream.respondWithFile(resource.file, {
					"content-type": resource.type,
				});
			});
		});

		// Main HTML response
		stream.respondWithFile("public/index.html", {
			"content-type": "text/html",
		});
	} else {
		// Handle other requests
		const filePath = path.join("public", reqPath);
		if (fs.existsSync(filePath)) {
			stream.respondWithFile(filePath);
		} else {
			stream.respond({ ":status": 404 });
			stream.end("Not Found");
		}
	}
});

server.listen(8443, () => {
	console.log("HTTP/2 server with push running on https://localhost:8443");
});
```

### Multiplexing Example

```javascript
const http2 = require("http2");

const client = http2.connect("https://example.com");

// Multiple parallel requests
const requests = ["/api/users", "/api/posts", "/api/comments"];

requests.forEach((path) => {
	const req = client.request({ ":path": path });

	req.on("response", (headers) => {
		console.log(`Response from ${path}:`, headers[":status"]);
	});

	let data = "";
	req.on("data", (chunk) => {
		data += chunk;
	});

	req.on("end", () => {
		console.log(`Data from ${path}:`, data.length, "bytes");
	});

	req.end();
});

// Close after all requests
setTimeout(() => {
	client.close();
}, 5000);
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 16. WebSocket এবং Real-time Communication

### WebSocket Server (ws library)

```javascript
const WebSocket = require("ws");
const http = require("http");

// HTTP server
const server = http.createServer();

// WebSocket server
const wss = new WebSocket.Server({ server });

// Connection tracking
const clients = new Map();

wss.on("connection", (ws, req) => {
	const clientId = generateId();
	clients.set(clientId, ws);

	console.log(`Client ${clientId} connected`);

	// Message handler
	ws.on("message", (data) => {
		const message = JSON.parse(data);
		console.log(`Received from ${clientId}:`, message);

		// Broadcast to all clients
		broadcast(
			{
				type: "message",
				from: clientId,
				data: message,
			},
			clientId,
		);
	});

	// Disconnect handler
	ws.on("close", () => {
		console.log(`Client ${clientId} disconnected`);
		clients.delete(clientId);
	});

	// Error handler
	ws.on("error", (error) => {
		console.error(`Client ${clientId} error:`, error);
	});

	// Send welcome message
	ws.send(
		JSON.stringify({
			type: "welcome",
			clientId,
			message: "Connected to WebSocket server",
		}),
	);
});

// Broadcast function
function broadcast(message, excludeId) {
	const data = JSON.stringify(message);
	clients.forEach((client, id) => {
		if (id !== excludeId && client.readyState === WebSocket.OPEN) {
			client.send(data);
		}
	});
}

// Heartbeat (keep-alive)
setInterval(() => {
	clients.forEach((client, id) => {
		if (client.readyState === WebSocket.OPEN) {
			client.ping();
		} else {
			clients.delete(id);
		}
	});
}, 30000);

server.listen(8080, () => {
	console.log("WebSocket server running on port 8080");
});

function generateId() {
	return Math.random().toString(36).substr(2, 9);
}
```

### Advanced WebSocket Patterns

```javascript
class WebSocketManager {
	constructor() {
		this.wss = null;
		this.rooms = new Map(); // Room-based messaging
		this.clients = new Map();
	}

	initialize(server) {
		this.wss = new WebSocket.Server({ server });

		this.wss.on("connection", (ws, req) => {
			const clientId = this.generateClientId();
			const client = {
				id: clientId,
				ws,
				rooms: new Set(),
				metadata: {},
			};

			this.clients.set(clientId, client);
			this.setupClient(client);
		});
	}

	setupClient(client) {
		const { ws, id } = client;

		ws.on("message", (data) => {
			try {
				const message = JSON.parse(data);
				this.handleMessage(client, message);
			} catch (error) {
				this.sendError(client, "Invalid message format");
			}
		});

		ws.on("close", () => {
			this.handleDisconnect(client);
		});

		this.sendToClient(client, {
			type: "connected",
			clientId: id,
		});
	}

	handleMessage(client, message) {
		switch (message.type) {
			case "join_room":
				this.joinRoom(client, message.room);
				break;
			case "leave_room":
				this.leaveRoom(client, message.room);
				break;
			case "room_message":
				this.sendToRoom(message.room, message.data, client.id);
				break;
			case "private_message":
				this.sendToClient(this.clients.get(message.to), message.data);
				break;
			default:
				this.sendError(client, "Unknown message type");
		}
	}

	joinRoom(client, roomName) {
		if (!this.rooms.has(roomName)) {
			this.rooms.set(roomName, new Set());
		}

		this.rooms.get(roomName).add(client.id);
		client.rooms.add(roomName);

		this.sendToRoom(roomName, {
			type: "user_joined",
			clientId: client.id,
		});
	}

	leaveRoom(client, roomName) {
		if (this.rooms.has(roomName)) {
			this.rooms.get(roomName).delete(client.id);
			client.rooms.delete(roomName);

			this.sendToRoom(roomName, {
				type: "user_left",
				clientId: client.id,
			});
		}
	}

	sendToRoom(roomName, data, excludeId) {
		if (!this.rooms.has(roomName)) return;

		const clients = Array.from(this.rooms.get(roomName))
			.filter((id) => id !== excludeId)
			.map((id) => this.clients.get(id));

		clients.forEach((client) => {
			this.sendToClient(client, data);
		});
	}

	sendToClient(client, data) {
		if (client && client.ws.readyState === WebSocket.OPEN) {
			client.ws.send(JSON.stringify(data));
		}
	}

	sendError(client, error) {
		this.sendToClient(client, {
			type: "error",
			message: error,
		});
	}

	handleDisconnect(client) {
		// Leave all rooms
		client.rooms.forEach((roomName) => {
			this.leaveRoom(client, roomName);
		});

		this.clients.delete(client.id);
	}

	generateClientId() {
		return `client_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
	}
}

// ব্যবহার
const manager = new WebSocketManager();
const server = http.createServer();
manager.initialize(server);
server.listen(8080);
```

### WebSocket Client

```javascript
// Browser/Node.js WebSocket client
class WebSocketClient {
	constructor(url) {
		this.url = url;
		this.ws = null;
		this.reconnectInterval = 5000;
		this.handlers = new Map();
	}

	connect() {
		return new Promise((resolve, reject) => {
			this.ws = new WebSocket(this.url);

			this.ws.on("open", () => {
				console.log("Connected to WebSocket server");
				resolve();
			});

			this.ws.on("message", (data) => {
				try {
					const message = JSON.parse(data);
					this.handleMessage(message);
				} catch (error) {
					console.error("Message parse error:", error);
				}
			});

			this.ws.on("close", () => {
				console.log("Disconnected from server");
				this.reconnect();
			});

			this.ws.on("error", (error) => {
				console.error("WebSocket error:", error);
				reject(error);
			});
		});
	}

	on(type, handler) {
		if (!this.handlers.has(type)) {
			this.handlers.set(type, []);
		}
		this.handlers.get(type).push(handler);
	}

	handleMessage(message) {
		const handlers = this.handlers.get(message.type);
		if (handlers) {
			handlers.forEach((handler) => handler(message));
		}
	}

	send(type, data) {
		if (this.ws && this.ws.readyState === WebSocket.OPEN) {
			this.ws.send(JSON.stringify({ type, data }));
		}
	}

	joinRoom(roomName) {
		this.send("join_room", { room: roomName });
	}

	sendToRoom(roomName, message) {
		this.send("room_message", { room: roomName, message });
	}

	reconnect() {
		setTimeout(() => {
			console.log("Attempting to reconnect...");
			this.connect().catch(() => {
				console.log("Reconnection failed");
			});
		}, this.reconnectInterval);
	}

	close() {
		if (this.ws) {
			this.ws.close();
		}
	}
}

// ব্যবহার
const client = new WebSocketClient("ws://localhost:8080");

client.on("connected", (data) => {
	console.log("Client ID:", data.clientId);
	client.joinRoom("general");
});

client.on("message", (data) => {
	console.log("Message received:", data);
});

client.connect();
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 17. TCP/UDP Networking

### TCP Server

```javascript
const net = require("net");

// TCP Server
const server = net.createServer((socket) => {
	console.log("Client connected:", socket.remoteAddress, socket.remotePort);

	// Socket info
	socket.setEncoding("utf8");
	socket.setTimeout(300000); // 5 minutes

	// Data handler
	socket.on("data", (data) => {
		console.log("Received:", data);

		// Echo back
		socket.write(`Echo: ${data}`);
	});

	// End handler
	socket.on("end", () => {
		console.log("Client disconnected");
	});

	// Error handler
	socket.on("error", (err) => {
		console.error("Socket error:", err);
	});

	// Timeout handler
	socket.on("timeout", () => {
		console.log("Socket timeout");
		socket.end();
	});

	// Welcome message
	socket.write("Welcome to TCP server\n");
});

server.on("error", (err) => {
	console.error("Server error:", err);
});

server.listen(9000, () => {
	console.log("TCP server listening on port 9000");
});
```

### TCP Client

```javascript
const net = require("net");

const client = net.createConnection({ port: 9000, host: "localhost" }, () => {
	console.log("Connected to server");
	client.write("Hello from client\n");
});

client.on("data", (data) => {
	console.log("Server response:", data.toString());
	client.end();
});

client.on("end", () => {
	console.log("Disconnected from server");
});

client.on("error", (err) => {
	console.error("Connection error:", err);
});
```

### Advanced TCP Protocol

```javascript
// Custom protocol implementation
class TCPProtocol {
	constructor() {
		this.buffer = Buffer.alloc(0);
	}

	// Message format: [4 bytes length][payload]
	encode(data) {
		const payload = Buffer.from(JSON.stringify(data));
		const length = Buffer.alloc(4);
		length.writeUInt32BE(payload.length, 0);
		return Buffer.concat([length, payload]);
	}

	decode(chunk) {
		this.buffer = Buffer.concat([this.buffer, chunk]);
		const messages = [];

		while (this.buffer.length >= 4) {
			const length = this.buffer.readUInt32BE(0);

			if (this.buffer.length < 4 + length) {
				break; // Incomplete message
			}

			const payload = this.buffer.slice(4, 4 + length);
			messages.push(JSON.parse(payload.toString()));

			this.buffer = this.buffer.slice(4 + length);
		}

		return messages;
	}
}

// Server with protocol
const protocol = new TCPProtocol();

const server = net.createServer((socket) => {
	const clientProtocol = new TCPProtocol();

	socket.on("data", (chunk) => {
		const messages = clientProtocol.decode(chunk);

		messages.forEach((message) => {
			console.log("Received message:", message);

			// Response
			const response = protocol.encode({
				status: "success",
				echo: message,
			});
			socket.write(response);
		});
	});
});

server.listen(9000);
```

### UDP Server/Client

```javascript
const dgram = require("dgram");

// UDP Server
const server = dgram.createSocket("udp4");

server.on("error", (err) => {
	console.error(`Server error:\n${err.stack}`);
	server.close();
});

server.on("message", (msg, rinfo) => {
	console.log(`Server got: ${msg} from ${rinfo.address}:${rinfo.port}`);

	// Send response
	const response = Buffer.from(`Echo: ${msg}`);
	server.send(response, rinfo.port, rinfo.address, (err) => {
		if (err) console.error("Send error:", err);
	});
});

server.on("listening", () => {
	const address = server.address();
	console.log(`UDP server listening ${address.address}:${address.port}`);
});

server.bind(41234);

// UDP Client
const client = dgram.createSocket("udp4");

const message = Buffer.from("Hello UDP server");

client.send(message, 41234, "localhost", (err) => {
	if (err) {
		console.error("Send error:", err);
		client.close();
	}
});

client.on("message", (msg, rinfo) => {
	console.log(`Client got: ${msg} from ${rinfo.address}:${rinfo.port}`);
	client.close();
});
```

### UDP Multicast

```javascript
const dgram = require("dgram");

// Multicast sender
const sender = dgram.createSocket("udp4");
const MULTICAST_ADDR = "239.255.0.1";
const PORT = 41234;

setInterval(() => {
	const message = Buffer.from(`Multicast message at ${Date.now()}`);
	sender.send(message, PORT, MULTICAST_ADDR, (err) => {
		if (err) console.error(err);
	});
}, 1000);

// Multicast receiver
const receiver = dgram.createSocket({ type: "udp4", reuseAddr: true });

receiver.on("message", (msg, rinfo) => {
	console.log(`Received: ${msg} from ${rinfo.address}:${rinfo.port}`);
});

receiver.on("listening", () => {
	receiver.addMembership(MULTICAST_ADDR);
	console.log("Listening for multicast messages");
});

receiver.bind(PORT);
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 18. SQLite Integration

### Basic SQLite Operations

```javascript
const sqlite3 = require("sqlite3").verbose();
const { promisify } = require("util");

// Database connection
const db = new sqlite3.Database("./database.db", (err) => {
	if (err) console.error("Connection error:", err);
	else console.log("Connected to SQLite database");
});

// Promisify methods
db.runAsync = promisify(db.run.bind(db));
db.getAsync = promisify(db.get.bind(db));
db.allAsync = promisify(db.all.bind(db));

// Create table
async function createTable() {
	await db.runAsync(`
    CREATE TABLE IF NOT EXISTS users (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL,
      email TEXT UNIQUE NOT NULL,
      age INTEGER,
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )
  `);
}

// Insert data
async function insertUser(name, email, age) {
	const result = await db.runAsync("INSERT INTO users (name, email, age) VALUES (?, ?, ?)", [
		name,
		email,
		age,
	]);
	return result.lastID;
}

// Query data
async function getUser(id) {
	return await db.getAsync("SELECT * FROM users WHERE id = ?", [id]);
}

async function getAllUsers() {
	return await db.allAsync("SELECT * FROM users");
}

// Update data
async function updateUser(id, data) {
	const fields = Object.keys(data)
		.map((key) => `${key} = ?`)
		.join(", ");
	const values = [...Object.values(data), id];

	await db.runAsync(`UPDATE users SET ${fields} WHERE id = ?`, values);
}

// Delete data
async function deleteUser(id) {
	await db.runAsync("DELETE FROM users WHERE id = ?", [id]);
}

// Transaction
async function transferBalance(fromId, toId, amount) {
	return new Promise((resolve, reject) => {
		db.serialize(() => {
			db.run("BEGIN TRANSACTION");

			db.run(
				"UPDATE accounts SET balance = balance - ? WHERE id = ?",
				[amount, fromId],
				function (err) {
					if (err) {
						db.run("ROLLBACK");
						return reject(err);
					}

					db.run(
						"UPDATE accounts SET balance = balance + ? WHERE id = ?",
						[amount, toId],
						function (err) {
							if (err) {
								db.run("ROLLBACK");
								return reject(err);
							}

							db.run("COMMIT", (err) => {
								if (err) reject(err);
								else resolve();
							});
						},
					);
				},
			);
		});
	});
}

// ব্যবহার
(async () => {
	await createTable();

	const userId = await insertUser("John Doe", "john@example.com", 30);
	console.log("User created with ID:", userId);

	const user = await getUser(userId);
	console.log("User:", user);

	await updateUser(userId, { age: 31 });

	const allUsers = await getAllUsers();
	console.log("All users:", allUsers);

	await deleteUser(userId);
})();
```

### Advanced SQLite Patterns

```javascript
class Database {
	constructor(filename) {
		this.db = new sqlite3.Database(filename);
		this.db.configure("busyTimeout", 10000);
	}

	// Generic query method
	async query(sql, params = []) {
		return new Promise((resolve, reject) => {
			this.db.all(sql, params, (err, rows) => {
				if (err) reject(err);
				else resolve(rows);
			});
		});
	}

	// Single row query
	async get(sql, params = []) {
		return new Promise((resolve, reject) => {
			this.db.get(sql, params, (err, row) => {
				if (err) reject(err);
				else resolve(row);
			});
		});
	}

	// Execute (INSERT, UPDATE, DELETE)
	async run(sql, params = []) {
		return new Promise((resolve, reject) => {
			this.db.run(sql, params, function (err) {
				if (err) reject(err);
				else
					resolve({
						lastID: this.lastID,
						changes: this.changes,
					});
			});
		});
	}

	// Transaction wrapper
	async transaction(callback) {
		await this.run("BEGIN TRANSACTION");

		try {
			await callback(this);
			await this.run("COMMIT");
		} catch (error) {
			await this.run("ROLLBACK");
			throw error;
		}
	}

	// Prepared statements
	prepare(sql) {
		const stmt = this.db.prepare(sql);

		return {
			run: promisify(stmt.run.bind(stmt)),
			get: promisify(stmt.get.bind(stmt)),
			all: promisify(stmt.all.bind(stmt)),
			finalize: promisify(stmt.finalize.bind(stmt)),
		};
	}

	// Batch insert
	async batchInsert(table, columns, rows) {
		const placeholders = columns.map(() => "?").join(", ");
		const sql = `INSERT INTO ${table} (${columns.join(", ")}) VALUES (${placeholders})`;

		const stmt = this.prepare(sql);

		await this.transaction(async () => {
			for (const row of rows) {
				await stmt.run(row);
			}
		});

		await stmt.finalize();
	}

	// Close connection
	close() {
		return new Promise((resolve, reject) => {
			this.db.close((err) => {
				if (err) reject(err);
				else resolve();
			});
		});
	}
}

// ব্যবহার
const db = new Database("./myapp.db");

// Migration example
async function migrate() {
	await db.run(`
    CREATE TABLE IF NOT EXISTS users (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      username TEXT UNIQUE NOT NULL,
      password TEXT NOT NULL,
      email TEXT UNIQUE NOT NULL,
      created_at INTEGER DEFAULT (strftime('%s', 'now'))
    )
  `);

	await db.run(`
    CREATE INDEX IF NOT EXISTS idx_users_email ON users(email)
  `);

	await db.run(`
    CREATE TABLE IF NOT EXISTS sessions (
      id TEXT PRIMARY KEY,
      user_id INTEGER NOT NULL,
      expires_at INTEGER NOT NULL,
      FOREIGN KEY (user_id) REFERENCES users(id)
    )
  `);
}

// Repository pattern
class UserRepository {
	constructor(db) {
		this.db = db;
	}

	async create(userData) {
		const result = await this.db.run(
			"INSERT INTO users (username, password, email) VALUES (?, ?, ?)",
			[userData.username, userData.password, userData.email],
		);
		return result.lastID;
	}

	async findById(id) {
		return await this.db.get("SELECT * FROM users WHERE id = ?", [id]);
	}

	async findByEmail(email) {
		return await this.db.get("SELECT * FROM users WHERE email = ?", [email]);
	}

	async findAll(limit = 100, offset = 0) {
		return await this.db.query("SELECT * FROM users LIMIT ? OFFSET ?", [limit, offset]);
	}

	async update(id, data) {
		const fields = Object.keys(data);
		const values = Object.values(data);

		const sql = `UPDATE users SET ${fields.map((f) => `${f} = ?`).join(", ")} WHERE id = ?`;

		return await this.db.run(sql, [...values, id]);
	}

	async delete(id) {
		return await this.db.run("DELETE FROM users WHERE id = ?", [id]);
	}
}

// ব্যবহার
await migrate();
const userRepo = new UserRepository(db);

const userId = await userRepo.create({
	username: "johndoe",
	password: "hashed_password",
	email: "john@example.com",
});

const user = await userRepo.findById(userId);
console.log(user);
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 19. File System Advanced

### Advanced File Operations

```javascript
const fs = require("fs").promises;
const path = require("path");
const { createReadStream, createWriteStream } = require("fs");

// Recursive directory creation
async function ensureDirectory(dirPath) {
	await fs.mkdir(dirPath, { recursive: true });
}

// Copy directory recursively
async function copyDirectory(src, dest) {
	await ensureDirectory(dest);

	const entries = await fs.readdir(src, { withFileTypes: true });

	for (const entry of entries) {
		const srcPath = path.join(src, entry.name);
		const destPath = path.join(dest, entry.name);

		if (entry.isDirectory()) {
			await copyDirectory(srcPath, destPath);
		} else {
			await fs.copyFile(srcPath, destPath);
		}
	}
}

// Delete directory recursively
async function deleteDirectory(dirPath) {
	const entries = await fs.readdir(dirPath, { withFileTypes: true });

	for (const entry of entries) {
		const fullPath = path.join(dirPath, entry.name);

		if (entry.isDirectory()) {
			await deleteDirectory(fullPath);
		} else {
			await fs.unlink(fullPath);
		}
	}

	await fs.rmdir(dirPath);
}

// Or use built-in (Node.js 14+)
async function deleteDirModern(dirPath) {
	await fs.rm(dirPath, { recursive: true, force: true });
}

// Find files by pattern
async function findFiles(dir, pattern) {
	const results = [];

	async function search(currentDir) {
		const entries = await fs.readdir(currentDir, { withFileTypes: true });

		for (const entry of entries) {
			const fullPath = path.join(currentDir, entry.name);

			if (entry.isDirectory()) {
				await search(fullPath);
			} else if (pattern.test(entry.name)) {
				results.push(fullPath);
			}
		}
	}

	await search(dir);
	return results;
}

// ব্যবহার
const jsFiles = await findFiles("./src", /\.js$/);
console.log("JavaScript files:", jsFiles);
```

### File Watching

```javascript
const fs = require("fs");
const { EventEmitter } = require("events");

class FileWatcher extends EventEmitter {
	constructor(directory) {
		super();
		this.directory = directory;
		this.watchers = new Map();
	}

	watch() {
		const watcher = fs.watch(this.directory, { recursive: true }, (eventType, filename) => {
			if (!filename) return;

			const fullPath = path.join(this.directory, filename);

			if (eventType === "change") {
				this.emit("change", fullPath);
			} else if (eventType === "rename") {
				fs.access(fullPath, fs.constants.F_OK, (err) => {
					if (err) {
						this.emit("delete", fullPath);
					} else {
						this.emit("create", fullPath);
					}
				});
			}
		});

		return watcher;
	}

	stop() {
		this.watchers.forEach((watcher) => watcher.close());
		this.watchers.clear();
	}
}

// ব্যবহার
const watcher = new FileWatcher("./src");

watcher.on("change", (file) => {
	console.log("File changed:", file);
});

watcher.on("create", (file) => {
	console.log("File created:", file);
});

watcher.on("delete", (file) => {
	console.log("File deleted:", file);
});

watcher.watch();
```

### File Locking

```javascript
const fs = require("fs").promises;
const fsSync = require("fs");

class FileLock {
	constructor(lockFile) {
		this.lockFile = lockFile;
		this.locked = false;
	}

	async acquire(timeout = 10000) {
		const startTime = Date.now();

		while (Date.now() - startTime < timeout) {
			try {
				// Try to create lock file exclusively
				await fs.writeFile(this.lockFile, process.pid.toString(), {
					flag: "wx", // Write + Exclusive
				});
				this.locked = true;
				return true;
			} catch (error) {
				if (error.code === "EEXIST") {
					// Lock exists, wait and retry
					await new Promise((resolve) => setTimeout(resolve, 100));
				} else {
					throw error;
				}
			}
		}

		throw new Error("Failed to acquire lock");
	}

	async release() {
		if (this.locked) {
			await fs.unlink(this.lockFile);
			this.locked = false;
		}
	}

	async withLock(callback) {
		await this.acquire();
		try {
			return await callback();
		} finally {
			await this.release();
		}
	}
}

// ব্যবহার
const lock = new FileLock("./data.lock");

await lock.withLock(async () => {
	// Critical section
	const data = await fs.readFile("./data.json", "utf8");
	const json = JSON.parse(data);
	json.counter++;
	await fs.writeFile("./data.json", JSON.stringify(json));
});
```

### Memory-Mapped Files

```javascript
const fs = require("fs");
const mmap = require("mmap-io");

// Large file processing with mmap
function processLargeFile(filename) {
	const fd = fs.openSync(filename, "r");
	const stats = fs.fstatSync(fd);
	const size = stats.size;

	// Memory map the file
	const buffer = mmap.map(size, mmap.PROT_READ, mmap.MAP_SHARED, fd, 0);

	// Process data
	let count = 0;
	for (let i = 0; i < buffer.length; i++) {
		if (buffer[i] === 0x0a) {
			// newline
			count++;
		}
	}

	// Cleanup
	mmap.advise(buffer, mmap.MADV_DONTNEED);
	fs.closeSync(fd);

	return count;
}
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 20. Database Connection Pooling

### MySQL Connection Pool

```javascript
const mysql = require("mysql2/promise");

// Connection pool configuration
const pool = mysql.createPool({
	host: "localhost",
	user: "root",
	password: "password",
	database: "mydb",
	waitForConnections: true,
	connectionLimit: 10,
	queueLimit: 0,
	enableKeepAlive: true,
	keepAliveInitialDelay: 0,
});

// Basic query
async function query(sql, params) {
	const [rows] = await pool.execute(sql, params);
	return rows;
}

// Transaction
async function transaction(callback) {
	const connection = await pool.getConnection();

	try {
		await connection.beginTransaction();
		const result = await callback(connection);
		await connection.commit();
		return result;
	} catch (error) {
		await connection.rollback();
		throw error;
	} finally {
		connection.release();
	}
}

// ব্যবহার
const users = await query("SELECT * FROM users WHERE age > ?", [18]);

await transaction(async (conn) => {
	await conn.execute("UPDATE accounts SET balance = balance - ? WHERE id = ?", [100, 1]);
	await conn.execute("UPDATE accounts SET balance = balance + ? WHERE id = ?", [100, 2]);
});
```

### PostgreSQL Connection Pool

```javascript
const { Pool } = require("pg");

const pool = new Pool({
	host: "localhost",
	port: 5432,
	user: "postgres",
	password: "password",
	database: "mydb",
	max: 20, // Maximum connections
	idleTimeoutMillis: 30000,
	connectionTimeoutMillis: 2000,
});

// Query helper
async function query(text, params) {
	const start = Date.now();
	const res = await pool.query(text, params);
	const duration = Date.now() - start;

	console.log("Executed query", { text, duration, rows: res.rowCount });
	return res;
}

// Transaction
async function transaction(callback) {
	const client = await pool.connect();

	try {
		await client.query("BEGIN");
		const result = await callback(client);
		await client.query("COMMIT");
		return result;
	} catch (error) {
		await client.query("ROLLBACK");
		throw error;
	} finally {
		client.release();
	}
}

// Named queries
const queries = {
	getUserById: "SELECT * FROM users WHERE id = $1",
	createUser: "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *",
	updateUser: "UPDATE users SET name = $1, email = $2 WHERE id = $3",
	deleteUser: "DELETE FROM users WHERE id = $1",
};

// Repository pattern
class UserRepository {
	async findById(id) {
		const result = await query(queries.getUserById, [id]);
		return result.rows[0];
	}

	async create(name, email) {
		const result = await query(queries.createUser, [name, email]);
		return result.rows[0];
	}

	async update(id, name, email) {
		await query(queries.updateUser, [name, email, id]);
	}

	async delete(id) {
		await query(queries.deleteUser, [id]);
	}
}
```

### Custom Connection Pool

```javascript
class ConnectionPool {
	constructor(factory, options = {}) {
		this.factory = factory;
		this.min = options.min || 2;
		this.max = options.max || 10;
		this.idleTimeout = options.idleTimeout || 30000;

		this.connections = [];
		this.available = [];
		this.pending = [];

		this.initialize();
	}

	async initialize() {
		for (let i = 0; i < this.min; i++) {
			const conn = await this.createConnection();
			this.connections.push(conn);
			this.available.push(conn);
		}
	}

	async createConnection() {
		const conn = await this.factory.create();

		conn.lastUsed = Date.now();
		conn.inUse = false;

		// Auto-cleanup idle connections
		conn.idleTimer = setTimeout(() => {
			this.removeConnection(conn);
		}, this.idleTimeout);

		return conn;
	}

	async acquire() {
		// Check available connections
		if (this.available.length > 0) {
			const conn = this.available.pop();
			conn.inUse = true;
			clearTimeout(conn.idleTimer);
			return conn;
		}

		// Create new if under max
		if (this.connections.length < this.max) {
			const conn = await this.createConnection();
			this.connections.push(conn);
			conn.inUse = true;
			return conn;
		}

		// Wait for available connection
		return new Promise((resolve) => {
			this.pending.push(resolve);
		});
	}

	release(conn) {
		conn.inUse = false;
		conn.lastUsed = Date.now();

		// Process pending requests
		if (this.pending.length > 0) {
			const resolve = this.pending.shift();
			conn.inUse = true;
			resolve(conn);
			return;
		}

		// Return to pool
		this.available.push(conn);

		// Set idle timeout
		conn.idleTimer = setTimeout(() => {
			if (!conn.inUse && this.connections.length > this.min) {
				this.removeConnection(conn);
			}
		}, this.idleTimeout);
	}

	async removeConnection(conn) {
		const index = this.connections.indexOf(conn);
		if (index > -1) {
			this.connections.splice(index, 1);
		}

		const availIndex = this.available.indexOf(conn);
		if (availIndex > -1) {
			this.available.splice(availIndex, 1);
		}

		clearTimeout(conn.idleTimer);
		await this.factory.destroy(conn);
	}

	async drain() {
		await Promise.all(this.connections.map((conn) => this.factory.destroy(conn)));

		this.connections = [];
		this.available = [];
		this.pending = [];
	}

	async with(callback) {
		const conn = await this.acquire();
		try {
			return await callback(conn);
		} finally {
			this.release(conn);
		}
	}

	getStats() {
		return {
			total: this.connections.length,
			available: this.available.length,
			inUse: this.connections.length - this.available.length,
			pending: this.pending.length,
		};
	}
}

// ব্যবহার
const dbFactory = {
	create: async () => {
		// Create database connection
		return await mysql.createConnection(config);
	},
	destroy: async (conn) => {
		await conn.end();
	},
};

const pool = new ConnectionPool(dbFactory, {
	min: 2,
	max: 10,
	idleTimeout: 30000,
});

// Use connection
await pool.with(async (conn) => {
	const [rows] = await conn.query("SELECT * FROM users");
	return rows;
});

console.log("Pool stats:", pool.getStats());
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

## 21. Test Runner

### Native Node.js Test Runner (Node 18+)

```javascript
const test = require("node:test");
const assert = require("node:assert");

// Basic test
test("addition works", () => {
	assert.strictEqual(2 + 2, 4);
});

// Async test
test("async operation", async () => {
	const result = await Promise.resolve(42);
	assert.strictEqual(result, 42);
});

// Subtests
test("user operations", async (t) => {
	await t.test("create user", () => {
		const user = { name: "John", age: 30 };
		assert.ok(user.name);
	});

	await t.test("update user", () => {
		const user = { name: "John", age: 31 };
		assert.strictEqual(user.age, 31);
	});
});

// Hooks
test("hooks example", async (t) => {
	let connection;

	t.before(() => {
		connection = createConnection();
	});

	t.after(() => {
		connection.close();
	});

	t.beforeEach(() => {
		connection.startTransaction();
	});

	t.afterEach(() => {
		connection.rollback();
	});

	await t.test("test 1", () => {
		// Test with transaction
	});
});

// Only and skip
test("only this test", { only: true }, () => {
	assert.ok(true);
});

test("skip this test", { skip: true }, () => {
	assert.ok(false);
});

// Timeout
test("slow test", { timeout: 5000 }, async () => {
	await new Promise((resolve) => setTimeout(resolve, 4000));
});
```

### Test Coverage

```javascript
// test-utils.js
class TestUtils {
	static async assertThrows(fn, expectedError) {
		let error;
		try {
			await fn();
		} catch (e) {
			error = e;
		}
		assert.ok(error, "Expected function to throw");
		if (expectedError) {
			assert.ok(error instanceof expectedError);
		}
	}

	static async assertDoesNotThrow(fn) {
		await fn(); // Should not throw
	}

	static assertDeepEqual(actual, expected) {
		assert.deepStrictEqual(actual, expected);
	}

	static mock(obj, method, implementation) {
		const original = obj[method];
		obj[method] = implementation;

		return () => {
			obj[method] = original;
		};
	}
}

// ব্যবহার
test("error handling", async () => {
	await TestUtils.assertThrows(async () => {
		throw new Error("Expected error");
	}, Error);
});
```

### Mocking

```javascript
const test = require("node:test");
const assert = require("node:assert");
const { mock } = require("node:test");

test("mocking example", async (t) => {
	// Mock function
	const mockFn = t.mock.fn((x) => x * 2);

	assert.strictEqual(mockFn(5), 10);
	assert.strictEqual(mockFn.mock.calls.length, 1);
	assert.deepStrictEqual(mockFn.mock.calls[0].arguments, [5]);

	// Mock method
	const obj = {
		method: (x) => x + 1,
	};

	t.mock.method(obj, "method", (x) => x * 2);

	assert.strictEqual(obj.method(5), 10);
});

// Custom mock implementation
class MockDatabase {
	constructor() {
		this.data = new Map();
		this.calls = [];
	}

	query(sql, params) {
		this.calls.push({ sql, params });

		// Return mock data
		if (sql.includes("SELECT")) {
			return Promise.resolve([{ id: 1, name: "John" }]);
		}

		return Promise.resolve({ affectedRows: 1 });
	}

	getCalls() {
		return this.calls;
	}

	reset() {
		this.calls = [];
	}
}

// ব্যবহার
test("database operations", async () => {
	const db = new MockDatabase();

	const results = await db.query("SELECT * FROM users", []);

	assert.strictEqual(results.length, 1);
	assert.strictEqual(db.getCalls().length, 1);
});
```

### Integration Tests

```javascript
const test = require("node:test");
const assert = require("node:assert");
const http = require("http");

// Test server
function createTestServer(handler) {
	return new Promise((resolve) => {
		const server = http.createServer(handler);
		server.listen(0, () => {
			const port = server.address().port;
			resolve({ server, port });
		});
	});
}

function request(port, path, options = {}) {
	return new Promise((resolve, reject) => {
		const req = http.request(
			{
				hostname: "localhost",
				port,
				path,
				...options,
			},
			(res) => {
				let data = "";
				res.on("data", (chunk) => (data += chunk));
				res.on("end", () => resolve({ status: res.statusCode, data }));
			},
		);

		req.on("error", reject);
		req.end();
	});
}

// Tests
test("HTTP server", async (t) => {
	let server, port;

	t.before(async () => {
		({ server, port } = await createTestServer((req, res) => {
			res.writeHead(200, { "Content-Type": "application/json" });
			res.end(JSON.stringify({ message: "Hello" }));
		}));
	});

	t.after(() => {
		server.close();
	});

	await t.test("GET request", async () => {
		const { status, data } = await request(port, "/");

		assert.strictEqual(status, 200);
		const json = JSON.parse(data);
		assert.strictEqual(json.message, "Hello");
	});
});
```

**[⬆ সূচিপত্রে ফিরে যান](#সূচিপত্র)**

---

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
