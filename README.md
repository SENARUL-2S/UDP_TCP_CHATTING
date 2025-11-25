# 🌐 Socket Programming Chat Application

A comprehensive **TCP/UDP socket programming demonstration** with a modern web-based interface. This project showcases both connection-oriented (TCP) and connectionless (UDP) communication protocols with an intuitive, real-time chat interface.

![Socket Programming](https://img.shields.io/badge/Socket-Programming-blue)
![TCP/UDP](https://img.shields.io/badge/Protocol-TCP%2FUDP-green)
![Flask](https://img.shields.io/badge/Framework-Flask-red)
![SocketIO](https://img.shields.io/badge/RealTime-Socket.IO-yellow)

## 🚀 Features

### 📡 **Protocol Support**
- **TCP (Transmission Control Protocol)**: Reliable, connection-oriented communication
- **UDP (User Datagram Protocol)**: Fast, connectionless communication with 1000 character limit

### 🎭 **Role-Based Architecture**
- **Server Mode**: Listen for incoming connections
- **Client Mode**: Connect to existing servers
- **Turn-Based Messaging**: Organized communication flow

### 🎨 **Modern Web Interface**
- **Protocol Selection**: Choose TCP or UDP with visual indicators
- **Role Selection**: Server/Client selection with intuitive icons
- **Real-Time Messaging**: Instant message delivery using Socket.IO
- **Character Counter**: Live character counting for UDP mode
- **Send Options**: Both Enter key and click-to-send button
- **Responsive Design**: Works on desktop, tablet, and mobile
- **Beautiful UI**: Gradient backgrounds, animations, and modern styling

### ⚡ **Advanced Features**
- **Auto-Connection**: Automatic socket setup after role selection
- **Status Indicators**: Real-time connection status
- **Message Validation**: Character limits and turn-based validation
- **Error Handling**: Comprehensive error messages and recovery
- **Multiple Instances**: Support for multiple simultaneous connections

## 🛠️ **Installation**

### Prerequisites
- Python 3.7 or higher
- pip package manager

### Install Dependencies
```bash
pip install -r requirements.txt
```

### Required Packages
```
Flask>=2.0.0
Flask-SocketIO>=5.3.2
gevent>=21.8.0
python-socketio>=5.0.0
```

## 🎯 **Quick Start**

### 1. Start the Application
```bash
python app.py
```

### 2. Open Web Interface
Navigate to: **http://127.0.0.1:5002**

### 3. Choose Protocol and Role
1. **Select Protocol**: TCP (reliable) or UDP (fast, 1000 char limit)
2. **Choose Role**: Server (listen) or Client (connect)
3. **Start Chatting**: Type messages and send!

### 4. Test with Multiple Instances
1. Open multiple browser tabs
2. Set one as SERVER, another as CLIENT  
3. Start real-time communication

## 📋 **Detailed Usage**

### TCP Mode
- **Reliable Communication**: Guaranteed message delivery
- **Connection-Based**: Establishes persistent connection
- **No Character Limit**: Send messages of any length
- **Turn-Based**: Organized messaging flow

### UDP Mode  
- **Fast Communication**: Low-latency message delivery
- **Connectionless**: Direct peer-to-peer communication
- **1000 Character Limit**: Enforced message size limit
- **Character Counter**: Real-time character counting
- **Best Effort**: No delivery guarantee

## 🏗️ **Architecture**

### Backend Components
- **`app.py`**: Flask-SocketIO server with protocol handling
- **`tcp_peer.py`**: TCP socket implementation with turn-based logic
- **`udp_peer.py`**: UDP socket implementation with character limits
- **`requirements.txt`**: Python dependencies

### Frontend Components
- **`templates/index.html`**: Modern responsive web interface
- **`static/chat.js`**: Real-time messaging and UI logic

### Key Technologies
- **Flask**: Web framework for HTTP server
- **Socket.IO**: Real-time bidirectional communication
- **Gevent**: Asynchronous I/O for scalability
- **HTML5/CSS3**: Modern web interface
- **JavaScript**: Interactive client-side logic

## 💻 **Core Implementation**

### Main Flask Application (app.py)
```python
from flask import Flask, render_template, request
from flask_socketio import SocketIO, emit
import threading

def create_app():
    app = Flask(__name__)
    app.config['SECRET_KEY'] = 'socket-chat-secret'
    return app

def main():
    app = create_app()
    socketio = SocketIO(app, async_mode='gevent', cors_allowed_origins="*")
    peers = {}

    @app.route('/')
    def index():
        return render_template('index.html')

    @socketio.on('configure')
    def handle_configure(config):
        session_id = request.sid
        
        # Dynamic protocol selection
        if config.get('mode') == 'udp':
            from udp_peer import UDPPeer as Peer
            peer = Peer(
                role=config['role'],
                host='127.0.0.1',
                port=int(config['localPort']),
                peer_host='127.0.0.1',
                peer_port=int(config['peerPort']),
                start_sender=(config['startSender'] == 'true')
            )
        else:
            from tcp_peer import TCPPeer as Peer
            peer = Peer(
                role=config['role'],
                host='127.0.0.1',
                port=int(config['localPort']),
                peer_host='127.0.0.1',
                peer_port=int(config['peerPort']),
                start_sender=(config['startSender'] == 'true')
            )
        
        # Set up message callback
        def on_recv(text):
            socketio.emit('recv', {'text': text}, room=session_id)
            socketio.emit('meta', {
                'ready': peer.is_ready(), 
                'your_turn': peer.your_turn
            }, room=session_id)
        
        peer.set_recv_callback(on_recv)
        peers[session_id] = peer
        
        # Start peer in background thread
        def start_peer():
            try:
                peer.start()
                socketio.emit('configured', {'success': True}, room=session_id)
            except Exception as e:
                socketio.emit('configured', {'success': False, 'error': str(e)}, room=session_id)
        
        threading.Thread(target=start_peer, daemon=True).start()

    @socketio.on('send')
    def handle_send(data):
        if request.sid in peers:
            try:
                peers[request.sid].send(data['text'])
                emit('sent', {'text': data['text']})
            except Exception as e:
                emit('error', {'error': str(e)})

    socketio.run(app, host='0.0.0.0', port=5002, debug=False)
```

### TCP Socket Implementation (tcp_peer.py)
```python
import socket
import threading

class TCPPeer:
    def __init__(self, role, host, port, peer_host, peer_port, start_sender=False):
        self.role = role
        self.host = host
        self.port = port
        self.peer_host = peer_host
        self.peer_port = peer_port
        self.your_turn = start_sender
        self.socket = None
        self.peer_socket = None
        self.running = False
        self.recv_callback = None

    def start(self):
        if self.role == 'server':
            self._start_server()
        else:
            self._start_client()

    def _start_server(self):
        self.socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.socket.bind((self.host, self.port))
        self.socket.listen(1)
        
        print(f'TCP server listening on {self.host}:{self.port}')
        self.peer_socket, addr = self.socket.accept()
        print(f'Accepted connection from {addr}')
        
        self.running = True
        self._start_recv_thread()

    def _start_client(self):
        self.socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.socket.connect((self.peer_host, self.peer_port))
        print(f'Connected to {self.peer_host}:{self.peer_port}')
        
        self.peer_socket = self.socket
        self.running = True
        self._start_recv_thread()

    def send(self, message):
        if self.peer_socket and self.running:
            self.peer_socket.send((message + '\n').encode())
            self.your_turn = False

    def _start_recv_thread(self):
        def recv_loop():
            buffer = ""
            while self.running:
                try:
                    data = self.peer_socket.recv(1024).decode()
                    if not data:
                        break
                    
                    buffer += data
                    while '\n' in buffer:
                        line, buffer = buffer.split('\n', 1)
                        if line.strip() and self.recv_callback:
                            self.your_turn = True
                            self.recv_callback(line.strip())
                except Exception as e:
                    print(f'TCP recv error: {e}')
                    break
        
        threading.Thread(target=recv_loop, daemon=True).start()
```

### UDP Socket Implementation (udp_peer.py)
```python
import socket
import threading

class UDPPeer:
    def __init__(self, role, host, port, peer_host, peer_port, start_sender=False):
        self.role = role
        self.host = host
        self.port = port
        self.peer_host = peer_host
        self.peer_port = peer_port
        self.your_turn = start_sender
        self.socket = None
        self.running = False
        self.recv_callback = None
        self.MAX_MESSAGE_SIZE = 1000

    def start(self):
        self.socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.socket.bind((self.host, self.port))
        
        print(f'UDP {self.role} listening on {self.host}:{self.port}')
        self.running = True
        
        # Start receiving thread
        threading.Thread(target=self._recv_loop, daemon=True).start()
        
        # Send initial ping for connection establishment
        if self.role == 'client':
            self._send_ping()

    def send(self, message):
        if len(message) > self.MAX_MESSAGE_SIZE:
            message = message[:self.MAX_MESSAGE_SIZE]
        
        try:
            self.socket.sendto(message.encode('utf-8'), (self.peer_host, self.peer_port))
            self.your_turn = False
            return True
        except Exception as e:
            raise e

    def _recv_loop(self):
        while self.running:
            try:
                data, addr = self.socket.recvfrom(self.MAX_MESSAGE_SIZE + 100)
                message = data.decode('utf-8').strip()
                
                # Handle connection messages
                if message in ["PING", "PONG"]:
                    if message == "PING":
                        self.socket.sendto("PONG".encode('utf-8'), addr)
                    continue
                
                # Process actual message
                if message and self.recv_callback:
                    self.your_turn = True
                    self.recv_callback(message)
            except Exception as e:
                if self.running:
                    print(f'UDP receive error: {e}')
                break

    def _send_ping(self):
        try:
            self.socket.sendto("PING".encode('utf-8'), (self.peer_host, self.peer_port))
        except Exception as e:
            print(f'Failed to send ping: {e}')
```

### Frontend JavaScript (chat.js)
```javascript
let socket = null;
let currentMode = 'tcp';
let currentRole = null;
let isConnected = false;
const UDP_MAX_CHARS = 1000;

// Protocol selection
function connectWithConfig(config) {
    socket = io();
    setupSocketListeners();
    
    socket.on('connect', () => {
        socket.emit('configure', config);
    });
}

// Message sending with validation
function sendMessage() {
    const text = input.value.trim();
    if (!text || !socket || !isConnected) return;
    
    // UDP character limit validation
    if (currentMode === 'udp' && text.length > UDP_MAX_CHARS) {
        appendMessage(`❌ Message too long! UDP limit is ${UDP_MAX_CHARS} characters.`, 'error');
        return;
    }
    
    socket.emit('send', { text });
    input.value = '';
    updateCharCounter();
}

// Real-time character counting for UDP
function updateCharCounter() {
    if (currentMode !== 'udp') {
        charCounter.style.display = 'none';
        return;
    }
    
    const length = input.value.length;
    charCounter.textContent = `${length}/${UDP_MAX_CHARS} characters`;
    charCounter.className = 'char-counter';
    
    if (length > UDP_MAX_CHARS * 0.8) charCounter.classList.add('warning');
    if (length >= UDP_MAX_CHARS) charCounter.classList.add('error');
}

// Socket event handlers
function setupSocketListeners() {
    socket.on('configured', (data) => {
        if (data.success) {
            isConnected = true;
            updateTurnIndicator();
        } else {
            appendMessage(`❌ Failed to start ${currentRole}: ${data.error}`, 'error');
        }
    });
    
    socket.on('recv', (d) => {
        appendMessage(d.text, 'received');
        updateTurnIndicator();
    });
    
    socket.on('sent', (d) => {
        appendMessage(d.text, 'sent');
        updateTurnIndicator();
    });
}
```

## 🔧 **Configuration**

### Default Ports
- **Web Interface**: 5002
- **TCP Server**: 9000
- **TCP Client**: 9001  
- **UDP Communication**: 9000-9001

### Customization
All ports and settings are automatically configured through the web interface. Advanced users can modify the source code for custom configurations.

## 🌟 **Educational Value**

This project demonstrates key socket programming concepts:

### TCP Concepts
- **Connection Establishment**: Three-way handshake
- **Reliable Delivery**: Acknowledgments and retransmission
- **Flow Control**: Preventing buffer overflow
- **Connection Termination**: Graceful closure

### UDP Concepts  
- **Connectionless Communication**: Direct message sending
- **Best Effort Delivery**: No reliability guarantees
- **Message Boundaries**: Discrete packet transmission
- **Size Limitations**: MTU and practical limits

### General Networking
- **Client-Server Architecture**: Role-based communication
- **Real-Time Communication**: Bidirectional message flow
- **Protocol Selection**: Choosing appropriate protocols
- **Error Handling**: Network failure recovery

## 🐛 **Troubleshooting**

### Common Issues

**Port Already in Use**
```bash
# Kill existing processes
netstat -ano | findstr :5002
taskkill /F /PID <process_id>
```

**Connection Failed**
- Ensure both peers select different roles (Server/Client)
- Check firewall settings
- Verify port availability

**UDP Messages Not Delivered**  
- Check character limit (1000 max)
- Verify network connectivity
- Consider packet loss in UDP

### Debugging Tips
- Check browser developer console for JavaScript errors
- Monitor terminal output for backend errors
- Test with localhost first, then network connections
- Use browser network tab to monitor WebSocket connections

## 🤝 **Contributing**

Feel free to contribute improvements:
1. Fork the repository
2. Create feature branch
3. Make improvements
4. Submit pull request

## 📚 **Learning Resources**

- **Socket Programming**: Understanding network communication
- **TCP vs UDP**: Protocol comparison and use cases  
- **Flask-SocketIO**: Real-time web applications
- **Network Programming**: Client-server architectures

## 📄 **License**

This project is for educational purposes in Computer Networks coursework.

## 🎓 **Academic Context**

**Course**: Computer Networks Sessional (CSE-314)  
**Level**: 3rd Year, 1st Semester  
**Topic**: Socket Programming with TCP and UDP protocols

---

**Happy Socket Programming! 🚀**

Files
- `app.py` — Flask web app and Socket.IO glue
- `tcp_peer.py` — TCP peer implementation
- `udp_peer.py` — UDP peer implementation
- `templates/index.html` and `static/chat.js` — web UI
