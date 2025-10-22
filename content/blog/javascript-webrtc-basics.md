---
title: "WebRTC cơ bản với JavaScript"
date: 2024-10-21T10:00:00+07:00
draft: false
tags: ["JavaScript", "WebRTC", "P2P", "Video", "Audio", "Networking"]
description: "Hướng dẫn sử dụng WebRTC để tạo ứng dụng video call và peer-to-peer communication"
---

# WebRTC cơ bản với JavaScript

WebRTC (Web Real-Time Communication) là một công nghệ cho phép giao tiếp real-time trực tiếp giữa các trình duyệt mà không cần server trung gian. Nó hỗ trợ video, audio và data transfer.

## WebRTC Architecture

```
Browser A ←→ STUN/TURN Server ←→ Browser B
    ↓
Media Stream (Camera/Microphone)
    ↓
PeerConnection
    ↓
Data Channel
```

## Các thành phần chính

1. **getUserMedia()**: Truy cập camera và microphone
2. **RTCPeerConnection**: Quản lý kết nối P2P
3. **RTCDataChannel**: Truyền dữ liệu tùy ý
4. **STUN/TURN Servers**: Giúp NAT traversal

## Video Call đơn giản

```html
<!DOCTYPE html>
<html>
<head>
    <title>WebRTC Video Call</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .video-container { 
            display: flex; 
            gap: 20px; 
            margin-bottom: 20px; 
        }
        video { 
            width: 300px; 
            height: 200px; 
            border: 2px solid #ccc; 
            border-radius: 8px;
        }
        .controls { 
            margin: 20px 0; 
        }
        button { 
            padding: 10px 20px; 
            margin: 5px; 
            border: none; 
            border-radius: 5px; 
            cursor: pointer;
        }
        .start { background-color: #28a745; color: white; }
        .call { background-color: #007bff; color: white; }
        .hangup { background-color: #dc3545; color: white; }
        .mute { background-color: #ffc107; color: black; }
        .status { 
            padding: 10px; 
            margin: 10px 0; 
            border-radius: 5px; 
            background-color: #f8f9fa;
        }
    </style>
</head>
<body>
    <h1>WebRTC Video Call</h1>
    
    <div class="video-container">
        <div>
            <h3>Local Video</h3>
            <video id="localVideo" autoplay muted></video>
        </div>
        <div>
            <h3>Remote Video</h3>
            <video id="remoteVideo" autoplay></video>
        </div>
    </div>
    
    <div class="controls">
        <button id="startButton" class="start">Start Camera</button>
        <button id="callButton" class="call" disabled>Call</button>
        <button id="hangupButton" class="hangup" disabled>Hang Up</button>
        <button id="muteButton" class="mute" disabled>Mute</button>
    </div>
    
    <div id="status" class="status">Click "Start Camera" to begin</div>
    
    <script>
        class VideoCall {
            constructor() {
                this.localStream = null;
                this.peerConnection = null;
                this.isMuted = false;
                
                this.localVideo = document.getElementById('localVideo');
                this.remoteVideo = document.getElementById('remoteVideo');
                this.startButton = document.getElementById('startButton');
                this.callButton = document.getElementById('callButton');
                this.hangupButton = document.getElementById('hangupButton');
                this.muteButton = document.getElementById('muteButton');
                this.status = document.getElementById('status');
                
                this.setupEventListeners();
            }
            
            setupEventListeners() {
                this.startButton.onclick = () => this.startCamera();
                this.callButton.onclick = () => this.startCall();
                this.hangupButton.onclick = () => this.hangUp();
                this.muteButton.onclick = () => this.toggleMute();
            }
            
            async startCamera() {
                try {
                    this.localStream = await navigator.mediaDevices.getUserMedia({
                        video: true,
                        audio: true
                    });
                    
                    this.localVideo.srcObject = this.localStream;
                    this.updateStatus('Camera started. Click "Call" to connect.');
                    
                    this.startButton.disabled = true;
                    this.callButton.disabled = false;
                    this.muteButton.disabled = false;
                    
                } catch (error) {
                    console.error('Error accessing camera:', error);
                    this.updateStatus('Error accessing camera: ' + error.message);
                }
            }
            
            async startCall() {
                try {
                    this.peerConnection = new RTCPeerConnection({
                        iceServers: [
                            { urls: 'stun:stun.l.google.com:19302' },
                            { urls: 'stun:stun1.l.google.com:19302' }
                        ]
                    });
                    
                    this.setupPeerConnection();
                    
                    // Add local stream
                    this.localStream.getTracks().forEach(track => {
                        this.peerConnection.addTrack(track, this.localStream);
                    });
                    
                    // Create offer
                    const offer = await this.peerConnection.createOffer();
                    await this.peerConnection.setLocalDescription(offer);
                    
                    this.updateStatus('Calling...');
                    this.callButton.disabled = true;
                    this.hangupButton.disabled = false;
                    
                    // In a real app, you would send the offer to the remote peer
                    // through a signaling server (WebSocket, Socket.IO, etc.)
                    console.log('Offer created:', offer);
                    
                } catch (error) {
                    console.error('Error starting call:', error);
                    this.updateStatus('Error starting call: ' + error.message);
                }
            }
            
            setupPeerConnection() {
                this.peerConnection.onicecandidate = (event) => {
                    if (event.candidate) {
                        console.log('ICE candidate:', event.candidate);
                        // Send candidate to remote peer
                    }
                };
                
                this.peerConnection.ontrack = (event) => {
                    console.log('Remote stream received');
                    this.remoteVideo.srcObject = event.streams[0];
                    this.updateStatus('Connected!');
                };
                
                this.peerConnection.onconnectionstatechange = () => {
                    console.log('Connection state:', this.peerConnection.connectionState);
                    
                    switch (this.peerConnection.connectionState) {
                        case 'connected':
                            this.updateStatus('Connected!');
                            break;
                        case 'disconnected':
                        case 'failed':
                            this.updateStatus('Connection lost');
                            this.resetCall();
                            break;
                    }
                };
            }
            
            toggleMute() {
                if (this.localStream) {
                    const audioTracks = this.localStream.getAudioTracks();
                    audioTracks.forEach(track => {
                        track.enabled = !track.enabled;
                    });
                    
                    this.isMuted = !this.isMuted;
                    this.muteButton.textContent = this.isMuted ? 'Unmute' : 'Mute';
                    this.updateStatus(this.isMuted ? 'Muted' : 'Unmuted');
                }
            }
            
            hangUp() {
                if (this.peerConnection) {
                    this.peerConnection.close();
                    this.peerConnection = null;
                }
                
                if (this.remoteVideo.srcObject) {
                    this.remoteVideo.srcObject = null;
                }
                
                this.resetCall();
                this.updateStatus('Call ended');
            }
            
            resetCall() {
                this.callButton.disabled = false;
                this.hangupButton.disabled = true;
            }
            
            updateStatus(message) {
                this.status.textContent = message;
                console.log('Status:', message);
            }
        }
        
        // Khởi tạo video call
        const videoCall = new VideoCall();
    </script>
</body>
</html>
```

