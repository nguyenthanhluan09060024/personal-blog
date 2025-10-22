---
title: "WebSocket Client với JavaScript"
date: 2024-10-19T10:00:00+07:00
draft: false
tags: ["JavaScript", "WebSocket", "Real-time", "Client", "Networking"]
description: "Hướng dẫn sử dụng WebSocket trong JavaScript để tạo ứng dụng real-time"
---

# WebSocket Client với JavaScript

WebSocket cho phép giao tiếp hai chiều real-time giữa client và server. Trong bài viết này, chúng ta sẽ tìm hiểu cách sử dụng WebSocket API trong JavaScript để xây dựng các ứng dụng real-time.

## WebSocket API cơ bản

### Kết nối WebSocket
```javascript
// Tạo kết nối WebSocket
const socket = new WebSocket('ws://localhost:8080/chat');

// Event listeners
socket.onopen = function(event) {
    console.log('Đã kết nối đến server');
};

socket.onmessage = function(event) {
    console.log('Nhận tin nhắn:', event.data);
};

socket.onclose = function(event) {
    console.log('Kết nối đã đóng:', event.code, event.reason);
};

socket.onerror = function(error) {
    console.error('Lỗi WebSocket:', error);
};
```

### Gửi dữ liệu
```javascript
// Gửi text message
socket.send('Hello Server!');

// Gửi JSON data
const message = {
    type: 'chat',
    content: 'Xin chào!',
    timestamp: Date.now()
};
socket.send(JSON.stringify(message));

// Gửi binary data
const buffer = new ArrayBuffer(8);
const view = new Uint8Array(buffer);
socket.send(buffer);
```

## Chat Application đơn giản

```html
<!DOCTYPE html>
<html>
<head>
    <title>WebSocket Chat</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        #messages { 
            height: 400px; 
            overflow-y: scroll; 
            border: 1px solid #ccc; 
            padding: 10px; 
            margin-bottom: 10px;
        }
        .message { 
            margin-bottom: 10px; 
            padding: 5px; 
            border-radius: 5px;
        }
        .message.own { 
            background-color: #e3f2fd; 
            text-align: right; 
        }
        .message.other { 
            background-color: #f5f5f5; 
        }
        #messageInput { 
            width: 70%; 
            padding: 8px; 
        }
        #sendButton { 
            width: 25%; 
            padding: 8px; 
        }
        .status { 
            margin-bottom: 10px; 
            padding: 5px; 
            border-radius: 3px;
        }
        .status.connected { background-color: #4caf50; color: white; }
        .status.disconnected { background-color: #f44336; color: white; }
    </style>
</head>
<body>
    <div id="status" class="status disconnected">Chưa kết nối</div>
    <div id="messages"></div>
    <input type="text" id="messageInput" placeholder="Nhập tin nhắn..." disabled>
    <button id="sendButton" disabled>Gửi</button>
    
    <script>
        class ChatClient {
            constructor(url) {
                this.url = url;
                this.socket = null;
                this.username = null;
                this.messageInput = document.getElementById('messageInput');
                this.sendButton = document.getElementById('sendButton');
                this.messages = document.getElementById('messages');
                this.status = document.getElementById('status');
                
                this.init();
            }
            
            init() {
                // Lấy username
                this.username = prompt('Nhập tên của bạn:') || 'Anonymous';
                
                // Event listeners
                this.sendButton.addEventListener('click', () => this.sendMessage());
                this.messageInput.addEventListener('keypress', (e) => {
                    if (e.key === 'Enter') {
                        this.sendMessage();
                    }
                });
                
                this.connect();
            }
            
            connect() {
                this.socket = new WebSocket(this.url);
                
                this.socket.onopen = () => {
                    this.updateStatus('Đã kết nối', 'connected');
                    this.messageInput.disabled = false;
                    this.sendButton.disabled = false;
                    
                    // Gửi thông tin user
                    this.send({
                        type: 'join',
                        username: this.username
                    });
                };
                
                this.socket.onmessage = (event) => {
                    const data = JSON.parse(event.data);
                    this.handleMessage(data);
                };
                
                this.socket.onclose = () => {
                    this.updateStatus('Đã ngắt kết nối', 'disconnected');
                    this.messageInput.disabled = true;
                    this.sendButton.disabled = true;
                };
                
                this.socket.onerror = (error) => {
                    console.error('WebSocket error:', error);
                    this.updateStatus('Lỗi kết nối', 'disconnected');
                };
            }
            
            sendMessage() {
                const content = this.messageInput.value.trim();
                if (content && this.socket.readyState === WebSocket.OPEN) {
                    this.send({
                        type: 'message',
                        content: content,
                        username: this.username,
                        timestamp: Date.now()
                    });
                    this.messageInput.value = '';
                }
            }
            
            send(data) {
                if (this.socket.readyState === WebSocket.OPEN) {
                    this.socket.send(JSON.stringify(data));
                }
            }
            
            handleMessage(data) {
                switch (data.type) {
                    case 'message':
                        this.addMessage(data);
                        break;
                    case 'user_joined':
                        this.addSystemMessage(`${data.username} đã tham gia chat`);
                        break;
                    case 'user_left':
                        this.addSystemMessage(`${data.username} đã rời khỏi chat`);
                        break;
                    case 'error':
                        this.addSystemMessage(`Lỗi: ${data.message}`, 'error');
                        break;
                }
            }
            
            addMessage(data) {
                const messageDiv = document.createElement('div');
                messageDiv.className = `message ${data.username === this.username ? 'own' : 'other'}`;
                
                const time = new Date(data.timestamp).toLocaleTimeString();
                messageDiv.innerHTML = `
                    <strong>${data.username}</strong> (${time}):<br>
                    ${this.escapeHtml(data.content)}
                `;
                
                this.messages.appendChild(messageDiv);
                this.messages.scrollTop = this.messages.scrollHeight;
            }
            
            addSystemMessage(message, type = 'info') {
                const messageDiv = document.createElement('div');
                messageDiv.className = `message system ${type}`;
                messageDiv.textContent = message;
                this.messages.appendChild(messageDiv);
                this.messages.scrollTop = this.messages.scrollHeight;
            }
            
            updateStatus(text, className) {
                this.status.textContent = text;
                this.status.className = `status ${className}`;
            }
            
            escapeHtml(text) {
                const div = document.createElement('div');
                div.textContent = text;
                return div.innerHTML;
            }
        }
        
        // Khởi tạo chat client
        const chat = new ChatClient('ws://localhost:8080/chat');
    </script>
</body>
</html>
```

