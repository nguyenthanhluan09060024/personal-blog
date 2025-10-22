---
title: "Server-Sent Events (SSE) với JavaScript"
date: 2024-10-20T10:00:00+07:00
draft: false
tags: ["JavaScript", "SSE", "Server-Sent Events", "Real-time", "Streaming"]
description: "Hướng dẫn sử dụng Server-Sent Events để nhận dữ liệu real-time từ server"
---

# Server-Sent Events (SSE) với JavaScript

Server-Sent Events (SSE) là một công nghệ cho phép server gửi dữ liệu real-time đến client thông qua HTTP connection. Khác với WebSocket, SSE chỉ cho phép server gửi dữ liệu đến client (one-way communication).

## SSE vs WebSocket vs Polling

| Đặc điểm | SSE | WebSocket | Polling |
|----------|-----|-----------|---------|
| Hướng | Server → Client | Bidirectional | Bidirectional |
| Protocol | HTTP | WebSocket | HTTP |
| Reconnection | Tự động | Manual | Manual |
| Browser Support | Tốt | Tốt | Tốt |
| Complexity | Thấp | Trung bình | Thấp |

## EventSource API cơ bản

### Kết nối SSE
```javascript
// Tạo kết nối SSE
const eventSource = new EventSource('/api/events');

// Event listeners
eventSource.onopen = function(event) {
    console.log('SSE connection opened');
};

eventSource.onmessage = function(event) {
    console.log('Message received:', event.data);
    console.log('Event type:', event.type);
    console.log('Last event ID:', event.lastEventId);
};

eventSource.onerror = function(event) {
    console.error('SSE error:', event);
};

// Đóng kết nối
eventSource.close();
```

### Xử lý các loại events khác nhau
```javascript
const eventSource = new EventSource('/api/events');

// Xử lý custom event types
eventSource.addEventListener('user_joined', function(event) {
    const data = JSON.parse(event.data);
    console.log('User joined:', data.username);
    addUserToList(data);
});

eventSource.addEventListener('message', function(event) {
    const data = JSON.parse(event.data);
    console.log('New message:', data.content);
    displayMessage(data);
});

eventSource.addEventListener('notification', function(event) {
    const data = JSON.parse(event.data);
    showNotification(data.message);
});
```

## Real-time Notifications System

