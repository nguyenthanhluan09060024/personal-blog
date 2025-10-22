---
title: "JavaScript Fetch API - Làm việc với HTTP"
date: 2024-10-18T10:00:00+07:00
draft: false
tags: ["JavaScript", "Fetch", "HTTP", "API", "Async"]
description: "Hướng dẫn sử dụng Fetch API trong JavaScript để giao tiếp với server"
---

# JavaScript Fetch API - Làm việc với HTTP

Fetch API là một interface hiện đại trong JavaScript để thực hiện HTTP requests. Nó thay thế XMLHttpRequest với syntax đơn giản hơn và hỗ trợ Promises.

## Fetch API cơ bản

### GET Request
```javascript
// GET request đơn giản
fetch('https://api.example.com/users')
    .then(response => response.json())
    .then(data => console.log(data))
    .catch(error => console.error('Lỗi:', error));

// Với async/await
async function getUsers() {
    try {
        const response = await fetch('https://api.example.com/users');
        const data = await response.json();
        console.log(data);
        return data;
    } catch (error) {
        console.error('Lỗi:', error);
    }
}
```

### POST Request
```javascript
// POST request với JSON data
async function createUser(userData) {
    try {
        const response = await fetch('https://api.example.com/users', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify(userData)
        });
        
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const newUser = await response.json();
        console.log('User đã được tạo:', newUser);
        return newUser;
    } catch (error) {
        console.error('Lỗi tạo user:', error);
    }
}

// Sử dụng
createUser({
    name: 'Nguyễn Văn A',
    email: 'nguyenvana@example.com'
});
```

## Xử lý Response

### Kiểm tra Status Code
```javascript
async function fetchUser(id) {
    const response = await fetch(`https://api.example.com/users/${id}`);
    
    if (response.status === 404) {
        throw new Error('User không tồn tại');
    }
    
    if (response.status === 500) {
        throw new Error('Lỗi server');
    }
    
    if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
    }
    
    return await response.json();
}
```

### Xử lý các loại Response khác nhau
```javascript
async function fetchData(url) {
    const response = await fetch(url);
    
    const contentType = response.headers.get('content-type');
    
    if (contentType.includes('application/json')) {
        return await response.json();
    } else if (contentType.includes('text/')) {
        return await response.text();
    } else if (contentType.includes('image/')) {
        return await response.blob();
    } else {
        return await response.arrayBuffer();
    }
}
```

## Headers và Authentication

### Custom Headers
```javascript
async function fetchWithHeaders() {
    const response = await fetch('https://api.example.com/data', {
        headers: {
            'Authorization': 'Bearer your-token-here',
            'X-Custom-Header': 'custom-value',
            'Accept': 'application/json',
            'Content-Type': 'application/json'
        }
    });
    
    return await response.json();
}
```

### Authentication với JWT
```javascript
class ApiClient {
    constructor(baseURL, token) {
        this.baseURL = baseURL;
        this.token = token;
    }
    