## Reconnection và Error Handling

```javascript
class RobustWebSocket {
    constructor(url, options = {}) {
        this.url = url;
        this.options = {
            maxReconnectAttempts: 5,
            reconnectDelay: 1000,
            heartbeatInterval: 30000,
            ...options
        };
        
        this.reconnectAttempts = 0;
        this.heartbeatTimer = null;
        this.isManualClose = false;
        
        this.connect();
    }
    
    connect() {
        try {
            this.socket = new WebSocket(this.url);
            this.setupEventListeners();
        } catch (error) {
            console.error('Lỗi tạo WebSocket:', error);
            this.handleReconnect();
        }
    }
    
    setupEventListeners() {
        this.socket.onopen = () => {
            console.log('WebSocket connected');
            this.reconnectAttempts = 0;
            this.startHeartbeat();
            this.onOpen?.();
        };
        
        this.socket.onmessage = (event) => {
            // Xử lý heartbeat
            if (event.data === 'ping') {
                this.socket.send('pong');
                return;
            }
            
            this.onMessage?.(event);
        };
        
        this.socket.onclose = (event) => {
            console.log('WebSocket closed:', event.code, event.reason);
            this.stopHeartbeat();
            this.onClose?.(event);
            
            if (!this.isManualClose) {
                this.handleReconnect();
            }
        };
        
        this.socket.onerror = (error) => {
            console.error('WebSocket error:', error);
            this.onError?.(error);
        };
    }
    
    handleReconnect() {
        if (this.reconnectAttempts < this.options.maxReconnectAttempts) {
            this.reconnectAttempts++;
            const delay = this.options.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1);
            
            console.log(`Thử kết nối lại lần ${this.reconnectAttempts} sau ${delay}ms`);
            
            setTimeout(() => {
                this.connect();
            }, delay);
        } else {
            console.error('Đã thử kết nối lại tối đa số lần');
            this.onMaxReconnectAttempts?.();
        }
    }
    
    startHeartbeat() {
        this.heartbeatTimer = setInterval(() => {
            if (this.socket.readyState === WebSocket.OPEN) {
                this.socket.send('ping');
            }
        }, this.options.heartbeatInterval);
    }
    
    stopHeartbeat() {
        if (this.heartbeatTimer) {
            clearInterval(this.heartbeatTimer);
            this.heartbeatTimer = null;
        }
    }
    
    send(data) {
        if (this.socket.readyState === WebSocket.OPEN) {
            this.socket.send(data);
        } else {
            console.warn('WebSocket không sẵn sàng để gửi dữ liệu');
        }
    }
    
    close() {
        this.isManualClose = true;
        this.stopHeartbeat();
        this.socket?.close();
    }
    
    // Callbacks
    onOpen = null;
    onMessage = null;
    onClose = null;
    onError = null;
    onMaxReconnectAttempts = null;
}

// Sử dụng
const ws = new RobustWebSocket('ws://localhost:8080/chat', {
    maxReconnectAttempts: 10,
    reconnectDelay: 2000
});

ws.onOpen = () => console.log('Connected!');
ws.onMessage = (event) => console.log('Message:', event.data);
ws.onClose = () => console.log('Disconnected');
ws.onError = (error) => console.error('Error:', error);
ws.onMaxReconnectAttempts = () => console.error('Max reconnection attempts reached');
```

## Binary Data và File Transfer

