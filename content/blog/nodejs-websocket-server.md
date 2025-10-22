---
title: "WebSocket Server với Node.js"
date: 2024-10-22T10:00:00+07:00
draft: false
tags: ["Node.js", "WebSocket", "Server", "Real-time", "Socket.IO"]
description: "Hướng dẫn xây dựng WebSocket server với Node.js và Socket.IO"
---

# WebSocket Server với Node.js

Node.js là một platform tuyệt vời để xây dựng WebSocket servers nhờ vào event-driven architecture và non-blocking I/O. Trong bài viết này, chúng ta sẽ tìm hiểu cách xây dựng WebSocket server với Node.js và Socket.IO.

## WebSocket với ws library

### Cài đặt dependencies
```bash
npm init -y
npm install ws express
```

### Basic WebSocket Server
```javascript
const WebSocket = require('ws');
const express = require('express');
const http = require('http');

const app = express();
const server = http.createServer(app);
const wss = new WebSocket.Server({ server });

// Serve static files
app.use(express.static('public'));

// Store connected clients
const clients = new Map();

wss.on('connection', (ws, req) => {
    const clientId = generateClientId();
    clients.set(clientId, {
        ws: ws,
        id: clientId,
        ip: req.socket.remoteAddress,
        connectedAt: new Date()
    });
    
    console.log(`Client ${clientId} connected from ${req.socket.remoteAddress}`);
    
    // Send welcome message
    ws.send(JSON.stringify({
        type: 'welcome',
        clientId: clientId,
        message: 'Chào mừng đến với WebSocket server!'
    }));
    
    // Handle messages
    ws.on('message', (data) => {
        try {
            const message = JSON.parse(data);
            handleMessage(clientId, message);
        } catch (error) {
            console.error('Error parsing message:', error);
            ws.send(JSON.stringify({
                type: 'error',
                message: 'Invalid message format'
            }));
        }
    });
    
    // Handle disconnection
    ws.on('close', () => {
        console.log(`Client ${clientId} disconnected`);
        clients.delete(clientId);
        broadcastUserList();
    });
    
    // Handle errors
    ws.on('error', (error) => {
        console.error(`Error with client ${clientId}:`, error);
    });
    
    // Send current user list
    broadcastUserList();
});

function handleMessage(clientId, message) {
    const client = clients.get(clientId);
    if (!client) return;
    
    switch (message.type) {
        case 'chat':
            broadcastMessage(clientId, message.content);
            break;
        case 'private':
            sendPrivateMessage(clientId, message.to, message.content);
            break;
        case 'ping':
            client.ws.send(JSON.stringify({ type: 'pong' }));
            break;
        default:
            console.log(`Unknown message type: ${message.type}`);
    }
}

function broadcastMessage(fromClientId, content) {
    const fromClient = clients.get(fromClientId);
    if (!fromClient) return;
    
    const message = {
        type: 'chat',
        from: fromClientId,
        content: content,
        timestamp: new Date().toISOString()
    };
    
    clients.forEach((client, clientId) => {
        if (client.ws.readyState === WebSocket.OPEN) {
            client.ws.send(JSON.stringify(message));
        }
    });
    
    console.log(`Broadcast from ${fromClientId}: ${content}`);
}

function sendPrivateMessage(fromClientId, toClientId, content) {
    const fromClient = clients.get(fromClientId);
    const toClient = clients.get(toClientId);
    
    if (!fromClient || !toClient) return;
    
    const message = {
        type: 'private',
        from: fromClientId,
        to: toClientId,
        content: content,
        timestamp: new Date().toISOString()
    };
    
    // Send to recipient
    if (toClient.ws.readyState === WebSocket.OPEN) {
        toClient.ws.send(JSON.stringify(message));
    }
    
    // Send confirmation to sender
    if (fromClient.ws.readyState === WebSocket.OPEN) {
        fromClient.ws.send(JSON.stringify({
            type: 'private_sent',
            to: toClientId,
            content: content
        }));
    }
    
    console.log(`Private message from ${fromClientId} to ${toClientId}: ${content}`);
}

function broadcastUserList() {
    const userList = Array.from(clients.values()).map(client => ({
        id: client.id,
        connectedAt: client.connectedAt
    }));
    
    const message = {
        type: 'user_list',
        users: userList
    };
    
    clients.forEach((client) => {
        if (client.ws.readyState === WebSocket.OPEN) {
            client.ws.send(JSON.stringify(message));
        }
    });
}

function generateClientId() {
    return Math.random().toString(36).substr(2, 9);
}

// Health check endpoint
app.get('/health', (req, res) => {
    res.json({
        status: 'ok',
        connectedClients: clients.size,
        uptime: process.uptime()
    });
});

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
    console.log(`WebSocket server running on port ${PORT}`);
    console.log(`Health check: http://localhost:${PORT}/health`);
});
```

## Socket.IO Server

### Cài đặt Socket.IO
```bash
npm install socket.io
```

### Socket.IO Server Implementation
```javascript
const express = require('express');
const http = require('http');
const socketIo = require('socket.io');
const cors = require('cors');