    async request(endpoint, options = {}) {
        const url = `${this.baseURL}${endpoint}`;
        
        const config = {
            ...options,
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${this.token}`,
                ...options.headers
            }
        };
        
        const response = await fetch(url, config);
        
        if (!response.ok) {
            throw new Error(`API Error: ${response.status} ${response.statusText}`);
        }
        
        return await response.json();
    }
    
    async get(endpoint) {
        return this.request(endpoint);
    }
    
    async post(endpoint, data) {
        return this.request(endpoint, {
            method: 'POST',
            body: JSON.stringify(data)
        });
    }
    
    async put(endpoint, data) {
        return this.request(endpoint, {
            method: 'PUT',
            body: JSON.stringify(data)
        });
    }
    
    async delete(endpoint) {
        return this.request(endpoint, {
            method: 'DELETE'
        });
    }
}

// Sử dụng
const api = new ApiClient('https://api.example.com', 'your-jwt-token');
const users = await api.get('/users');
```

## Upload Files

### Upload File đơn giản
```javascript
async function uploadFile(file) {
    const formData = new FormData();
    formData.append('file', file);
    
    try {
        const response = await fetch('/api/upload', {
            method: 'POST',
            body: formData
        });
        
        if (!response.ok) {
            throw new Error('Upload failed');
        }
        
        const result = await response.json();
        console.log('Upload thành công:', result);
        return result;
    } catch (error) {
        console.error('Lỗi upload:', error);
    }
}

// Sử dụng với file input
document.getElementById('fileInput').addEventListener('change', async (event) => {
    const file = event.target.files[0];
    if (file) {
        await uploadFile(file);
    }
});
```

### Upload với Progress
```javascript
async function uploadWithProgress(file, onProgress) {
    const formData = new FormData();
    formData.append('file', file);
    
    return new Promise((resolve, reject) => {
        const xhr = new XMLHttpRequest();
        
        xhr.upload.addEventListener('progress', (event) => {
            if (event.lengthComputable) {
                const percentComplete = (event.loaded / event.total) * 100;
                onProgress(percentComplete);
            }
        });
        
        xhr.addEventListener('load', () => {
            if (xhr.status === 200) {
                resolve(JSON.parse(xhr.responseText));
            } else {
                reject(new Error('Upload failed'));
            }
        });
        
        xhr.addEventListener('error', () => {
            reject(new Error('Upload failed'));
        });
        
        xhr.open('POST', '/api/upload');
        xhr.send(formData);
    });
}
```

## Error Handling nâng cao

### Retry Logic
```javascript
async function fetchWithRetry(url, options = {}, maxRetries = 3) {
    let lastError;
    
    for (let i = 0; i < maxRetries; i++) {
        try {
            const response = await fetch(url, options);
            
            if (!response.ok) {
                throw new Error(`HTTP ${response.status}: ${response.statusText}`);
            }
            
            return await response.json();
        } catch (error) {
            lastError = error;
            console.warn(`Lần thử ${i + 1} thất bại:`, error.message);
            
            if (i < maxRetries - 1) {
                // Exponential backoff
                const delay = Math.pow(2, i) * 1000;
                await new Promise(resolve => setTimeout(resolve, delay));
            }
        }
    }
    
    throw lastError;
}
```

### Timeout
```javascript
function fetchWithTimeout(url, options = {}, timeout = 5000) {
    return Promise.race([
        fetch(url, options),
        new Promise((_, reject) => 
            setTimeout(() => reject(new Error('Request timeout')), timeout)
        )
    ]);
}
```

## Request/Response Interceptors

```javascript
class FetchInterceptor {
    constructor() {
        this.requestInterceptors = [];
        this.responseInterceptors = [];
    }
    
    addRequestInterceptor(interceptor) {
        this.requestInterceptors.push(interceptor);
    }
    
    addResponseInterceptor(interceptor) {
        this.responseInterceptors.push(interceptor);
    }
    
    async fetch(url, options = {}) {
        // Apply request interceptors
        let modifiedOptions = { ...options };
        for (const interceptor of this.requestInterceptors) {
            modifiedOptions = await interceptor(url, modifiedOptions);
        }
        
        // Make request
        let response = await fetch(url, modifiedOptions);
        
        // Apply response interceptors
        for (const interceptor of this.responseInterceptors) {
            response = await interceptor(response);
        }
        
        return response;
    }
}

// Sử dụng
const interceptor = new FetchInterceptor();

// Request interceptor để thêm token
interceptor.addRequestInterceptor(async (url, options) => {
    const token = localStorage.getItem('token');
    if (token) {
        options.headers = {
            ...options.headers,
            'Authorization': `Bearer ${token}`
        };
    }
    return options;
});

// Response interceptor để xử lý lỗi
interceptor.addResponseInterceptor(async (response) => {
    if (response.status === 401) {
        // Token hết hạn, redirect đến login
        localStorage.removeItem('token');
        window.location.href = '/login';
    }
    return response;
});

// Sử dụng interceptor
const data = await interceptor.fetch('/api/data');
```

## Caching và Performance

### Simple Cache
```javascript
class ApiCache {
    constructor(ttl = 5 * 60 * 1000) { // 5 minutes
        this.cache = new Map();
        this.ttl = ttl;
    }
    
    get(key) {
        const item = this.cache.get(key);
        if (!item) return null;
        
        if (Date.now() - item.timestamp > this.ttl) {
            this.cache.delete(key);
            return null;
        }
        
        return item.data;
    }
    
    set(key, data) {
        this.cache.set(key, {
            data,
            timestamp: Date.now()
        });
    }
    
    clear() {
        this.cache.clear();
    }
}

const cache = new ApiCache();

async function fetchCached(url) {
    const cached = cache.get(url);
    if (cached) {
        return cached;
    }
    
    const response = await fetch(url);
    const data = await response.json();
    
    cache.set(url, data);
    return data;
}
```

## Kết luận

Fetch API cung cấp một cách hiện đại và linh hoạt để thực hiện HTTP requests trong JavaScript. Một số điểm quan trọng:

- Sử dụng async/await để code dễ đọc hơn
- Luôn xử lý lỗi đúng cách
- Sử dụng interceptors để tái sử dụng logic
- Implement caching để cải thiện performance
- Sử dụng retry logic cho các request quan trọng

Trong bài viết tiếp theo, chúng ta sẽ tìm hiểu về WebSocket với JavaScript và cách xây dựng ứng dụng real-time.

