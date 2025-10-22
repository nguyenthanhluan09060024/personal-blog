---
title: "Best Practices cho Lập trình Mạng"
date: 2024-10-23T10:00:00+07:00
draft: false
tags: ["Best Practices", "Performance", "Security", "Scalability", "Networking"]
description: "Tổng hợp các best practices và patterns quan trọng trong lập trình mạng"
---

# Best Practices cho Lập trình Mạng

Sau khi đã tìm hiểu các công nghệ lập trình mạng cơ bản, bài viết này sẽ tổng hợp các best practices và patterns quan trọng để xây dựng các ứng dụng mạng hiệu quả, bảo mật và có thể mở rộng.

## 1. Error Handling và Resilience

### Circuit Breaker Pattern
```javascript
class CircuitBreaker {
    constructor(threshold = 5, timeout = 60000) {
        this.failureThreshold = threshold;
        this.timeout = timeout;
        this.failureCount = 0;
        this.lastFailureTime = null;
        this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    }
    
    async execute(operation) {
        if (this.state === 'OPEN') {
            if (Date.now() - this.lastFailureTime > this.timeout) {
                this.state = 'HALF_OPEN';
            } else {
                throw new Error('Circuit breaker is OPEN');
            }
        }
        
        try {
            const result = await operation();
            this.onSuccess();
            return result;
        } catch (error) {
            this.onFailure();
            throw error;
        }
    }
    
    onSuccess() {
        this.failureCount = 0;
        this.state = 'CLOSED';
    }
    
    onFailure() {
        this.failureCount++;
        this.lastFailureTime = Date.now();
        
        if (this.failureCount >= this.failureThreshold) {
            this.state = 'OPEN';
        }
    }
}

// Sử dụng
const circuitBreaker = new CircuitBreaker(3, 30000);

async function makeApiCall() {
    return circuitBreaker.execute(async () => {
        const response = await fetch('/api/data');
        if (!response.ok) {
            throw new Error(`HTTP ${response.status}`);
        }
        return response.json();
    });
}
```

### Retry với Exponential Backoff
```javascript
class RetryHandler {
    constructor(maxRetries = 3, baseDelay = 1000) {
        this.maxRetries = maxRetries;
        this.baseDelay = baseDelay;
    }
    
    async execute(operation, context = {}) {
        let lastError;
        
        for (let attempt = 0; attempt <= this.maxRetries; attempt++) {
            try {
                return await operation();
            } catch (error) {
                lastError = error;
                
                if (attempt === this.maxRetries) {
                    break;
                }
                
                if (!this.shouldRetry(error)) {
                    break;
                }
                
                const delay = this.calculateDelay(attempt);
                console.log(`Retry attempt ${attempt + 1} after ${delay}ms`);
                await this.sleep(delay);
            }
        }
        
        throw lastError;
    }
    
    shouldRetry(error) {
        // Retry on network errors, timeouts, 5xx errors
        if (error.code === 'ECONNRESET' || 
            error.code === 'ETIMEDOUT' ||
            error.message.includes('timeout')) {
            return true;
        }
        
        if (error.status >= 500 && error.status < 600) {
            return true;
        }
        
        return false;
    }
    
    calculateDelay(attempt) {
        // Exponential backoff with jitter
        const delay = this.baseDelay * Math.pow(2, attempt);
        const jitter = Math.random() * 0.1 * delay;
        return Math.min(delay + jitter, 30000); // Max 30 seconds
    }
    
    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}
```

## 2. Connection Pooling và Resource Management

### HTTP Connection Pool
```javascript
const http = require('http');
const https = require('https');

class ConnectionPool {
    constructor(options = {}) {
        this.maxSockets = options.maxSockets || 10;
        this.keepAlive = options.keepAlive !== false;
        this.timeout = options.timeout || 5000;
        
        this.httpAgent = new http.Agent({
            keepAlive: this.keepAlive,
            maxSockets: this.maxSockets,
            timeout: this.timeout
        });
        
        this.httpsAgent = new https.Agent({
            keepAlive: this.keepAlive,
            maxSockets: this.maxSockets,
            timeout: this.timeout
        });
    }
    
    request(url, options = {}) {
        const isHttps = url.startsWith('https://');
        const agent = isHttps ? this.httpsAgent : this.httpAgent;
        
        return new Promise((resolve, reject) => {
            const req = (isHttps ? https : http).request(url, {
                ...options,
                agent
            }, (res) => {
                let data = '';
                res.on('data', chunk => data += chunk);
                res.on('end', () => {
                    try {
                        const jsonData = JSON.parse(data);
                        resolve(jsonData);
                    } catch (error) {
                        resolve(data);
                    }
                });
            });
            
            req.on('error', reject);
            req.on('timeout', () => {
                req.destroy();
                reject(new Error('Request timeout'));
            });
            
            req.setTimeout(this.timeout);
            req.end();
        });
    }
    
    destroy() {
        this.httpAgent.destroy();
        this.httpsAgent.destroy();
    }
}
```

