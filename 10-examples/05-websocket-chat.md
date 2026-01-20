# রিয়েল-টাইম চ্যাট (WebSocket)

WebSocket ব্যবহার করে রিয়েল-টাইম চ্যাট অ্যাপ্লিকেশন তৈরি করুন।

## প্রজেক্ট স্ট্রাকচার

```
chat-api/
├── main.go
├── models/
│   ├── message.go
│   └── room.go
├── handlers/
│   └── chat.go
├── hub/
│   └── hub.go
└── public/
    └── index.html
```

## ধাপ ১: ডিপেন্ডেন্সি ইনস্টল করুন

```bash
go get github.com/gofiber/fiber/v3
go get github.com/gofiber/contrib/websocket
go get github.com/google/uuid
```

## ধাপ ২: Models তৈরি করুন

**models/message.go**

```go
package models

import "time"

type Message struct {
    ID        string    `json:"id"`
    RoomID    string    `json:"room_id"`
    UserID    string    `json:"user_id"`
    Username  string    `json:"username"`
    Content   string    `json:"content"`
    Type      string    `json:"type"` // text, join, leave, typing
    Timestamp time.Time `json:"timestamp"`
}

type MessageRequest struct {
    Content string `json:"content"`
    Type    string `json:"type"`
}
```

**models/room.go**

```go
package models

type Room struct {
    ID      string             `json:"id"`
    Name    string             `json:"name"`
    Clients map[string]*Client `json:"-"`
}

type Client struct {
    ID       string
    Username string
    RoomID   string
    Conn     interface{} // websocket.Conn
}
```

## ধাপ ৩: Chat Hub (Central Manager)

**hub/hub.go**

```go
package hub

import (
    "chat-api/models"
    "sync"
    "time"

    "github.com/gofiber/contrib/websocket"
    "github.com/google/uuid"
)

type Hub struct {
    rooms      map[string]*models.Room
    register   chan *models.Client
    unregister chan *models.Client
    broadcast  chan *BroadcastMessage
    mu         sync.RWMutex
}

type BroadcastMessage struct {
    RoomID  string
    Message *models.Message
}

var instance *Hub
var once sync.Once

// Singleton Hub ইনস্ট্যান্স পান
func GetHub() *Hub {
    once.Do(func() {
        instance = &Hub{
            rooms:      make(map[string]*models.Room),
            register:   make(chan *models.Client),
            unregister: make(chan *models.Client),
            broadcast:  make(chan *BroadcastMessage),
        }
        go instance.Run()
    })
    return instance
}

// Hub চালান
func (h *Hub) Run() {
    for {
        select {
        case client := <-h.register:
            h.RegisterClient(client)

        case client := <-h.unregister:
            h.UnregisterClient(client)

        case message := <-h.broadcast:
            h.BroadcastToRoom(message.RoomID, message.Message)
        }
    }
}

// ক্লায়েন্ট রেজিস্টার করুন
func (h *Hub) RegisterClient(client *models.Client) {
    h.mu.Lock()
    defer h.mu.Unlock()

    room, exists := h.rooms[client.RoomID]
    if !exists {
        room = &models.Room{
            ID:      client.RoomID,
            Name:    client.RoomID,
            Clients: make(map[string]*Client),
        }
        h.rooms[client.RoomID] = room
    }

    room.Clients[client.ID] = client

    // Join মেসেজ পাঠান
    joinMsg := &models.Message{
        ID:        uuid.New().String(),
        RoomID:    client.RoomID,
        UserID:    client.ID,
        Username:  client.Username,
        Content:   client.Username + " joined the room",
        Type:      "join",
        Timestamp: time.Now(),
    }

    h.broadcast <- &BroadcastMessage{
        RoomID:  client.RoomID,
        Message: joinMsg,
    }
}

// ক্লায়েন্ট আনরেজিস্টার করুন
func (h *Hub) UnregisterClient(client *models.Client) {
    h.mu.Lock()
    defer h.mu.Unlock()

    room, exists := h.rooms[client.RoomID]
    if !exists {
        return
    }

    if _, ok := room.Clients[client.ID]; ok {
        delete(room.Clients, client.ID)

        // Leave মেসেজ পাঠান
        leaveMsg := &models.Message{
            ID:        uuid.New().String(),
            RoomID:    client.RoomID,
            UserID:    client.ID,
            Username:  client.Username,
            Content:   client.Username + " left the room",
            Type:      "leave",
            Timestamp: time.Now(),
        }

        h.broadcast <- &BroadcastMessage{
            RoomID:  client.RoomID,
            Message: leaveMsg,
        }

        // রুম খালি হলে ডিলিট করুন
        if len(room.Clients) == 0 {
            delete(h.rooms, client.RoomID)
        }
    }
}

// রুমে মেসেজ ব্রডকাস্ট করুন
func (h *Hub) BroadcastToRoom(roomID string, message *models.Message) {
    h.mu.RLock()
    defer h.mu.RUnlock()

    room, exists := h.rooms[roomID]
    if !exists {
        return
    }

    for _, client := range room.Clients {
        conn := client.Conn.(*websocket.Conn)
        if err := conn.WriteJSON(message); err != nil {
            // এরর হলে ক্লায়েন্ট আনরেজিস্টার করুন
            go func(c *models.Client) {
                h.unregister <- c
            }(client)
        }
    }
}

// রুম তথ্য পান
func (h *Hub) GetRoom(roomID string) (*models.Room, bool) {
    h.mu.RLock()
    defer h.mu.RUnlock()
    room, exists := h.rooms[roomID]
    return room, exists
}

// সব রুম পান
func (h *Hub) GetAllRooms() []*models.Room {
    h.mu.RLock()
    defer h.mu.RUnlock()

    rooms := make([]*models.Room, 0, len(h.rooms))
    for _, room := range h.rooms {
        rooms = append(rooms, room)
    }
    return rooms
}

// রুমে মেসেজ পাঠান
func (h *Hub) SendMessage(roomID string, message *models.Message) {
    h.broadcast <- &BroadcastMessage{
        RoomID:  roomID,
        Message: message,
    }
}
```

