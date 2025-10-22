---
title: "Lập trình WebSocket với Java"
date: 2024-10-17T10:00:00+07:00
draft: false
tags: ["Java", "WebSocket", "Real-time", "Networking", "JSR-356"]
description: "Hướng dẫn lập trình WebSocket với Java, tạo ứng dụng real-time chat"
---

# Lập trình WebSocket với Java

WebSocket là một giao thức truyền thông cho phép giao tiếp hai chiều real-time giữa client và server. Khác với HTTP, WebSocket duy trì kết nối liên tục, rất phù hợp cho các ứng dụng chat, game online, và real-time collaboration.

## WebSocket vs HTTP

| Đặc điểm | HTTP | WebSocket |
|----------|------|-----------|
| Kết nối | Request-Response | Persistent |
| Overhead | Cao (headers mỗi request) | Thấp (sau handshake) |
| Real-time | Không | Có |
| Server push | Không (cần polling) | Có |

## Thiết lập WebSocket Server với Java

### 1. Dependency (Maven)

```xml
<dependency>
    <groupId>javax.websocket</groupId>
    <artifactId>javax.websocket-api</artifactId>
    <version>1.1</version>
</dependency>
```

### 2. WebSocket Endpoint

```java
import javax.websocket.*;
import javax.websocket.server.ServerEndpoint;
import java.io.IOException;
import java.util.Collections;
import java.util.HashSet;
import java.util.Set;

@ServerEndpoint("/chat")
public class ChatEndpoint {
    private static Set<Session> sessions = Collections.synchronizedSet(new HashSet<>());
    
    @OnOpen
    public void onOpen(Session session) {
        sessions.add(session);
        System.out.println("Client kết nối: " + session.getId());
        broadcast("User " + session.getId() + " đã tham gia chat");
    }
    
    @OnMessage
    public void onMessage(String message, Session session) {
        System.out.println("Nhận tin nhắn từ " + session.getId() + ": " + message);
        broadcast("User " + session.getId() + ": " + message);
    }
    
    @OnClose
    public void onClose(Session session) {
        sessions.remove(session);
        System.out.println("Client ngắt kết nối: " + session.getId());
        broadcast("User " + session.getId() + " đã rời khỏi chat");
    }
    
    @OnError
    public void onError(Session session, Throwable throwable) {
        System.err.println("Lỗi WebSocket: " + throwable.getMessage());
        sessions.remove(session);
    }
    
    private void broadcast(String message) {
        synchronized (sessions) {
            for (Session session : sessions) {
                try {
                    session.getBasicRemote().sendText(message);
                } catch (IOException e) {
                    System.err.println("Lỗi gửi tin nhắn: " + e.getMessage());
                }
            }
        }
    }
}
```

### 3. Server Configuration

```java
import javax.websocket.server.ServerContainer;
import javax.websocket.server.ServerEndpointConfig;
import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.servlet.ServletContextHandler;
import org.eclipse.jetty.websocket.jsr356.server.deploy.WebSocketServerContainer;

public class WebSocketServer {
    public static void main(String[] args) {
        Server server = new Server(8080);
        ServletContextHandler context = new ServletContextHandler();
        context.setContextPath("/");
        server.setHandler(context);
        
        try {
            server.start();
            ServerContainer wsContainer = WebSocketServerContainer.initialize(context);
            wsContainer.addEndpoint(ChatEndpoint.class);
            
            System.out.println("WebSocket Server đang chạy trên port 8080");
            System.out.println("Kết nối đến: ws://localhost:8080/chat");
            
            server.join();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## WebSocket Client với JavaScript

```html
<!DOCTYPE html>
<html>
<head>
    <title>WebSocket Chat</title>
    <style>
        #messages { 
            height: 300px; 
            overflow-y: scroll; 
            border: 1px solid #ccc; 
            padding: 10px; 
        }
        #messageInput { 
            width: 70%; 
            padding: 5px; 
        }
        #sendButton { 
            width: 25%; 
            padding: 5px; 
        }
    </style>
</head>
<body>
    <div id="messages"></div>
    <input type="text" id="messageInput" placeholder="Nhập tin nhắn...">
    <button id="sendButton">Gửi</button>
    
    <script>
        const messages = document.getElementById('messages');
        const messageInput = document.getElementById('messageInput');
        const sendButton = document.getElementById('sendButton');
        
        // Kết nối WebSocket
        const socket = new WebSocket('ws://localhost:8080/chat');
        
        socket.onopen = function(event) {
            addMessage('Đã kết nối đến server');
        };
        
        socket.onmessage = function(event) {
            addMessage(event.data);
        };
        
        socket.onclose = function(event) {
            addMessage('Đã ngắt kết nối');
        };
        
        socket.onerror = function(error) {
            addMessage('Lỗi: ' + error);
        };
        
        function addMessage(message) {
            const div = document.createElement('div');
            div.textContent = new Date().toLocaleTimeString() + ': ' + message;
            messages.appendChild(div);
            messages.scrollTop = messages.scrollHeight;
        }
        
        function sendMessage() {
            const message = messageInput.value.trim();
            if (message) {
                socket.send(message);
                messageInput.value = '';
            }
        }
        
        sendButton.onclick = sendMessage;
        messageInput.onkeypress = function(e) {
            if (e.key === 'Enter') {
                sendMessage();
            }
        };
    </script>