### WebSocket Connection Manager
```javascript
class WebSocketManager {
    constructor() {
        this.connections = new Map();
        this.rooms = new Map();
        this.heartbeatInterval = 30000;
        this.maxConnections = 1000;
        
        this.startHeartbeat();
    }
    
    addConnection(socket, userId) {
        if (this.connections.size >= this.maxConnections) {
            socket.close(1013, 'Server overloaded');
            return false;
        }
        
        this.connections.set(socket.id, {
            socket,
            userId,
            lastPing: Date.now(),
            roomId: null
        });
        
        return true;
    }
    
    removeConnection(socketId) {
        const connection = this.connections.get(socketId);
        if (connection) {
            if (connection.roomId) {
                this.leaveRoom(socketId, connection.roomId);
            }
            this.connections.delete(socketId);
        }
    }
    
    joinRoom(socketId, roomId) {
        const connection = this.connections.get(socketId);
        if (!connection) return false;
        
        // Leave current room
        if (connection.roomId) {
            this.leaveRoom(socketId, connection.roomId);
        }
        
        // Join new room
        if (!this.rooms.has(roomId)) {
            this.rooms.set(roomId, new Set());
        }
        
        this.rooms.get(roomId).add(socketId);
        connection.roomId = roomId;
        
        return true;
    }
    
    leaveRoom(socketId, roomId) {
        const room = this.rooms.get(roomId);
        if (room) {
            room.delete(socketId);
            if (room.size === 0) {
                this.rooms.delete(roomId);
            }
        }
        
        const connection = this.connections.get(socketId);
        if (connection) {
            connection.roomId = null;
        }
    }
    
    broadcastToRoom(roomId, message, excludeSocketId = null) {
        const room = this.rooms.get(roomId);
        if (!room) return;
        
        room.forEach(socketId => {
            if (socketId !== excludeSocketId) {
                const connection = this.connections.get(socketId);
                if (connection && connection.socket.readyState === 1) {
                    connection.socket.send(JSON.stringify(message));
                }
            }
        });
    }
    
    startHeartbeat() {
        setInterval(() => {
            const now = Date.now();
            const timeout = 60000; // 1 minute
            
            this.connections.forEach((connection, socketId) => {
                if (now - connection.lastPing > timeout) {
                    console.log(`Connection ${socketId} timed out`);
                    connection.socket.close(1000, 'Heartbeat timeout');
                    this.removeConnection(socketId);
                }
            });
        }, this.heartbeatInterval);
    }
    
    getStats() {
        return {
            totalConnections: this.connections.size,
            totalRooms: this.rooms.size,
            roomStats: Array.from(this.rooms.entries()).map(([roomId, members]) => ({
                roomId,
                memberCount: members.size
            }))
        };
    }
}
```

## 3. Security Best Practices

### Input Validation và Sanitization
```javascript
const validator = require('validator');
const xss = require('xss');

class InputValidator {
    static validateChatMessage(data) {
        const errors = [];
        
        if (!data.content || typeof data.content !== 'string') {
            errors.push('Content is required and must be a string');
        } else {
            // Length validation
            if (data.content.length > 1000) {
                errors.push('Content must be less than 1000 characters');
            }
            
            // XSS protection
            data.content = xss(data.content);
            
            // Remove excessive whitespace
            data.content = data.content.trim().replace(/\s+/g, ' ');
        }
        
        if (data.roomId && !validator.isAlphanumeric(data.roomId)) {
            errors.push('Invalid room ID format');
        }
        
        return {
            isValid: errors.length === 0,
            errors,
            data
        };
    }
    
    static validateFileUpload(file) {
        const errors = [];
        const maxSize = 10 * 1024 * 1024; // 10MB
        const allowedTypes = ['image/jpeg', 'image/png', 'image/gif', 'application/pdf'];
        
        if (file.size > maxSize) {
            errors.push('File size must be less than 10MB');
        }
        
        if (!allowedTypes.includes(file.type)) {
            errors.push('File type not allowed');
        }
        
        // Check file extension
        const allowedExtensions = ['.jpg', '.jpeg', '.png', '.gif', '.pdf'];
        const extension = path.extname(file.name).toLowerCase();
        if (!allowedExtensions.includes(extension)) {
            errors.push('File extension not allowed');
        }
        
        return {
            isValid: errors.length === 0,
            errors
        };
    }
}
```