## ধাপ ৪: WebSocket Handler

**handlers/chat.go**

```go
package handlers

import (
    "chat-api/hub"
    "chat-api/models"
    "log"
    "time"

    "github.com/gofiber/contrib/websocket"
    "github.com/gofiber/fiber/v3"
    "github.com/google/uuid"
)

type ChatHandler struct {
    hub *hub.Hub
}

func NewChatHandler() *ChatHandler {
    return &ChatHandler{
        hub: hub.GetHub(),
    }
}

// WebSocket Upgrade Middleware
func (h *ChatHandler) WebSocketUpgrade(c fiber.Ctx) error {
    if websocket.IsWebSocketUpgrade(c) {
        return c.Next()
    }
    return fiber.ErrUpgradeRequired
}

// WebSocket Connection Handler
func (h *ChatHandler) HandleWebSocket(c *websocket.Conn) {
    // Query params থেকে তথ্য পান
    roomID := c.Query("room")
    username := c.Query("username")

    if roomID == "" {
        roomID = "general"
    }
    if username == "" {
        username = "Anonymous"
    }

    // ক্লায়েন্ট তৈরি করুন
    client := &models.Client{
        ID:       uuid.New().String(),
        Username: username,
        RoomID:   roomID,
        Conn:     c,
    }

    // Hub-এ রেজিস্টার করুন
    h.hub.RegisterClient(client)
    defer func() {
        h.hub.UnregisterClient(client)
        c.Close()
    }()

    // মেসেজ পড়ুন
    for {
        var req models.MessageRequest
        if err := c.ReadJSON(&req); err != nil {
            log.Println("Read error:", err)
            break
        }

        // মেসেজ তৈরি করুন
        message := &models.Message{
            ID:        uuid.New().String(),
            RoomID:    roomID,
            UserID:    client.ID,
            Username:  username,
            Content:   req.Content,
            Type:      req.Type,
            Timestamp: time.Now(),
        }

        // মেসেজ পাঠান
        h.hub.SendMessage(roomID, message)
    }
}

// GET /api/rooms - সব রুম পান
func (h *ChatHandler) GetRooms(c fiber.Ctx) error {
    rooms := h.hub.GetAllRooms()

    roomList := make([]fiber.Map, 0, len(rooms))
    for _, room := range rooms {
        roomList = append(roomList, fiber.Map{
            "id":           room.ID,
            "name":         room.Name,
            "client_count": len(room.Clients),
        })
    }

    return c.JSON(fiber.Map{
        "data": roomList,
    })
}

// GET /api/rooms/:id - রুম তথ্য পান
func (h *ChatHandler) GetRoom(c fiber.Ctx) error {
    roomID := c.Params("id")

    room, exists := h.hub.GetRoom(roomID)
    if !exists {
        return c.Status(fiber.StatusNotFound).JSON(fiber.Map{
            "error": "Room not found",
        })
    }

    clients := make([]fiber.Map, 0, len(room.Clients))
    for _, client := range room.Clients {
        clients = append(clients, fiber.Map{
            "id":       client.ID,
            "username": client.Username,
        })
    }

    return c.JSON(fiber.Map{
        "data": fiber.Map{
            "id":      room.ID,
            "name":    room.Name,
            "clients": clients,
        },
    })
}
```