## Signaling Server với Socket.IO

```javascript
// server.js
const express = require('express');
const http = require('http');
const socketIo = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = socketIo(server);

const rooms = new Map();

io.on('connection', (socket) => {
    console.log('User connected:', socket.id);
    
    socket.on('join-room', (roomId) => {
        socket.join(roomId);
        
        if (!rooms.has(roomId)) {
            rooms.set(roomId, new Set());
        }
        rooms.get(roomId).add(socket.id);
        
        socket.to(roomId).emit('user-joined', socket.id);
        
        // Send existing users to new user
        const existingUsers = Array.from(rooms.get(roomId))
            .filter(id => id !== socket.id);
        socket.emit('existing-users', existingUsers);
    });
    
    socket.on('offer', (data) => {
        socket.to(data.roomId).emit('offer', {
            offer: data.offer,
            from: socket.id
        });
    });
    
    socket.on('answer', (data) => {
        socket.to(data.roomId).emit('answer', {
            answer: data.answer,
            from: socket.id
        });
    });
    
    socket.on('ice-candidate', (data) => {
        socket.to(data.roomId).emit('ice-candidate', {
            candidate: data.candidate,
            from: socket.id
        });
    });
    
    socket.on('disconnect', () => {
        console.log('User disconnected:', socket.id);
        
        // Remove from all rooms
        for (const [roomId, users] of rooms.entries()) {
            if (users.has(socket.id)) {
                users.delete(socket.id);
                socket.to(roomId).emit('user-left', socket.id);
                
                if (users.size === 0) {
                    rooms.delete(roomId);
                }
                break;
            }
        }
    });
});

server.listen(3000, () => {
    console.log('Signaling server running on port 3000');
});
```