const app = express();
const server = http.createServer(app);
const io = socketIo(server, {
    cors: {
        origin: "*",
        methods: ["GET", "POST"]
    }
});

app.use(cors());
app.use(express.json());

// Store rooms and users
const rooms = new Map();
const users = new Map();

io.on('connection', (socket) => {
    console.log('User connected:', socket.id);
    
    // Join room
    socket.on('join-room', (data) => {
        const { roomId, username } = data;
        
        socket.join(roomId);
        socket.roomId = roomId;
        socket.username = username;
        
        // Store user info
        users.set(socket.id, {
            id: socket.id,
            username: username,
            roomId: roomId,
            connectedAt: new Date()
        });
        
        // Initialize room if not exists
        if (!rooms.has(roomId)) {
            rooms.set(roomId, new Set());
        }
        rooms.get(roomId).add(socket.id);
        
        // Notify others in room
        socket.to(roomId).emit('user-joined', {
            userId: socket.id,
            username: username
        });
        
        // Send current users in room
        const roomUsers = Array.from(rooms.get(roomId))
            .map(userId => users.get(userId))
            .filter(user => user);
        
        socket.emit('room-users', roomUsers);
        
        console.log(`${username} joined room ${roomId}`);
    });
    
    // Handle chat messages
    socket.on('chat-message', (data) => {
        const user = users.get(socket.id);
        if (!user) return;
        
        const message = {
            id: generateMessageId(),
            userId: socket.id,
            username: user.username,
            content: data.content,
            timestamp: new Date().toISOString(),
            type: 'text'
        };
        
        // Broadcast to room
        io.to(socket.roomId).emit('chat-message', message);
        
        // Store message in room history
        storeMessage(socket.roomId, message);
        
        console.log(`Message in ${socket.roomId} from ${user.username}: ${data.content}`);
    });
    
    // Handle typing indicators
    socket.on('typing-start', () => {
        const user = users.get(socket.id);
        if (user) {
            socket.to(socket.roomId).emit('user-typing', {
                userId: socket.id,
                username: user.username
            });
        }
    });
    
    socket.on('typing-stop', () => {
        socket.to(socket.roomId).emit('user-stop-typing', {
            userId: socket.id
        });
    });
    
    // Handle file uploads
    socket.on('file-upload', (data) => {
        const user = users.get(socket.id);
        if (!user) return;
        
        const message = {
            id: generateMessageId(),
            userId: socket.id,
            username: user.username,
            content: data.filename,
            fileData: data.fileData,
            fileType: data.fileType,
            fileSize: data.fileSize,
            timestamp: new Date().toISOString(),
            type: 'file'
        };
        
        io.to(socket.roomId).emit('file-upload', message);
        storeMessage(socket.roomId, message);
        
        console.log(`File uploaded in ${socket.roomId} by ${user.username}: ${data.filename}`);
    });
    
    // Handle disconnection
    socket.on('disconnect', () => {
        const user = users.get(socket.id);
        if (user) {
            console.log(`${user.username} disconnected from room ${user.roomId}`);
            
            // Remove from room
            if (rooms.has(user.roomId)) {
                rooms.get(user.roomId).delete(socket.id);
                
                // Notify others
                socket.to(user.roomId).emit('user-left', {
                    userId: socket.id,
                    username: user.username
                });
                
                // Clean up empty rooms
                if (rooms.get(user.roomId).size === 0) {
                    rooms.delete(user.roomId);
                }
            }
            
            users.delete(socket.id);
        }
    });
});

// Store message history
const messageHistory = new Map();

function storeMessage(roomId, message) {
    if (!messageHistory.has(roomId)) {
        messageHistory.set(roomId, []);
    }
    
    const history = messageHistory.get(roomId);
    history.push(message);
    
    // Keep only last 100 messages per room
    if (history.length > 100) {
        history.shift();
    }
}

// API endpoints
app.get('/api/rooms', (req, res) => {
    const roomList = Array.from(rooms.entries()).map(([roomId, users]) => ({
        roomId,
        userCount: users.size
    }));
    res.json(roomList);
});

app.get('/api/rooms/:roomId/messages', (req, res) => {
    const { roomId } = req.params;
    const messages = messageHistory.get(roomId) || [];
    res.json(messages);
});