```html
<!DOCTYPE html>
<html>
<head>
    <title>Real-time Notifications</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .notification { 
            padding: 10px; 
            margin: 5px 0; 
            border-radius: 5px; 
            border-left: 4px solid #007bff;
            background-color: #f8f9fa;
        }
        .notification.success { border-left-color: #28a745; }
        .notification.warning { border-left-color: #ffc107; }
        .notification.error { border-left-color: #dc3545; }
        .notification-header { 
            font-weight: bold; 
            margin-bottom: 5px; 
        }
        .notification-time { 
            font-size: 0.8em; 
            color: #666; 
        }
        .status { 
            padding: 10px; 
            margin-bottom: 20px; 
            border-radius: 5px;
        }
        .status.connected { background-color: #d4edda; color: #155724; }
        .status.disconnected { background-color: #f8d7da; color: #721c24; }
        #notifications { 
            max-height: 400px; 
            overflow-y: auto; 
        }
    </style>
</head>
<body>
    <div id="status" class="status disconnected">Chưa kết nối</div>
    <h2>Thông báo real-time</h2>
    <div id="notifications"></div>
    
    <script>
        class NotificationSystem {
            constructor() {
                this.eventSource = null;
                this.statusElement = document.getElementById('status');
                this.notificationsElement = document.getElementById('notifications');
                this.reconnectAttempts = 0;
                this.maxReconnectAttempts = 5;
                
                this.connect();
            }
            
            connect() {
                this.eventSource = new EventSource('/api/notifications');
                
                this.eventSource.onopen = () => {
                    console.log('SSE connection opened');
                    this.updateStatus('Đã kết nối', 'connected');
                    this.reconnectAttempts = 0;
                };
                
                this.eventSource.onmessage = (event) => {
                    this.handleMessage(event);
                };
                
                this.eventSource.onerror = (event) => {
                    console.error('SSE error:', event);
                    this.updateStatus('Lỗi kết nối', 'disconnected');
                    this.handleReconnect();
                };
                
                // Xử lý custom events
                this.eventSource.addEventListener('notification', (event) => {
                    this.handleNotification(JSON.parse(event.data));
                });
                
                this.eventSource.addEventListener('system', (event) => {
                    this.handleSystemEvent(JSON.parse(event.data));
                });
            }
            
            handleMessage(event) {
                try {
                    const data = JSON.parse(event.data);
                    this.handleNotification(data);
                } catch (error) {
                    console.error('Error parsing message:', error);
                }
            }
            
            handleNotification(data) {
                const notification = this.createNotificationElement(data);
                this.notificationsElement.insertBefore(notification, this.notificationsElement.firstChild);
                
                // Giới hạn số lượng notifications
                const notifications = this.notificationsElement.children;
                if (notifications.length > 50) {
                    this.notificationsElement.removeChild(notifications[notifications.length - 1]);
                }
                
                // Auto remove sau 10 giây
                setTimeout(() => {
                    if (notification.parentNode) {
                        notification.parentNode.removeChild(notification);
                    }
                }, 10000);
            }
            
            handleSystemEvent(data) {
                switch (data.type) {
                    case 'maintenance':
                        this.showMaintenanceNotification(data);
                        break;
                    case 'update':
                        this.showUpdateNotification(data);
                        break;
                }
            }
            
            createNotificationElement(data) {
                const div = document.createElement('div');
                div.className = `notification ${data.level || 'info'}`;
                
                const time = new Date(data.timestamp).toLocaleTimeString();
                div.innerHTML = `
                    <div class="notification-header">${this.escapeHtml(data.title)}</div>
                    <div class="notification-content">${this.escapeHtml(data.message)}</div>
                    <div class="notification-time">${time}</div>
                `;
                
                // Click để đóng
                div.addEventListener('click', () => {
                    div.remove();
                });
                
                return div;
            }
            
            showMaintenanceNotification(data) {
                const notification = {
                    title: 'Bảo trì hệ thống',
                    message: data.message,
                    level: 'warning',
                    timestamp: Date.now()
                };
                this.handleNotification(notification);
            }
            
            showUpdateNotification(data) {
                const notification = {
                    title: 'Cập nhật hệ thống',
                    message: data.message,
                    level: 'success',
                    timestamp: Date.now()
                };
                this.handleNotification(notification);
            }
            
            handleReconnect() {
                if (this.reconnectAttempts < this.maxReconnectAttempts) {
                    this.reconnectAttempts++;
                    const delay = Math.pow(2, this.reconnectAttempts) * 1000;
                    
                    console.log(`Thử kết nối lại lần ${this.reconnectAttempts} sau ${delay}ms`);
                    
                    setTimeout(() => {
                        this.connect();
                    }, delay);
                } else {
                    console.error('Đã thử kết nối lại tối đa số lần');
                    this.updateStatus('Không thể kết nối', 'disconnected');
                }
            }
            
            updateStatus(text, className) {
                this.statusElement.textContent = text;
                this.statusElement.className = `status ${className}`;
            }
            
            escapeHtml(text) {
                const div = document.createElement('div');
                div.textContent = text;
                return div.innerHTML;
            }
            
            disconnect() {
                if (this.eventSource) {
                    this.eventSource.close();
                    this.eventSource = null;
                }
            }
        }
        
        // Khởi tạo notification system
        const notificationSystem = new NotificationSystem();
        
        // Cleanup khi trang bị đóng
        window.addEventListener('beforeunload', () => {
            notificationSystem.disconnect();
        });
    </script>
</body>
</html>
```

## Live Data Dashboard

```javascript
class LiveDashboard {
    constructor() {
        this.eventSource = null;
        this.charts = {};
        this.data = {};
        
        this.connect();
    }
    
    connect() {
        this.eventSource = new EventSource('/api/dashboard/stream');
        
        this.eventSource.onopen = () => {
            console.log('Dashboard SSE connected');
        };
        
        this.eventSource.onmessage = (event) => {
            this.handleData(JSON.parse(event.data));
        };
        
        // Xử lý các loại data khác nhau
        this.eventSource.addEventListener('metrics', (event) => {
            this.updateMetrics(JSON.parse(event.data));
        });
        
        this.eventSource.addEventListener('chart_data', (event) => {
            this.updateChart(JSON.parse(event.data));
        });
        
        this.eventSource.addEventListener('alert', (event) => {
            this.handleAlert(JSON.parse(event.data));
        });
    }
    
    handleData(data) {
        this.data[data.type] = data.value;
        this.updateUI();
    }
    
    updateMetrics(data) {
        // Cập nhật các metrics
        document.getElementById('cpu-usage').textContent = data.cpu + '%';
        document.getElementById('memory-usage').textContent = data.memory + '%';
        document.getElementById('active-users').textContent = data.activeUsers;
        
        // Cập nhật progress bars
        this.updateProgressBar('cpu-progress', data.cpu);
        this.updateProgressBar('memory-progress', data.memory);
    }
    
    updateChart(data) {
        if (this.charts[data.chartId]) {
            this.charts[data.chartId].addData(data.point);
        }
    }
    
    updateProgressBar(elementId, value) {
        const element = document.getElementById(elementId);
        if (element) {
            element.style.width = value + '%';
            element.className = value > 80 ? 'progress-bar danger' : 
                               value > 60 ? 'progress-bar warning' : 'progress-bar';
        }
    }
    
    handleAlert(data) {
        this.showAlert(data.message, data.level);
    }
    
    showAlert(message, level = 'info') {
        const alertDiv = document.createElement('div');
        alertDiv.className = `alert alert-${level}`;
        alertDiv.textContent = message;
        
        document.body.insertBefore(alertDiv, document.body.firstChild);
        
        setTimeout(() => {
            alertDiv.remove();
        }, 5000);
    }
    
    updateUI() {
        // Cập nhật UI dựa trên data hiện tại
        console.log('Current data:', this.data);
    }
}
```

