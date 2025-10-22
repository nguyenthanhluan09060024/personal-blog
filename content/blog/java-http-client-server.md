---
title: "Xây dựng HTTP Client/Server với Java"
date: 2024-10-16T10:00:00+07:00
draft: false
tags: ["Java", "HTTP", "Server", "Client", "Networking"]
description: "Hướng dẫn xây dựng HTTP server và client từ đầu với Java, không sử dụng framework"
---

# Xây dựng HTTP Client/Server với Java

HTTP (HyperText Transfer Protocol) là giao thức cơ bản của web. Trong bài viết này, chúng ta sẽ học cách xây dựng một HTTP server và client đơn giản từ đầu bằng Java.

## HTTP Protocol cơ bản

HTTP hoạt động theo mô hình request-response:
- **Request**: Client gửi yêu cầu đến server
- **Response**: Server phản hồi với dữ liệu được yêu cầu

### Cấu trúc HTTP Request
```
GET /path HTTP/1.1
Host: localhost:8080
User-Agent: Mozilla/5.0
Accept: text/html

[Body nếu có]
```

### Cấu trúc HTTP Response
```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1234

<html>...</html>
```

## HTTP Server đơn giản

```java
import java.io.*;
import java.net.*;
import java.util.HashMap;
import java.util.Map;

public class SimpleHttpServer {
    private static final int PORT = 8080;
    
    public static void main(String[] args) {
        try {
            ServerSocket serverSocket = new ServerSocket(PORT);
            System.out.println("HTTP Server đang chạy trên port " + PORT);
            
            while (true) {
                Socket clientSocket = serverSocket.accept();
                new Thread(() -> handleRequest(clientSocket)).start();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    
    private static void handleRequest(Socket clientSocket) {
        try (BufferedReader in = new BufferedReader(
                new InputStreamReader(clientSocket.getInputStream()));
             PrintWriter out = new PrintWriter(
                clientSocket.getOutputStream(), true)) {
            
            // Đọc HTTP request
            String requestLine = in.readLine();
            if (requestLine == null) return;
            
            System.out.println("Request: " + requestLine);
            
            // Parse request
            String[] requestParts = requestLine.split(" ");
            String method = requestParts[0];
            String path = requestParts[1];
            
            // Đọc headers
            Map<String, String> headers = new HashMap<>();
            String line;
            while ((line = in.readLine()) != null && !line.isEmpty()) {
                String[] headerParts = line.split(": ", 2);
                if (headerParts.length == 2) {
                    headers.put(headerParts[0], headerParts[1]);
                }
            }
            
            // Xử lý request
            String response = handleHttpRequest(method, path, headers);
            
            // Gửi response
            out.print(response);
            out.flush();
            
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                clientSocket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    
    private static String handleHttpRequest(String method, String path, 
                                         Map<String, String> headers) {
        StringBuilder response = new StringBuilder();
        
        if ("GET".equals(method)) {
            if ("/".equals(path)) {
                response.append("HTTP/1.1 200 OK\r\n");
                response.append("Content-Type: text/html\r\n");
                response.append("\r\n");
                response.append("<html><body>");
                response.append("<h1>Chào mừng đến với HTTP Server!</h1>");
                response.append("<p>Server được xây dựng bằng Java thuần</p>");
                response.append("</body></html>");
            } else if ("/api/hello".equals(path)) {
                response.append("HTTP/1.1 200 OK\r\n");
                response.append("Content-Type: application/json\r\n");
                response.append("\r\n");
                response.append("{\"message\": \"Hello from Java HTTP Server!\"}");
            } else {
                response.append("HTTP/1.1 404 Not Found\r\n");
                response.append("Content-Type: text/html\r\n");
                response.append("\r\n");
                response.append("<html><body><h1>404 - Not Found</h1></body></html>");
            }
        } else {
            response.append("HTTP/1.1 405 Method Not Allowed\r\n");
            response.append("Content-Type: text/html\r\n");
            response.append("\r\n");
            response.append("<html><body><h1>405 - Method Not Allowed</h1></body></html>");
        }
        
        return response.toString();
    }
}
```