### Rate Limiting
```javascript
class RateLimiter {
    constructor() {
        this.requests = new Map();
        this.cleanupInterval = 60000; // 1 minute
        
        setInterval(() => this.cleanup(), this.cleanupInterval);
    }
    
    isAllowed(identifier, limit = 100, windowMs = 60000) {
        const now = Date.now();
        const windowStart = now - windowMs;
        
        if (!this.requests.has(identifier)) {
            this.requests.set(identifier, []);
        }
        
        const userRequests = this.requests.get(identifier);
        
        // Remove old requests
        const recentRequests = userRequests.filter(time => time > windowStart);
        this.requests.set(identifier, recentRequests);
        
        if (recentRequests.length >= limit) {
            return false;
        }
        
        recentRequests.push(now);
        return true;
    }
    
    cleanup() {
        const now = Date.now();
        const maxAge = 300000; // 5 minutes
        
        for (const [identifier, requests] of this.requests.entries()) {
            const recentRequests = requests.filter(time => now - time < maxAge);
            
            if (recentRequests.length === 0) {
                this.requests.delete(identifier);
            } else {
                this.requests.set(identifier, recentRequests);
            }
        }
    }
}

// Middleware for Express
function rateLimitMiddleware(limit = 100, windowMs = 60000) {
    const rateLimiter = new RateLimiter();
    
    return (req, res, next) => {
        const identifier = req.ip || req.connection.remoteAddress;
        
        if (!rateLimiter.isAllowed(identifier, limit, windowMs)) {
            return res.status(429).json({
                error: 'Too many requests',
                retryAfter: Math.ceil(windowMs / 1000)
            });
        }
        
        next();
    };
}
```

## 4. Performance Optimization

### Caching Strategy
```javascript
const NodeCache = require('node-cache');

class CacheManager {
    constructor() {
        this.cache = new NodeCache({
            stdTTL: 300, // 5 minutes default
            checkperiod: 120, // 2 minutes
            useClones: false
        });
        
        this.cache.on('set', (key, value) => {
            console.log(`Cache SET: ${key}`);
        });
        
        this.cache.on('del', (key, value) => {
            console.log(`Cache DEL: ${key}`);
        });
    }
    
    async get(key, fallbackFunction, ttl = 300) {
        let value = this.cache.get(key);
        
        if (value === undefined) {
            console.log(`Cache MISS: ${key}`);
            value = await fallbackFunction();
            this.cache.set(key, value, ttl);
        } else {
            console.log(`Cache HIT: ${key}`);
        }
        
        return value;
    }
    
    set(key, value, ttl = 300) {
        this.cache.set(key, value, ttl);
    }
    
    del(key) {
        this.cache.del(key);
    }
    
    clear() {
        this.cache.flushAll();
    }
    
    getStats() {
        return this.cache.getStats();
    }
}

// Usage example
const cache = new CacheManager();

async function getUserData(userId) {
    return cache.get(`user:${userId}`, async () => {
        // Expensive database query
        return await database.getUserById(userId);
    }, 600); // 10 minutes TTL
}
```

### Message Batching
```javascript
class MessageBatcher {
    constructor(batchSize = 10, flushInterval = 100) {
        this.batchSize = batchSize;
        this.flushInterval = flushInterval;
        this.batches = new Map();
        this.timers = new Map();
    }
    
    addMessage(roomId, message) {
        if (!this.batches.has(roomId)) {
            this.batches.set(roomId, []);
        }
        
        const batch = this.batches.get(roomId);
        batch.push(message);
        
        if (batch.length >= this.batchSize) {
            this.flushBatch(roomId);
        } else if (!this.timers.has(roomId)) {
            const timer = setTimeout(() => {
                this.flushBatch(roomId);
            }, this.flushInterval);
            this.timers.set(roomId, timer);
        }
    }
    
    flushBatch(roomId) {
        const batch = this.batches.get(roomId);
        if (!batch || batch.length === 0) return;
        
        // Send batch to room
        this.sendBatchToRoom(roomId, batch);
        
        // Clear batch and timer
        this.batches.set(roomId, []);
        if (this.timers.has(roomId)) {
            clearTimeout(this.timers.get(roomId));
            this.timers.delete(roomId);
        }
    }
    
    sendBatchToRoom(roomId, messages) {
        // Implementation depends on your WebSocket library
        console.log(`Sending batch of ${messages.length} messages to room ${roomId}`);
    }
}
```

## 5. Monitoring và Logging