```javascript
class FileTransferClient {
    constructor(socket) {
        this.socket = socket;
        this.setupEventListeners();
    }
    
    setupEventListeners() {
        this.socket.onmessage = (event) => {
            if (event.data instanceof ArrayBuffer) {
                this.handleBinaryMessage(event.data);
            } else {
                this.handleTextMessage(event.data);
            }
        };
    }
    
    sendFile(file) {
        const reader = new FileReader();
        
        reader.onload = (event) => {
            const arrayBuffer = event.target.result;
            
            // Gửi metadata trước
            this.socket.send(JSON.stringify({
                type: 'file_start',
                filename: file.name,
                size: file.size,
                type: file.type
            }));
            
            // Chia file thành chunks
            const chunkSize = 64 * 1024; // 64KB
            let offset = 0;
            
            const sendChunk = () => {
                const chunk = arrayBuffer.slice(offset, offset + chunkSize);
                
                this.socket.send(JSON.stringify({
                    type: 'file_chunk',
                    offset: offset,
                    size: chunk.byteLength
                }));
                
                this.socket.send(chunk);
                
                offset += chunkSize;
                
                if (offset < arrayBuffer.byteLength) {
                    setTimeout(sendChunk, 10); // Small delay để tránh overwhelm
                } else {
                    this.socket.send(JSON.stringify({
                        type: 'file_end'
                    }));
                }
            };
            
            sendChunk();
        };
        
        reader.readAsArrayBuffer(file);
    }
    
    handleBinaryMessage(data) {
        // Xử lý binary data nhận được
        console.log('Received binary data:', data.byteLength, 'bytes');
    }
    
    handleTextMessage(data) {
        const message = JSON.parse(data);
        
        switch (message.type) {
            case 'file_start':
                console.log('Bắt đầu nhận file:', message.filename);
                break;
            case 'file_chunk':
                console.log('Nhận chunk:', message.offset, message.size);
                break;
            case 'file_end':
                console.log('Hoàn thành nhận file');
                break;
        }
    }
}
```

## WebSocket với Authentication

```javascript
class AuthenticatedWebSocket {
    constructor(url, token) {
        this.url = url;
        this.token = token;
        this.connect();
    }
    
    connect() {
        // Thêm token vào query string
        const urlWithToken = `${this.url}?token=${this.token}`;
        this.socket = new WebSocket(urlWithToken);
        
        this.socket.onopen = () => {
            console.log('Authenticated WebSocket connected');
        };
        
        this.socket.onmessage = (event) => {
            const data = JSON.parse(event.data);
            
            if (data.type === 'auth_error') {
                console.error('Authentication failed:', data.message);
                this.handleAuthError();
            } else {
                this.handleMessage(data);
            }
        };
        
        this.socket.onclose = (event) => {
            if (event.code === 1008) { // Policy violation (auth failed)
                this.handleAuthError();
            }
        };
    }
    
    handleAuthError() {
        // Redirect to login hoặc refresh token
        localStorage.removeItem('token');
        window.location.href = '/login';
    }
    
    handleMessage(data) {
        // Xử lý message thông thường
        console.log('Message:', data);
    }
    
    send(data) {
        if (this.socket.readyState === WebSocket.OPEN) {
            this.socket.send(JSON.stringify(data));
        }
    }
}
```

## Performance và Optimization

```javascript
class OptimizedWebSocket {
    constructor(url, options = {}) {
        this.url = url;
        this.options = {
            maxMessageQueue: 100,
            messageBatchSize: 10,
            batchDelay: 100,
            ...options
        };
        
        this.messageQueue = [];
        this.batchTimer = null;
        
        this.connect();
    }
    
    send(data) {
        this.messageQueue.push(data);
        
        if (this.messageQueue.length >= this.options.messageBatchSize) {
            this.flushMessages();
        } else if (!this.batchTimer) {
            this.batchTimer = setTimeout(() => {
                this.flushMessages();
            }, this.options.batchDelay);
        }
        
        // Prevent queue overflow
        if (this.messageQueue.length > this.options.maxMessageQueue) {
            this.messageQueue.shift(); // Remove oldest message
        }
    }
    
    flushMessages() {
        if (this.messageQueue.length === 0) return;
        
        if (this.socket.readyState === WebSocket.OPEN) {
            const batch = this.messageQueue.splice(0, this.options.messageBatchSize);
            this.socket.send(JSON.stringify({
                type: 'batch',
                messages: batch
            }));
        }
        
        if (this.batchTimer) {
            clearTimeout(this.batchTimer);
            this.batchTimer = null;
        }
    }
}
```

## Kết luận

WebSocket với JavaScript cung cấp khả năng giao tiếp real-time mạnh mẽ. Một số điểm quan trọng:

- Luôn xử lý reconnection và error handling
- Sử dụng heartbeat để duy trì kết nối
- Implement authentication cho security
- Optimize performance với message batching
- Xử lý binary data cho file transfer

Trong bài viết tiếp theo, chúng ta sẽ tìm hiểu về Server-Sent Events (SSE) và so sánh với WebSocket.