## ধাপ ৫: Main Application

**main.go**

```go
package main

import (
    "log"
    "chat-api/handlers"

    "github.com/gofiber/fiber/v3"
    "github.com/gofiber/fiber/v3/middleware/cors"
    "github.com/gofiber/fiber/v3/middleware/logger"
    "github.com/gofiber/contrib/websocket"
)

func main() {
    app := fiber.New(fiber.Config{
        AppName: "Chat API v1.0",
    })

    // মিডলওয়্যার
    app.Use(logger.New())
    app.Use(cors.New())

    // স্ট্যাটিক ফাইল (HTML ক্লায়েন্ট)
    app.Static("/", "./public")

    // Handler ইনিশিয়ালাইজ করুন
    chatHandler := handlers.NewChatHandler()

    // REST API রাউট
    api := app.Group("/api")
    api.Get("/rooms", chatHandler.GetRooms)
    api.Get("/rooms/:id", chatHandler.GetRoom)

    // WebSocket রাউট
    app.Use("/ws", chatHandler.WebSocketUpgrade)
    app.Get("/ws", websocket.New(chatHandler.HandleWebSocket))

    log.Println("Server started on :3000")
    log.Fatal(app.Listen(":3000"))
}
```

## ধাপ ৬: HTML Client

**public/index.html**

```html
<!DOCTYPE html>
<html lang="bn">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>রিয়েল-টাইম চ্যাট</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; }
        
        .container { max-width: 800px; margin: 0 auto; padding: 20px; }
        
        .login-form { 
            background: #f5f5f5; 
            padding: 30px; 
            border-radius: 10px; 
            margin-top: 50px;
        }
        
        .chat-container { display: none; }
        .chat-container.active { display: block; }
        
        .chat-header {
            background: #007bff;
            color: white;
            padding: 15px;
            border-radius: 10px 10px 0 0;
        }
        
        .messages {
            height: 400px;
            overflow-y: auto;
            border: 1px solid #ddd;
            padding: 15px;
            background: white;
        }
        
        .message {
            margin-bottom: 15px;
            padding: 10px;
            border-radius: 5px;
            background: #f0f0f0;
        }
        
        .message.own {
            background: #007bff;
            color: white;
            text-align: right;
        }
        
        .message.system {
            background: #ffc107;
            text-align: center;
            font-style: italic;
        }
        
        .message-info {
            font-size: 12px;
            opacity: 0.7;
            margin-bottom: 5px;
        }
        
        .input-group {
            display: flex;
            gap: 10px;
            margin-top: 10px;
        }
        
        input, button {
            padding: 12px;
            border: 1px solid #ddd;
            border-radius: 5px;
            font-size: 14px;
        }
        
        input { flex: 1; }
        
        button {
            background: #007bff;
            color: white;
            border: none;
            cursor: pointer;
            min-width: 100px;
        }
        
        button:hover { background: #0056b3; }
        
        .status {
            padding: 10px;
            text-align: center;
            font-size: 12px;
        }
        
        .status.connected { background: #28a745; color: white; }
        .status.disconnected { background: #dc3545; color: white; }
    </style>
</head>
<body>
    <div class="container">
        <!-- Login Form -->
        <div class="login-form" id="loginForm">
            <h2>চ্যাটে যোগ দিন</h2>
            <div class="input-group">
                <input type="text" id="usernameInput" placeholder="আপনার নাম" required>
            </div>
            <div class="input-group">
                <input type="text" id="roomInput" placeholder="রুম নাম (ডিফল্ট: general)" value="general">
            </div>
            <div class="input-group">
                <button onclick="joinChat()">যোগ দিন</button>
            </div>
        </div>

        <!-- Chat Container -->
        <div class="chat-container" id="chatContainer">
            <div class="chat-header">
                <h2>রুম: <span id="roomName"></span></h2>
                <p>ইউজার: <span id="currentUser"></span></p>
            </div>
            
            <div class="status disconnected" id="status">সংযোগ বিচ্ছিন্ন</div>
            
            <div class="messages" id="messages"></div>
            
            <div class="input-group">
                <input type="text" id="messageInput" placeholder="মেসেজ লিখুন..." disabled>
                <button onclick="sendMessage()" id="sendBtn" disabled>পাঠান</button>
            </div>
        </div>
    </div>

    <script>
        let ws = null;
        let username = '';
        let roomID = '';

        function joinChat() {
            username = document.getElementById('usernameInput').value.trim();
            roomID = document.getElementById('roomInput').value.trim() || 'general';

            if (!username) {
                alert('নাম লিখুন');
                return;
            }

            // UI আপডেট করুন
            document.getElementById('loginForm').style.display = 'none';
            document.getElementById('chatContainer').classList.add('active');
            document.getElementById('roomName').textContent = roomID;
            document.getElementById('currentUser').textContent = username;

            // WebSocket কানেক্ট করুন
            connectWebSocket();
        }

        function connectWebSocket() {
            const wsUrl = `ws://localhost:3000/ws?room=${roomID}&username=${encodeURIComponent(username)}`;
            ws = new WebSocket(wsUrl);

            ws.onopen = () => {
                console.log('Connected to chat');
                updateStatus(true);
                document.getElementById('messageInput').disabled = false;
                document.getElementById('sendBtn').disabled = false;
            };

            ws.onmessage = (event) => {
                const message = JSON.parse(event.data);
                displayMessage(message);
            };

            ws.onerror = (error) => {
                console.error('WebSocket error:', error);
            };

            ws.onclose = () => {
                console.log('Disconnected from chat');
                updateStatus(false);
                document.getElementById('messageInput').disabled = true;
                document.getElementById('sendBtn').disabled = true;
            };
        }

        function sendMessage() {
            const input = document.getElementById('messageInput');
            const content = input.value.trim();

            if (!content || !ws) return;

            const message = {
                content: content,
                type: 'text'
            };

            ws.send(JSON.stringify(message));
            input.value = '';
        }

        function displayMessage(message) {
            const messagesDiv = document.getElementById('messages');
            const messageDiv = document.createElement('div');
            
            let className = 'message';
            if (message.type === 'join' || message.type === 'leave') {
                className += ' system';
            } else if (message.username === username) {
                className += ' own';
            }
            
            messageDiv.className = className;

            const time = new Date(message.timestamp).toLocaleTimeString('bn-BD');
            
            if (message.type === 'text') {
                messageDiv.innerHTML = `
                    <div class="message-info">${message.username} • ${time}</div>
                    <div>${message.content}</div>
                `;
            } else {
                messageDiv.innerHTML = `<div>${message.content}</div>`;
            }

            messagesDiv.appendChild(messageDiv);
            messagesDiv.scrollTop = messagesDiv.scrollHeight;
        }

        function updateStatus(connected) {
            const statusDiv = document.getElementById('status');
            if (connected) {
                statusDiv.textContent = 'সংযুক্ত';
                statusDiv.className = 'status connected';
            } else {
                statusDiv.textContent = 'সংযোগ বিচ্ছিন্ন';
                statusDiv.className = 'status disconnected';
            }
        }

        // Enter চাপলে মেসেজ পাঠান
        document.addEventListener('DOMContentLoaded', () => {
            document.getElementById('messageInput').addEventListener('keypress', (e) => {
                if (e.key === 'Enter') {
                    sendMessage();
                }
            });
        });
    </script>