## HTTP Client đơn giản

```java
import java.io.*;
import java.net.*;

public class SimpleHttpClient {
    public static void main(String[] args) {
        try {
            // Kết nối đến server
            Socket socket = new Socket("localhost", 8080);
            
            PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
            BufferedReader in = new BufferedReader(
                new InputStreamReader(socket.getInputStream())
            );
            
            // Gửi HTTP GET request
            out.println("GET / HTTP/1.1");
            out.println("Host: localhost:8080");
            out.println("User-Agent: Java-HTTP-Client");
            out.println("Accept: text/html");
            out.println(); // Empty line để kết thúc headers
            
            // Đọc response
            String line;
            while ((line = in.readLine()) != null) {
                System.out.println(line);
            }
            
            socket.close();
            
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## Cải tiến với URLConnection

Java cung cấp `URLConnection` để làm việc với HTTP dễ dàng hơn:

```java
import java.io.*;
import java.net.*;

public class ImprovedHttpClient {
    public static void main(String[] args) {
        try {
            URL url = new URL("http://localhost:8080/api/hello");
            HttpURLConnection connection = (HttpURLConnection) url.openConnection();
            
            // Cấu hình request
            connection.setRequestMethod("GET");
            connection.setRequestProperty("Accept", "application/json");
            connection.setConnectTimeout(5000);
            connection.setReadTimeout(5000);
            
            // Đọc response
            int responseCode = connection.getResponseCode();
            System.out.println("Response Code: " + responseCode);
            
            BufferedReader reader = new BufferedReader(
                new InputStreamReader(connection.getInputStream())
            );
            
            String line;
            StringBuilder response = new StringBuilder();
            while ((line = reader.readLine()) != null) {
                response.append(line);
            }
            reader.close();
            
            System.out.println("Response: " + response.toString());
            
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## Xử lý POST Request

```java
private static String handlePostRequest(String path, Map<String, String> headers, 
                                      String body) {
    StringBuilder response = new StringBuilder();
    
    if ("/api/users".equals(path)) {
        // Parse JSON body (đơn giản)
        System.out.println("POST body: " + body);
        
        response.append("HTTP/1.1 201 Created\r\n");
        response.append("Content-Type: application/json\r\n");
        response.append("\r\n");
        response.append("{\"id\": 1, \"message\": \"User created successfully\"}");
    } else {
        response.append("HTTP/1.1 404 Not Found\r\n");
        response.append("Content-Type: text/html\r\n");
        response.append("\r\n");
        response.append("<html><body><h1>404 - Not Found</h1></body></html>");
    }
    
    return response.toString();
}
```

## Best Practices

### 1. Sử dụng try-with-resources
```java
try (Socket socket = new Socket("localhost", 8080);
     PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
     BufferedReader in = new BufferedReader(
         new InputStreamReader(socket.getInputStream())
     )) {
    // Xử lý request/response
}
```

### 2. Xử lý lỗi đúng cách
```java
try {
    // HTTP operations
} catch (ConnectException e) {
    System.err.println("Không thể kết nối đến server");
} catch (SocketTimeoutException e) {
    System.err.println("Timeout khi kết nối");
} catch (IOException e) {
    System.err.println("Lỗi I/O: " + e.getMessage());
}
```

### 3. Sử dụng Thread Pool
```java
ExecutorService executor = Executors.newFixedThreadPool(20);
executor.submit(() -> handleRequest(clientSocket));
```

## Kết luận

Việc hiểu cách HTTP hoạt động ở mức thấp sẽ giúp bạn:
- Debug các vấn đề mạng tốt hơn
- Tối ưu hóa performance
- Hiểu rõ hơn về các framework như Spring Boot

Trong bài viết tiếp theo, chúng ta sẽ tìm hiểu về WebSocket và real-time communication với Java.