## Client với Signaling

```javascript
class WebRTCClient {
    constructor() {
        this.socket = io('http://localhost:3000');
        this.peerConnections = new Map();
        this.localStream = null;
        this.roomId = null;
        
        this.setupSocketListeners();
    }
    
    setupSocketListeners() {
        this.socket.on('user-joined', (userId) => {
            console.log('User joined:', userId);
            this.createPeerConnection(userId);
        });
        
        this.socket.on('existing-users', (userIds) => {
            console.log('Existing users:', userIds);
            userIds.forEach(userId => {
                this.createPeerConnection(userId);
            });
        });
        
        this.socket.on('offer', async (data) => {
            await this.handleOffer(data.offer, data.from);
        });
        
        this.socket.on('answer', async (data) => {
            await this.handleAnswer(data.answer, data.from);
        });
        
        this.socket.on('ice-candidate', async (data) => {
            await this.handleIceCandidate(data.candidate, data.from);
        });
        
        this.socket.on('user-left', (userId) => {
            console.log('User left:', userId);
            this.removePeerConnection(userId);
        });
    }
    
    async joinRoom(roomId) {
        this.roomId = roomId;
        this.socket.emit('join-room', roomId);
        
        // Start camera
        this.localStream = await navigator.mediaDevices.getUserMedia({
            video: true,
            audio: true
        });
        
        document.getElementById('localVideo').srcObject = this.localStream;
    }
    
    async createPeerConnection(userId) {
        const peerConnection = new RTCPeerConnection({
            iceServers: [
                { urls: 'stun:stun.l.google.com:19302' }
            ]
        });
        
        // Add local stream
        if (this.localStream) {
            this.localStream.getTracks().forEach(track => {
                peerConnection.addTrack(track, this.localStream);
            });
        }
        
        peerConnection.onicecandidate = (event) => {
            if (event.candidate) {
                this.socket.emit('ice-candidate', {
                    candidate: event.candidate,
                    roomId: this.roomId,
                    to: userId
                });
            }
        };
        
        peerConnection.ontrack = (event) => {
            console.log('Remote stream received from:', userId);
            // Display remote video
            this.displayRemoteVideo(event.streams[0], userId);
        };
        
        this.peerConnections.set(userId, peerConnection);
        
        // Create offer
        const offer = await peerConnection.createOffer();
        await peerConnection.setLocalDescription(offer);
        
        this.socket.emit('offer', {
            offer: offer,
            roomId: this.roomId,
            to: userId
        });
    }
    
    async handleOffer(offer, from) {
        const peerConnection = this.peerConnections.get(from);
        if (peerConnection) {
            await peerConnection.setRemoteDescription(offer);
            
            const answer = await peerConnection.createAnswer();
            await peerConnection.setLocalDescription(answer);
            
            this.socket.emit('answer', {
                answer: answer,
                roomId: this.roomId,
                to: from
            });
        }
    }
    
    async handleAnswer(answer, from) {
        const peerConnection = this.peerConnections.get(from);
        if (peerConnection) {
            await peerConnection.setRemoteDescription(answer);
        }
    }
    
    async handleIceCandidate(candidate, from) {
        const peerConnection = this.peerConnections.get(from);
        if (peerConnection) {
            await peerConnection.addIceCandidate(candidate);
        }
    }
    
    displayRemoteVideo(stream, userId) {
        // Create video element for remote user
        const video = document.createElement('video');
        video.srcObject = stream;
        video.autoplay = true;
        video.id = `remoteVideo_${userId}`;
        document.getElementById('remoteVideos').appendChild(video);
    }
    
    removePeerConnection(userId) {
        const peerConnection = this.peerConnections.get(userId);
        if (peerConnection) {
            peerConnection.close();
            this.peerConnections.delete(userId);
        }
        
        const video = document.getElementById(`remoteVideo_${userId}`);
        if (video) {
            video.remove();
        }
    }
}

// Sử dụng
const client = new WebRTCClient();
client.joinRoom('room1');
```