app.get('/api/rooms/:roomId/users', (req, res) => {
    const { roomId } = req.params;
    const roomUsers = rooms.get(roomId) ? 
        Array.from(rooms.get(roomId)).map(userId => users.get(userId)).filter(user => user) : [];
    res.json(roomUsers);
});

function generateMessageId() {
    return Date.now().toString(36) + Math.random().toString(36).substr(2);
}

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
    console.log(`Socket.IO server running on port ${PORT}`);
});
```

## Authentication và Authorization

```javascript
const jwt = require('jsonwebtoken');

// Middleware for Socket.IO authentication
io.use((socket, next) => {
    const token = socket.handshake.auth.token;
    
    if (!token) {
        return next(new Error('Authentication error: No token provided'));
    }
    
    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        socket.userId = decoded.userId;
        socket.username = decoded.username;
        next();
    } catch (error) {
        next(new Error('Authentication error: Invalid token'));
    }
});

// Protected room joining
socket.on('join-room', (data) => {
    const { roomId } = data;
    
    // Check if user has permission to join this room
    if (!hasRoomPermission(socket.userId, roomId)) {
        socket.emit('error', { message: 'Permission denied' });
        return;
    }
    
    // Join room logic...
});

function hasRoomPermission(userId, roomId) {
    // Implement your permission logic here
    return true; // Simplified for example
}
```

## Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');

// Rate limiting for API endpoints
const apiLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100 // limit each IP to 100 requests per windowMs
});

app.use('/api/', apiLimiter);

// Rate limiting for Socket.IO events
const messageRateLimit = new Map();

socket.on('chat-message', (data) => {
    const userId = socket.userId;
    const now = Date.now();
    
    if (!messageRateLimit.has(userId)) {
        messageRateLimit.set(userId, []);
    }
    
    const userMessages = messageRateLimit.get(userId);
    
    // Remove messages older than 1 minute
    const oneMinuteAgo = now - 60000;
    const recentMessages = userMessages.filter(timestamp => timestamp > oneMinuteAgo);
    
    // Check if user is sending too many messages
    if (recentMessages.length >= 10) { // Max 10 messages per minute
        socket.emit('error', { message: 'Rate limit exceeded' });
        return;
    }
    
    recentMessages.push(now);
    messageRateLimit.set(userId, recentMessages);
    
    // Process message...
});
```

## Monitoring và Logging

```javascript
const winston = require('winston');

// Configure logger
const logger = winston.createLogger({
    level: 'info',
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
    ),
    transports: [
        new winston.transports.File({ filename: 'error.log', level: 'error' }),
        new winston.transports.File({ filename: 'combined.log' }),
        new winston.transports.Console()
    ]
});

// Log WebSocket events
io.on('connection', (socket) => {
    logger.info('User connected', { socketId: socket.id });
    
    socket.on('chat-message', (data) => {
        logger.info('Chat message', {
            socketId: socket.id,
            roomId: socket.roomId,
            messageLength: data.content.length
        });
    });
    
    socket.on('disconnect', () => {
        logger.info('User disconnected', { socketId: socket.id });
    });
});

// Health check with metrics
app.get('/health', (req, res) => {
    const metrics = {
        status: 'ok',
        timestamp: new Date().toISOString(),
        uptime: process.uptime(),
        memory: process.memoryUsage(),
        connectedClients: io.engine.clientsCount,
        rooms: rooms.size,
        users: users.size
    };
    
    res.json(metrics);
});
```

## Scaling với Redis

```javascript
const redis = require('redis');
const { createAdapter } = require('@socket.io/redis-adapter');

// Redis configuration
const pubClient = redis.createClient({
    host: process.env.REDIS_HOST || 'localhost',
    port: process.env.REDIS_PORT || 6379
});

const subClient = pubClient.duplicate();

// Use Redis adapter for scaling
io.adapter(createAdapter(pubClient, subClient));

// Store room data in Redis
async function storeRoomData(roomId, data) {
    await pubClient.hset(`room:${roomId}`, data);
    await pubClient.expire(`room:${roomId}`, 3600); // 1 hour TTL
}

async function getRoomData(roomId) {
    return await pubClient.hgetall(`room:${roomId}`);
}
```

## Kết luận

Node.js cung cấp một nền tảng mạnh mẽ để xây dựng WebSocket servers. Một số điểm quan trọng:

- Sử dụng Socket.IO cho features nâng cao
- Implement authentication và authorization
- Rate limiting để tránh spam
- Monitoring và logging
- Scaling với Redis cho multiple instances
- Error handling và reconnection logic

WebSocket servers phù hợp cho:
- Real-time chat applications
- Live notifications
- Collaborative editing
- Gaming
- Live streaming

Trong bài viết cuối cùng, chúng ta sẽ tìm hiểu về performance optimization và best practices cho các ứng dụng real-time.

