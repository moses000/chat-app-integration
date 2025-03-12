### **Hybrid Chat App Backend: Node.js (Main Server) + C++ (Microservice for Encryption)**  

1. **Node.js** serves as the **main backend**, handling WebSockets and API requests.  
2. **C++** acts as a **microservice**, handling **message encryption and decryption** before sending messages over the WebSocket.  

---

## **1️⃣ Node.js - WebSocket Server + API**  
This will:  
✅ Handle WebSocket connections  
✅ Send/receive messages  
✅ Call the C++ microservice for **message encryption**  

### **📌 Install Dependencies**
```sh
npm install express socket.io axios body-parser
```

### **📌 Node.js (Server - `server.js`)**
```javascript
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const axios = require('axios'); // Call C++ microservice

const app = express();
const server = http.createServer(app);
const io = new Server(server);

app.use(express.json());

// Store connected users
let users = {};

io.on('connection', (socket) => {
    console.log(`User connected: ${socket.id}`);

    socket.on('chat message', async (data) => {
        console.log(`Received from ${socket.id}: ${data.message}`);

        // Encrypt message via C++ microservice
        try {
            const response = await axios.post('http://localhost:5000/encrypt', { message: data.message });
            const encryptedMessage = response.data.encrypted;

            // Broadcast encrypted message
            io.emit('chat message', { sender: socket.id, message: encryptedMessage });
        } catch (error) {
            console.error('Encryption error:', error);
        }
    });

    socket.on('disconnect', () => {
        console.log(`User disconnected: ${socket.id}`);
    });
});

server.listen(3000, () => {
    console.log('Chat server running on http://localhost:3000');
});
```

---

## **2️⃣ C++ - Microservice for Message Encryption**  
This will:  
✅ Receive messages from the **Node.js** backend  
✅ Encrypt messages using **AES (Advanced Encryption Standard)**  
✅ Send back the **encrypted message**  

### **📌 Install Dependencies**  
```sh
sudo apt install libcpprest-dev  # Install C++ REST SDK (for API server)
sudo apt install libssl-dev      # For OpenSSL encryption
```

### **📌 C++ Microservice (`encryptor.cpp`)**
```cpp
#include <iostream>
#include <cpprest/http_listener.h>
#include <cpprest/json.h>
#include <openssl/aes.h>

using namespace web;
using namespace web::http;
using namespace web::http::experimental::listener;

const unsigned char key[16] = "1234567890abcdef"; // Simple AES key (change for security)

std::string encrypt_message(const std::string &message) {
    AES_KEY encryptKey;
    AES_set_encrypt_key(key, 128, &encryptKey);

    unsigned char encrypted[128];
    AES_encrypt((const unsigned char *)message.c_str(), encrypted, &encryptKey);

    return std::string((char *)encrypted, 16); // Return first 16 bytes
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

## **3️⃣ Running the Hybrid Chat Backend**
### **Start the C++ Microservice**  
```sh
g++ encryptor.cpp -o encryptor -lssl -lcrypto -lcpprest
./encryptor
```

### **Start the Node.js Server**  
```sh
node server.js
```

---

## **4️⃣ Explanation of the Hybrid Approach**
| Component       | Technology | Role |
|----------------|------------|-----------------------------|
| **Node.js**    | JavaScript | WebSocket handling & API |
| **Socket.io**  | JavaScript | Real-time chat communication |
| **Express.js** | JavaScript | REST API to communicate with C++ |
| **C++ (AES)**  | C++ | Message encryption & microservice |
| **Axios**      | JavaScript | Calls C++ microservice from Node.js |

---

### 🚀 **How it Works**
1️⃣ **User sends a chat message** → Node.js **calls the C++ microservice**  
2️⃣ **C++ encrypts the message** → Sends it **back to Node.js**  
3️⃣ **Node.js broadcasts the encrypted message** to other users  

---

## **5️⃣ Future Improvements**
✅ **Use stronger AES encryption** with better key management  
✅ **Store chat messages in a database** (e.g., MongoDB/PostgreSQL)  
✅ **Add authentication** (JWT, OAuth)  
✅ **Use gRPC** for even faster Node.js ↔ C++ communication  
