---
title: "Lập trình Socket với Java - Cơ bản"
date: 2024-10-15T10:00:00+07:00
draft: false
tags: ["Java", "Socket", "Networking", "Cơ bản"]
description: "Tìm hiểu cơ bản về lập trình socket với Java, cách tạo client-server đơn giản"
---

# Lập trình Socket với Java - Cơ bản

Lập trình socket là một trong những kỹ năng cơ bản nhất trong lập trình mạng. Trong bài viết này, chúng ta sẽ tìm hiểu cách sử dụng Java để tạo các ứng dụng client-server đơn giản.

## Socket là gì?

Socket là một endpoint để giao tiếp giữa hai chương trình chạy trên mạng. Nó cung cấp một cơ chế để các tiến trình có thể giao tiếp với nhau thông qua mạng.

## Các loại Socket trong Java

### 1. TCP Socket (Socket)
- Kết nối đáng tin cậy
- Đảm bảo dữ liệu được truyền đúng thứ tự
- Sử dụng cho các ứng dụng cần độ tin cậy cao

### 2. UDP Socket (DatagramSocket)
- Kết nối không đáng tin cậy
- Tốc độ nhanh hơn
- Sử dụng cho các ứng dụng real-time

## Ví dụ Server đơn giản

```java
import java.io.*;
import java.net.*;

public class SimpleServer {
    public static void main(String[] args) {
        try {
            // Tạo server socket trên port 8080
            ServerSocket serverSocket = new ServerSocket(8080);
            System.out.println("Server đang chạy trên port 8080...");
            
            while (true) {
                // Chờ client kết nối
                Socket clientSocket = serverSocket.accept();
                System.out.println("Client đã kết nối: " + clientSocket.getInetAddress());
                
                // Xử lý client trong thread riêng
                new Thread(() -> handleClient(clientSocket)).start();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    
    private static void handleClient(Socket clientSocket) {
        try {
            BufferedReader in = new BufferedReader(
                new InputStreamReader(clientSocket.getInputStream())
            );
            PrintWriter out = new PrintWriter(
                clientSocket.getOutputStream(), true
            );
            
            String inputLine;
            while ((inputLine = in.readLine()) != null) {
                System.out.println("Nhận từ client: " + inputLine);
                out.println("Server phản hồi: " + inputLine);
            }
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
}
```

## Ví dụ Client đơn giản

```java
import java.io.*;
import java.net.*;

public class SimpleClient {
    public static void main(String[] args) {
        try {
            // Kết nối đến server
            Socket socket = new Socket("localhost", 8080);
            
            BufferedReader in = new BufferedReader(
                new InputStreamReader(socket.getInputStream())
            );
            PrintWriter out = new PrintWriter(
                socket.getOutputStream(), true
            );
            
            // Gửi dữ liệu
            out.println("Xin chào từ client!");
            
            // Nhận phản hồi
            String response = in.readLine();
            System.out.println("Phản hồi từ server: " + response);
            
            socket.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## Xử lý lỗi và Best Practices

### 1. Luôn đóng resources
```java
try (Socket socket = new Socket("localhost", 8080);
     BufferedReader in = new BufferedReader(
         new InputStreamReader(socket.getInputStream())
     );
     PrintWriter out = new PrintWriter(
         socket.getOutputStream(), true
     )) {
    // Xử lý dữ liệu
} catch (IOException e) {
    e.printStackTrace();
}
```

### 2. Sử dụng timeout
```java
socket.setSoTimeout(5000); // 5 giây timeout
```

### 3. Xử lý nhiều client với Thread Pool
```java
ExecutorService executor = Executors.newFixedThreadPool(10);
executor.submit(() -> handleClient(clientSocket));
```

## Kết luận

Lập trình socket với Java cung cấp nền tảng mạnh mẽ để xây dựng các ứng dụng mạng. Mặc dù có nhiều framework cao cấp hơn như Spring Boot, việc hiểu rõ socket programming sẽ giúp bạn debug và tối ưu hóa ứng dụng tốt hơn.

Trong bài viết tiếp theo, chúng ta sẽ tìm hiểu về HTTP client/server với Java và cách xây dựng RESTful APIs.