### Structured Logging
```javascript
const winston = require('winston');

class Logger {
    constructor() {
        this.logger = winston.createLogger({
            level: process.env.LOG_LEVEL || 'info',
            format: winston.format.combine(
                winston.format.timestamp(),
                winston.format.errors({ stack: true }),
                winston.format.json()
            ),
            transports: [
                new winston.transports.File({ filename: 'error.log', level: 'error' }),
                new winston.transports.File({ filename: 'combined.log' }),
                new winston.transports.Console({
                    format: winston.format.simple()
                })
            ]
        });
    }
    
    logConnection(socketId, event, metadata = {}) {
        this.logger.info('WebSocket connection', {
            socketId,
            event,
            timestamp: new Date().toISOString(),
            ...metadata
        });
    }
    
    logMessage(roomId, userId, messageType, metadata = {}) {
        this.logger.info('Message processed', {
            roomId,
            userId,
            messageType,
            timestamp: new Date().toISOString(),
            ...metadata
        });
    }
    
    logError(error, context = {}) {
        this.logger.error('Error occurred', {
            error: error.message,
            stack: error.stack,
            timestamp: new Date().toISOString(),
            ...context
        });
    }
    
    logPerformance(operation, duration, metadata = {}) {
        this.logger.info('Performance metric', {
            operation,
            duration,
            timestamp: new Date().toISOString(),
            ...metadata
        });
    }
}
```

### Health Checks
```javascript
class HealthChecker {
    constructor() {
        this.checks = new Map();
        this.status = 'healthy';
    }
    
    addCheck(name, checkFunction, timeout = 5000) {
        this.checks.set(name, {
            check: checkFunction,
            timeout,
            lastCheck: null,
            status: 'unknown'
        });
    }
    
    async runChecks() {
        const results = {};
        let allHealthy = true;
        
        for (const [name, checkConfig] of this.checks.entries()) {
            try {
                const startTime = Date.now();
                const result = await Promise.race([
                    checkConfig.check(),
                    new Promise((_, reject) => 
                        setTimeout(() => reject(new Error('Check timeout')), checkConfig.timeout)
                    )
                ]);
                
                const duration = Date.now() - startTime;
                
                results[name] = {
                    status: 'healthy',
                    duration,
                    result
                };
                
                checkConfig.status = 'healthy';
                checkConfig.lastCheck = new Date();
                
            } catch (error) {
                results[name] = {
                    status: 'unhealthy',
                    error: error.message
                };
                
                checkConfig.status = 'unhealthy';
                checkConfig.lastCheck = new Date();
                allHealthy = false;
            }
        }
        
        this.status = allHealthy ? 'healthy' : 'unhealthy';
        return {
            status: this.status,
            checks: results,
            timestamp: new Date().toISOString()
        };
    }
    
    getStatus() {
        return {
            status: this.status,
            checks: Array.from(this.checks.entries()).map(([name, config]) => ({
                name,
                status: config.status,
                lastCheck: config.lastCheck
            }))
        };
    }
}

// Usage
const healthChecker = new HealthChecker();

healthChecker.addCheck('database', async () => {
    // Check database connection
    return await database.ping();
});

healthChecker.addCheck('redis', async () => {
    // Check Redis connection
    return await redis.ping();
});

// Express endpoint
app.get('/health', async (req, res) => {
    const health = await healthChecker.runChecks();
    res.status(health.status === 'healthy' ? 200 : 503).json(health);
});
```

## 6. Testing Strategies

### WebSocket Testing
```javascript
const WebSocket = require('ws');

describe('WebSocket Server', () => {
    let server;
    let ws;
    
    beforeEach((done) => {
        server = new WebSocket.Server({ port: 8080 });
        server.on('listening', done);
    });
    
    afterEach(() => {
        if (ws) {
            ws.close();
        }
        server.close();
    });
    
    it('should handle connection', (done) => {
        ws = new WebSocket('ws://localhost:8080');
        
        ws.on('open', () => {
            expect(ws.readyState).toBe(WebSocket.OPEN);
            done();
        });
    });
    
    it('should handle messages', (done) => {
        ws = new WebSocket('ws://localhost:8080');
        
        ws.on('open', () => {
            ws.send(JSON.stringify({ type: 'test', data: 'hello' }));
        });
        
        ws.on('message', (data) => {
            const message = JSON.parse(data);
            expect(message.type).toBe('echo');
            expect(message.data).toBe('hello');
            done();
        });
    });
});
```

## Kết luận

Các best practices quan trọng cho lập trình mạng:

1. **Error Handling**: Circuit breaker, retry logic, graceful degradation
2. **Resource Management**: Connection pooling, proper cleanup
3. **Security**: Input validation, rate limiting, authentication
4. **Performance**: Caching, message batching, optimization
5. **Monitoring**: Structured logging, health checks, metrics
6. **Testing**: Unit tests, integration tests, load testing

Áp dụng những practices này sẽ giúp bạn xây dựng các ứng dụng mạng ổn định, bảo mật và có thể mở rộng.