## SSE với Authentication

```javascript
class AuthenticatedSSE {
    constructor(url, token) {
        this.url = url;
        this.token = token;
        this.eventSource = null;
        this.connect();
    }
    
    connect() {
        // Thêm token vào headers thông qua URL parameters
        const urlWithToken = `${this.url}?token=${this.token}`;
        this.eventSource = new EventSource(urlWithToken);
        
        this.eventSource.onopen = () => {
            console.log('Authenticated SSE connected');
        };
        
        this.eventSource.onmessage = (event) => {
            const data = JSON.parse(event.data);
            
            if (data.type === 'auth_error') {
                this.handleAuthError(data.message);
            } else {
                this.handleMessage(data);
            }
        };
        
        this.eventSource.onerror = (event) => {
            console.error('SSE error:', event);
        };
    }
    
    handleAuthError(message) {
        console.error('Authentication failed:', message);
        this.eventSource.close();
        
        // Redirect to login
        localStorage.removeItem('token');
        window.location.href = '/login';
    }
    
    handleMessage(data) {
        console.log('Message received:', data);
    }
    
    disconnect() {
        if (this.eventSource) {
            this.eventSource.close();
        }
    }
}

// Sử dụng
const token = localStorage.getItem('token');
const sse = new AuthenticatedSSE('/api/events', token);
```

## Error Handling và Reconnection

```javascript
class RobustSSE {
    constructor(url, options = {}) {
        this.url = url;
        this.options = {
            maxReconnectAttempts: 5,
            reconnectDelay: 1000,
            heartbeatTimeout: 30000,
            ...options
        };
        
        this.reconnectAttempts = 0;
        this.heartbeatTimer = null;
        this.lastMessageTime = Date.now();
        
        this.connect();
    }
    
    connect() {
        this.eventSource = new EventSource(this.url);
        
        this.eventSource.onopen = () => {
            console.log('SSE connected');
            this.reconnectAttempts = 0;
            this.startHeartbeat();
        };
        
        this.eventSource.onmessage = (event) => {
            this.lastMessageTime = Date.now();
            this.handleMessage(event);
        };
        
        this.eventSource.onerror = (event) => {
            console.error('SSE error:', event);
            this.handleError();
        };
    }
    
    startHeartbeat() {
        this.heartbeatTimer = setInterval(() => {
            const timeSinceLastMessage = Date.now() - this.lastMessageTime;
            
            if (timeSinceLastMessage > this.options.heartbeatTimeout) {
                console.warn('Heartbeat timeout, reconnecting...');
                this.reconnect();
            }
        }, this.options.heartbeatTimeout);
    }
    
    stopHeartbeat() {
        if (this.heartbeatTimer) {
            clearInterval(this.heartbeatTimer);
            this.heartbeatTimer = null;
        }
    }
    
    handleError() {
        this.stopHeartbeat();
        this.reconnect();
    }
    
    reconnect() {
        if (this.reconnectAttempts < this.options.maxReconnectAttempts) {
            this.reconnectAttempts++;
            const delay = this.options.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1);
            
            console.log(`Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts})`);
            
            setTimeout(() => {
                this.eventSource.close();
                this.connect();
            }, delay);
        } else {
            console.error('Max reconnection attempts reached');
            this.onMaxReconnectAttempts?.();
        }
    }
    
    handleMessage(event) {
        // Override in subclass
    }
    
    disconnect() {
        this.stopHeartbeat();
        this.eventSource?.close();
    }
    
    onMaxReconnectAttempts = null;
}
```

## Kết luận

Server-Sent Events cung cấp một cách đơn giản và hiệu quả để nhận dữ liệu real-time từ server. Một số điểm quan trọng:

- SSE phù hợp cho one-way communication (server → client)
- Tự động reconnection và error handling
- Hỗ trợ custom event types
- Dễ dàng implement authentication
- Performance tốt hơn polling

Khi nào sử dụng SSE:
- Live notifications
- Real-time dashboards
- Live updates (news, sports scores)
- Progress tracking

Khi nào không nên dùng SSE:
- Chat applications (cần bidirectional)
- Gaming (cần low latency)
- File transfer (cần binary data)

Trong bài viết tiếp theo, chúng ta sẽ tìm hiểu về WebRTC và peer-to-peer communication.

