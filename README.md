## **Hybrid Chat App Backend – Project Structure**  

```
chat-app-backend/
│── server/                  # Node.js backend (WebSockets + API)
│   ├── node_modules/        # Node.js dependencies
│   ├── src/                 # Source code for Node.js server
│   │   ├── routes/          # Express API routes
│   │   │   ├── message.js   # Message API (calls C++ service)
│   │   ├── services/        # Business logic
│   │   │   ├── encryptService.js  # Calls C++ encryption service
│   │   ├── server.js        # WebSocket + API server
│   ├── package.json         # Node.js dependencies
│   ├── .env                 # Environment variables
│── cpp-service/             # C++ microservice (encryption)
│   ├── encryptor.cpp        # Encryption microservice (REST API)
│   ├── Makefile             # Build instructions for C++ service
│   ├── bin/                 # Compiled C++ binaries
│── config/                  # Config files for both Node.js & C++
│   ├── config.json          # Configuration (e.g., API URLs, ports)
│── logs/                    # Log files for debugging
│── README.md                # Project documentation
```

---

### **1️⃣ Node.js Backend (`server/`)**
This handles **WebSocket connections**, **API requests**, and **communication with the C++ microservice**.

#### **📌 `server/src/server.js` (Main Server)**
```javascript
require('dotenv').config();
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const encryptService = require('./services/encryptService');

const app = express();
const server = http.createServer(app);
const io = new Server(server);

app.use(express.json());

// WebSocket logic
io.on('connection', (socket) => {
    console.log(`User connected: ${socket.id}`);

    socket.on('chat message', async (data) => {
        try {
            const encryptedMessage = await encryptService.encryptMessage(data.message);
            io.emit('chat message', { sender: socket.id, message: encryptedMessage });
        } catch (error) {
            console.error('Encryption error:', error);
        }
    });

    socket.on('disconnect', () => {
        console.log(`User disconnected: ${socket.id}`);
    });
});

server.listen(process.env.PORT || 3000, () => {
    console.log(`Server running on port ${process.env.PORT || 3000}`);
});
```

#### **📌 `server/src/services/encryptService.js` (Calls C++ Microservice)**
```javascript
const axios = require('axios');

async function encryptMessage(message) {
    try {
        const response = await axios.post('http://localhost:5000/encrypt', { message });
        return response.data.encrypted;
    } catch (error) {
        console.error('Error calling C++ service:', error);
        throw error;
    }
}

module.exports = { encryptMessage };
```

#### **📌 `server/src/routes/message.js` (REST API for messages)**
```javascript
const express = require('express');
const encryptService = require('../services/encryptService');
const router = express.Router();

router.post('/send', async (req, res) => {
    try {
        const { message } = req.body;
        const encryptedMessage = await encryptService.encryptMessage(message);
        res.json({ encryptedMessage });
    } catch (error) {
        res.status(500).json({ error: 'Encryption failed' });
    }
});

module.exports = router;
```

#### **📌 `.env` (Environment Variables)**
```
PORT=3000
CPP_SERVICE_URL=http://localhost:5000/encrypt
```

---

### **2️⃣ C++ Microservice (`cpp-service/`)**
Handles **AES encryption** and serves as a **REST API**.

#### **📌 `cpp-service/encryptor.cpp` (C++ Microservice)**
```cpp
#include <iostream>
#include <cpprest/http_listener.h>
#include <cpprest/json.h>
#include <openssl/aes.h>

using namespace web;
using namespace web::http;
using namespace web::http::experimental::listener;

const unsigned char key[16] = "1234567890abcdef"; // Simple AES key

std::string encrypt_message(const std::string &message) {
    AES_KEY encryptKey;
    AES_set_encrypt_key(key, 128, &encryptKey);

    unsigned char encrypted[128];
    AES_encrypt((const unsigned char *)message.c_str(), encrypted, &encryptKey);

    return std::string((char *)encrypted, 16);
}

void handle_post(http_request request) {
    request.extract_json().then([=](json::value body) {
        std::string message = body[U("message")].as_string();
        std::string encrypted = encrypt_message(message);

        json::value response;
        response[U("encrypted")] = json::value::string(encrypted);
        request.reply(status_codes::OK, response);
    });
}

int main() {
    http_listener listener(U("http://localhost:5000/encrypt"));
    listener.support(methods::POST, handle_post);

    try {
        listener.open().wait();
        std::cout << "C++ Encryption Microservice Running on port 5000...\n";
        std::cin.get(); // Keep running
    } catch (std::exception &e) {
        std::cerr << "Error: " << e.what() << std::endl;
    }
    return 0;
}
```

---

### **3️⃣ Building & Running the Hybrid Chat Backend**

#### **📌 Step 1: Install Dependencies (Node.js)**
```sh
cd server/
npm install
```

#### **📌 Step 2: Start the C++ Microservice**
```sh
cd cpp-service/
g++ encryptor.cpp -o encryptor -lssl -lcrypto -lcpprest
./encryptor
```

#### **📌 Step 3: Start the Node.js Server**
```sh
cd server/
node src/server.js
```

---

## **4️⃣ Future Enhancements 🚀**
✅ **Use gRPC instead of REST API for faster Node.js ↔ C++ communication**  
✅ **Store messages in a database (MongoDB/PostgreSQL)**  
✅ **Add authentication (JWT/OAuth)**  
✅ **Use Kubernetes for microservice scaling**  