## Data Channel

```javascript
class DataChannelExample {
    constructor() {
        this.peerConnection = null;
        this.dataChannel = null;
        this.setupPeerConnection();
    }
    
    async setupPeerConnection() {
        this.peerConnection = new RTCPeerConnection({
            iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
        });
        
        // Create data channel
        this.dataChannel = this.peerConnection.createDataChannel('messages', {
            ordered: true
        });
        
        this.dataChannel.onopen = () => {
            console.log('Data channel opened');
            this.sendMessage('Hello from data channel!');
        };
        
        this.dataChannel.onmessage = (event) => {
            console.log('Message received:', event.data);
            this.displayMessage(event.data);
        };
        
        this.dataChannel.onclose = () => {
            console.log('Data channel closed');
        };
        
        this.dataChannel.onerror = (error) => {
            console.error('Data channel error:', error);
        };
    }
    
    sendMessage(message) {
        if (this.dataChannel && this.dataChannel.readyState === 'open') {
            this.dataChannel.send(message);
        }
    }
    
    displayMessage(message) {
        const div = document.createElement('div');
        div.textContent = new Date().toLocaleTimeString() + ': ' + message;
        document.getElementById('messages').appendChild(div);
    }
}
```

## Screen Sharing

```javascript
class ScreenShare {
    async startScreenShare() {
        try {
            const stream = await navigator.mediaDevices.getDisplayMedia({
                video: true,
                audio: true
            });
            
            // Replace video source
            const video = document.getElementById('localVideo');
            video.srcObject = stream;
            
            // Add to peer connection
            if (this.peerConnection) {
                const videoTrack = stream.getVideoTracks()[0];
                const sender = this.peerConnection.getSenders().find(s => 
                    s.track && s.track.kind === 'video'
                );
                
                if (sender) {
                    sender.replaceTrack(videoTrack);
                }
            }
            
            // Handle when user stops sharing
            stream.getVideoTracks()[0].onended = () => {
                this.stopScreenShare();
            };
            
        } catch (error) {
            console.error('Error starting screen share:', error);
        }
    }
    
    stopScreenShare() {
        // Switch back to camera
        navigator.mediaDevices.getUserMedia({ video: true, audio: true })
            .then(stream => {
                document.getElementById('localVideo').srcObject = stream;
            });
    }
}
```

## Kết luận

WebRTC cung cấp khả năng giao tiếp real-time mạnh mẽ cho web applications. Một số điểm quan trọng:

- Cần signaling server để trao đổi offer/answer
- STUN/TURN servers để NAT traversal
- Xử lý lỗi và reconnection
- Hỗ trợ video, audio và data channels
- Screen sharing capabilities

WebRTC phù hợp cho:
- Video/audio calling
- Screen sharing
- File transfer P2P
- Gaming
- Live streaming

Trong bài viết tiếp theo, chúng ta sẽ tìm hiểu về WebSocket với Node.js và xây dựng real-time chat application hoàn chỉnh.