</body>
</html>
```

## চালান

```bash
go run main.go
```

ব্রাউজারে খুলুন: `http://localhost:3000`

## API টেস্ট করুন

### 1. সব রুম দেখুন

```bash
curl http://localhost:3000/api/rooms
```

### 2. নির্দিষ্ট রুম দেখুন

```bash
curl http://localhost:3000/api/rooms/general
```

## WebSocket Client (JavaScript)

```javascript
const ws = new WebSocket('ws://localhost:3000/ws?room=general&username=রহিম');

ws.onopen = () => {
    console.log('Connected');
};

ws.onmessage = (event) => {
    const message = JSON.parse(event.data);
    console.log('Received:', message);
};

// মেসেজ পাঠান
ws.send(JSON.stringify({
    content: 'হ্যালো সবাই!',
    type: 'text'
}));
```

## Features

✅ রিয়েল-টাইম মেসেজিং  
✅ মাল্টিপল রুম সাপোর্ট  
✅ ইউজার join/leave নোটিফিকেশন  
✅ Thread-safe অপারেশন  
✅ Auto-reconnect সাপোর্ট  

## পরবর্তী উন্নতি

1. ✅ মেসেজ পার্সিস্টেন্স (ডাটাবেস)
2. ✅ প্রাইভেট মেসেজিং
3. ✅ ফাইল শেয়ারিং
4. ✅ Typing indicator
5. ✅ Message reactions
6. ✅ User authentication

---

[← আগের: ফাইল আপলোড](04-file-upload.md) | [পরবর্তী: Testing →](06-testing.md)