</body>
</html>
```

## Xử lý Binary Data

```java
@OnMessage
public void onBinaryMessage(byte[] data, Session session) {
    System.out.println("Nhận binary data: " + data.length + " bytes");
    
    // Xử lý file upload
    try {
        FileOutputStream fos = new FileOutputStream("uploaded_file_" + System.currentTimeMillis());
        fos.write(data);
        fos.close();
        
        session.getBasicRemote().sendText("File đã được upload thành công");
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

## Authentication và Authorization

```java
@ServerEndpoint(value = "/secure-chat", 
                configurator = CustomConfigurator.class)
public class SecureChatEndpoint {
    
    @OnOpen
    public void onOpen(Session session, EndpointConfig config) {
        String token = (String) config.getUserProperties().get("token");
        
        if (isValidToken(token)) {
            // Cho phép kết nối
            sessions.add(session);
        } else {
            try {
                session.close(new CloseReason(CloseReason.CloseCodes.CANNOT_ACCEPT, "Invalid token"));
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    
    private boolean isValidToken(String token) {
        // Logic xác thực token
        return token != null && token.startsWith("valid_");
    }
}
```

## Custom Configurator

```java
import javax.websocket.HandshakeResponse;
import javax.websocket.server.HandshakeRequest;
import javax.websocket.server.ServerEndpointConfig;

public class CustomConfigurator extends ServerEndpointConfig.Configurator {
    @Override
    public void modifyHandshake(ServerEndpointConfig config, 
                               HandshakeRequest request, 
                               HandshakeResponse response) {
        // Lấy token từ query parameter
        String token = request.getQueryString();
        if (token != null) {
            config.getUserProperties().put("token", token);
        }
    }
}
```

## Heartbeat và Ping/Pong

```java
@OnMessage
public void onPongMessage(PongMessage pong, Session session) {
    System.out.println("Nhận pong từ " + session.getId());
    // Cập nhật thời gian ping cuối cùng
    lastPingTime.put(session.getId(), System.currentTimeMillis());
}

// Gửi ping định kỳ
private void sendPing(Session session) {
    try {
        session.getBasicRemote().sendPing(ByteBuffer.wrap("ping".getBytes()));
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

## Error Handling và Reconnection

```javascript
class WebSocketClient {
    constructor(url) {
        this.url = url;
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 5;
        this.reconnectDelay = 1000;
        this.connect();
    }
    
    connect() {
        this.socket = new WebSocket(this.url);
        
        this.socket.onopen = () => {
            console.log('Kết nối thành công');
            this.reconnectAttempts = 0;
        };
        
        this.socket.onclose = () => {
            console.log('Kết nối bị đóng');
            this.reconnect();
        };
        
        this.socket.onerror = (error) => {
            console.error('Lỗi WebSocket:', error);
        };
    }
    
    reconnect() {
        if (this.reconnectAttempts < this.maxReconnectAttempts) {
            this.reconnectAttempts++;
            console.log(`Thử kết nối lại lần ${this.reconnectAttempts}`);
            
            setTimeout(() => {
                this.connect();
            }, this.reconnectDelay * this.reconnectAttempts);
        }
    }
}
```

## Performance và Scaling

### 1. Connection Pooling
```java
private static final int MAX_CONNECTIONS = 1000;
private static final Semaphore connectionSemaphore = new Semaphore(MAX_CONNECTIONS);

@OnOpen
public void onOpen(Session session) {
    if (connectionSemaphore.tryAcquire()) {
        sessions.add(session);
    } else {
        try {
            session.close(new CloseReason(CloseReason.CloseCodes.TRY_AGAIN_LATER, "Server quá tải"));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 2. Message Queuing
```java
private static final BlockingQueue<String> messageQueue = new LinkedBlockingQueue<>();

// Producer
@OnMessage
public void onMessage(String message, Session session) {
    messageQueue.offer(message);
}

// Consumer
private void processMessages() {
    while (true) {
        try {
            String message = messageQueue.take();
            broadcast(message);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            break;
        }
    }
}
```

## Kết luận

WebSocket cung cấp khả năng giao tiếp real-time mạnh mẽ cho các ứng dụng Java. Khi sử dụng WebSocket, cần chú ý:

- Xử lý lỗi và reconnection
- Authentication và authorization
- Performance và scaling
- Message queuing cho high-throughput applications

Trong bài viết tiếp theo, chúng ta sẽ tìm hiểu về RESTful APIs với Spring Boot và cách tích hợp với WebSocket.

