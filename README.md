# sipalingIoT

# Sistem Smart Cam ESP32-CAM dengan MQTT

## 1. Arsitektur Sistem

### Diagram Alur
```
ESP32-CAM → WiFi → MQTT Broker → Web Dashboard
    ↓                    ↓              ↓
Motion Detection    Mosquitto     Real-time Updates
Photo Capture       (Docker)      Socket.IO/MQTT.js
JSON Status         Topic Mgmt    Photo Display
```

### Komponen Utama
- **ESP32-CAM**: Sensor kamera dengan motion detection
- **MQTT Broker**: Eclipse Mosquitto (containerized)
- **Web Dashboard**: Node.js + Express + Socket.IO + MQTT.js
- **Database**: File-based storage untuk log events
- **Docker**: Container orchestration

### Topik MQTT
- `security/cam/status`: Status dan metadata (JSON)
- `security/cam/photo`: Data foto dalam format base64/binary

---

## 2. Kode ESP32-CAM

### Dependencies untuk Arduino IDE
```cpp
// Library yang diperlukan:
// - WiFi (built-in ESP32)
// - PubSubClient by Nick O'Leary
// - ArduinoJson by Benoit Blanchon
// - ESP32 Camera by Espressif
```

### Main ESP32-CAM Code
```cpp
#include "esp_camera.h"
#include <WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include <base64.h>

// ===========================================
// KONFIGURASI - SESUAIKAN DENGAN SETUP ANDA
// ===========================================

// WiFi Credentials
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";

// MQTT Broker Configuration
const char* mqtt_server = "192.168.1.100";  // Ganti dengan IP server Docker
const int mqtt_port = 1883;
const char* mqtt_user = "";                  // Kosongkan jika tidak pakai auth
const char* mqtt_password = "";              // Kosongkan jika tidak pakai auth
const char* client_id = "ESP32_CAM_001";

// MQTT Topics
const char* topic_status = "security/cam/status";
const char* topic_photo = "security/cam/photo";

// Pin Configuration untuk ESP32-CAM AI-Thinker
#define PWDN_GPIO_NUM     32
#define RESET_GPIO_NUM    -1
#define XCLK_GPIO_NUM      0
#define SIOD_GPIO_NUM     26
#define SIOC_GPIO_NUM     27
#define Y9_GPIO_NUM       35
#define Y8_GPIO_NUM       34
#define Y7_GPIO_NUM       39
#define Y6_GPIO_NUM       36
#define Y5_GPIO_NUM       21
#define Y4_GPIO_NUM       19
#define Y3_GPIO_NUM       18
#define Y2_GPIO_NUM        5
#define VSYNC_GPIO_NUM    25
#define HREF_GPIO_NUM     23
#define PCLK_GPIO_NUM     22

// Motion Detection (menggunakan GPIO 13 sebagai RCWL sensor)
#define PIN_RCWL 13
#define PIN_LED_FLASH 4
#define PIN_LED_RED 15

// Global Variables
WiFiClient espClient;
PubSubClient client(espClient);
unsigned long lastMotionTime = 0;
unsigned long last_millis1 = 0, last_millis2 = 0;
const unsigned long motionCooldown = 5000; // 5 detik cooldown

// Status ruangan
bool ruang_tidak_dipakai = 0;
bool ruang_sedang_dipakai = 1;
bool status_ruangan = ruang_tidak_dipakai;
bool motion_detected = false;
bool flag_status_change = false;
int motion_state = LOW;

// ===========================================
// SETUP FUNCTIONS
// ===========================================

void setup() {
  Serial.begin(115200);
  Serial.println("=== ESP32-CAM Smart Security System ===");
  
  // Initialize pins
  pinMode(PIN_LED_FLASH, OUTPUT);
  digitalWrite(PIN_LED_FLASH, LOW);     // LED Flash OFF
  
  pinMode(PIN_LED_RED, OUTPUT);
  digitalWrite(PIN_LED_RED, HIGH);      // LED Red OFF (LOW = ON, HIGH = OFF)
  
  pinMode(PIN_RCWL, INPUT);             // RCWL motion sensor
  
  // Initialize camera
  if (!initCamera()) {
    Serial.println("Camera init failed!");
    ESP.restart();
  }
  
  // Connect to WiFi
  connectWiFi();
  
  // Setup MQTT
  client.setServer(mqtt_server, mqtt_port);
  client.setCallback(mqttCallback);
  
  // Send initial status
  connectMQTT();
  
  Serial.println("System ready!");
  last_millis1 = last_millis2 = millis();
}

bool initCamera() {
  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = Y2_GPIO_NUM;
  config.pin_d1 = Y3_GPIO_NUM;
  config.pin_d2 = Y4_GPIO_NUM;
  config.pin_d3 = Y5_GPIO_NUM;
  config.pin_d4 = Y6_GPIO_NUM;
  config.pin_d5 = Y7_GPIO_NUM;
  config.pin_d6 = Y8_GPIO_NUM;
  config.pin_d7 = Y9_GPIO_NUM;
  config.pin_xclk = XCLK_GPIO_NUM;
  config.pin_pclk = PCLK_GPIO_NUM;
  config.pin_vsync = VSYNC_GPIO_NUM;
  config.pin_href = HREF_GPIO_NUM;
  config.pin_sscb_sda = SIOD_GPIO_NUM;
  config.pin_sscb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn = PWDN_GPIO_NUM;
  config.pin_reset = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;
  
  // Frame size configuration
  if(psramFound()){
    config.frame_size = FRAMESIZE_UXGA;
    config.jpeg_quality = 10;
    config.fb_count = 2;
  } else {
    config.frame_size = FRAMESIZE_SVGA;
    config.jpeg_quality = 12;
    config.fb_count = 1;
  }
  
  // Initialize camera
  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Camera init failed with error 0x%x", err);
    return false;
  }
  
  return true;
}

void connectWiFi() {
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  
  Serial.println();
  Serial.println("WiFi connected!");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());
}

// ===========================================
// MQTT FUNCTIONS
// ===========================================

void mqttCallback(char* topic, byte* payload, unsigned int length) {
  // Handle incoming MQTT messages if needed
  Serial.printf("Message received on topic: %s\n", topic);
}

bool connectMQTT() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    
    if (client.connect(client_id, mqtt_user, mqtt_password)) {
      Serial.println("connected");
      
      // Send initial status
      publishStatus("system_startup", "ESP32-CAM initialized successfully");
      return true;
    } else {
      Serial.printf("failed, rc=%d. Retrying in 5 seconds...\n", client.state());
      delay(5000);
    }
  }
  return false;
}

void publishStatus(const char* event, const char* message) {
  StaticJsonDocument<300> doc;
  doc["timestamp"] = millis();
  doc["device_id"] = client_id;
  doc["event"] = event;
  doc["message"] = message;
  doc["ip_address"] = WiFi.localIP().toString();
  doc["rssi"] = WiFi.RSSI();
  doc["room_status"] = (status_ruangan == ruang_sedang_dipakai) ? "occupied" : "empty";
  doc["motion_state"] = (motion_state == HIGH) ? "active" : "inactive";
  doc["flag_status_change"] = flag_status_change;
  
  char buffer[350];
  serializeJson(doc, buffer);
  
  client.publish(topic_status, buffer);
  Serial.printf("Status published: %s\n", buffer);
}

bool captureAndPublishPhoto() {
  Serial.println("Taking a photo...");
  
  // Turn on LED Flash
  digitalWrite(PIN_LED_FLASH, HIGH);
  delay(500);
  
  // Dispose first picture because of bad quality
  camera_fb_t * fb = esp_camera_fb_get();
  if(fb) {
    esp_camera_fb_return(fb);
  }
  
  // Dispose second picture because of bad quality  
  fb = esp_camera_fb_get();
  if(fb) {
    esp_camera_fb_return(fb);
  }
  
  // Take the actual photo
  fb = esp_camera_fb_get();
  
  // Turn off LED Flash
  digitalWrite(PIN_LED_FLASH, LOW);
  
  if(!fb) {
    Serial.println("Camera capture failed");
    return false;
  }
  
  // Convert to base64 for MQTT transmission
  String imageBase64 = base64::encode(fb->buf, fb->len);
  
  // Create JSON payload with metadata
  StaticJsonDocument<100> metadata;
  metadata["timestamp"] = millis();
  metadata["device_id"] = client_id;
  metadata["format"] = "jpeg";
  metadata["size"] = fb->len;
  metadata["room_status"] = (status_ruangan == ruang_sedang_dipakai) ? "occupied" : "empty";
  
  char metaBuffer[150];
  serializeJson(metadata, metaBuffer);
  
  // Publish metadata first
  client.publish((String(topic_photo) + "/meta").c_str(), metaBuffer);
  
  // Publish photo data (may need to split for large images)
  const int maxChunkSize = 1000; // Adjust based on MQTT broker limits
  int totalChunks = (imageBase64.length() + maxChunkSize - 1) / maxChunkSize;
  
  for(int i = 0; i < totalChunks; i++) {
    String chunk = imageBase64.substring(i * maxChunkSize, (i + 1) * maxChunkSize);
    String chunkTopic = String(topic_photo) + "/chunk/" + String(i) + "/" + String(totalChunks);
    client.publish(chunkTopic.c_str(), chunk.c_str());
    delay(100); // Small delay between chunks
  }
  
  esp_camera_fb_return(fb);
  Serial.printf("Photo captured and published (%d bytes, %d chunks)\n", fb->len, totalChunks);
  return true;
}

// ===========================================
// MAIN LOOP
// ===========================================

void loop() {
  // Maintain MQTT connection
  if (!client.connected()) {
    connectMQTT();
  }
  client.loop();
  
  // Motion detection logic (berdasarkan kode original)
  if (digitalRead(PIN_RCWL) == HIGH) {     // terdeteksi gerakan
    digitalWrite(PIN_LED_RED, LOW);        // LED red ON
    
    if (motion_state == LOW) {
      Serial.println("Motion detected!");
      motion_state = HIGH;                 // update variable state to HIGH
      
      // Jika ruangan tidak dipakai, kirim foto dan notifikasi
      if (status_ruangan == ruang_tidak_dipakai) {
        // Publish status motion detected
        publishStatus("motion_detected", "Motion detected in empty room, capturing photo...");
        
        // Capture and publish photo
        if (captureAndPublishPhoto()) {
          publishStatus("photo_captured", "Photo captured and sent successfully");
        } else {
          publishStatus("photo_failed", "Failed to capture photo");
        }
      }
    }
    
    last_millis2 = millis();
    
    // Logika perubahan status ruangan
    if (status_ruangan == ruang_tidak_dipakai && flag_status_change == false) {
      flag_status_change = true;
      Serial.println("flag_status_change set to true");
      publishStatus("room_activity_detected", "Continuous motion detected, monitoring room status...");
      last_millis1 = millis();
    }
    
    // Jika motion terus terdeteksi selama 2 menit, ubah status jadi "sedang dipakai"
    if (flag_status_change == true && (millis() - last_millis1) / 1000 >= 120) { // 2 menit
      status_ruangan = ruang_sedang_dipakai;
      flag_status_change = false;
      Serial.println("status_ruangan : ruang_sedang_dipakai!");
      publishStatus("room_occupied", "Room status changed to OCCUPIED");
    }
  }
  else {
    digitalWrite(PIN_LED_RED, HIGH);       // LED red OFF
    
    if (motion_state == HIGH) {
      Serial.println("Motion stopped!");
      motion_state = LOW;                  // update variable state to LOW
      publishStatus("motion_stopped", "Motion detection ended");
    }
    
    // Jika ruangan sedang dipakai dan tidak ada motion selama 5 menit, ubah status jadi "tidak dipakai"
    if (status_ruangan == ruang_sedang_dipakai && (millis() - last_millis2) / 1000 >= 300) { // 5 menit
      status_ruangan = ruang_tidak_dipakai;
      flag_status_change = false;
      Serial.println("status_ruangan : ruang_tidak_dipakai!");
      publishStatus("room_empty", "Room status changed to EMPTY after 5 minutes of no motion");
    }
    
    // Reset flag jika tidak ada motion selama 3 menit
    if (status_ruangan == ruang_tidak_dipakai && flag_status_change == true && (millis() - last_millis1) / 1000 >= 180) { // 3 menit
      flag_status_change = false;
      Serial.println("flag_status_change set to false");
      publishStatus("room_monitoring_reset", "Room monitoring flag reset");
    }
  }
  
  delay(10); // Small delay seperti kode original
}
```

---

## 3. Web Dashboard

### Package.json
```json
{
  "name": "smart-cam-dashboard",
  "version": "1.0.0",
  "description": "ESP32-CAM Security Dashboard",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "socket.io": "^4.7.2",
    "mqtt": "^4.3.7",
    "multer": "^1.4.5",
    "fs-extra": "^11.1.1"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

### Server.js (Backend)
```javascript
const express = require('express');
const http = require('http');
const socketIo = require('socket.io');
const mqtt = require('mqtt');
const fs = require('fs-extra');
const path = require('path');

const app = express();
const server = http.createServer(app);
const io = socketIo(server);

// Configuration
const MQTT_BROKER = process.env.MQTT_BROKER || 'mqtt://localhost:1883';
const PORT = process.env.PORT || 3000;
const DATA_DIR = './data';
const PHOTOS_DIR = path.join(DATA_DIR, 'photos');

// Ensure directories exist
fs.ensureDirSync(DATA_DIR);
fs.ensureDirSync(PHOTOS_DIR);

// Middleware
app.use(express.static('public'));
app.use('/photos', express.static(PHOTOS_DIR));

// MQTT Client Setup
const mqttClient = mqtt.connect(MQTT_BROKER);
let photoChunks = {}; // Store photo chunks temporarily

mqttClient.on('connect', () => {
    console.log('Connected to MQTT broker');
    mqttClient.subscribe('security/cam/+');
    mqttClient.subscribe('security/cam/photo/+');
});

mqttClient.on('message', (topic, message) => {
    console.log(`Received message on topic: ${topic}`);
    
    if (topic === 'security/cam/status') {
        handleStatusMessage(message);
    } else if (topic.startsWith('security/cam/photo/chunk/')) {
        handlePhotoChunk(topic, message);
    } else if (topic === 'security/cam/photo/meta') {
        handlePhotoMetadata(message);
    }
});

function handleStatusMessage(message) {
    try {
        const status = JSON.parse(message.toString());
        
        // Log to file
        const logEntry = {
            ...status,
            received_at: new Date().toISOString()
        };
        
        const logFile = path.join(DATA_DIR, 'events.jsonl');
        fs.appendFileSync(logFile, JSON.stringify(logEntry) + '\n');
        
        // Broadcast to all connected clients
        io.emit('status_update', status);
        
        console.log('Status update:', status.event);
    } catch (error) {
        console.error('Error handling status message:', error);
    }
}

function handlePhotoMetadata(message) {
    try {
        const metadata = JSON.parse(message.toString());
        console.log('Photo metadata received:', metadata);
        
        // Reset chunks for new photo
        photoChunks = {
            metadata: metadata,
            chunks: {},
            totalChunks: 0
        };
    } catch (error) {
        console.error('Error handling photo metadata:', error);
    }
}

function handlePhotoChunk(topic, message) {
    try {
        // Parse topic: security/cam/photo/chunk/0/5
        const parts = topic.split('/');
        const chunkIndex = parseInt(parts[5]);
        const totalChunks = parseInt(parts[6]);
        
        photoChunks.chunks[chunkIndex] = message.toString();
        photoChunks.totalChunks = totalChunks;
        
        console.log(`Received chunk ${chunkIndex}/${totalChunks}`);
        
        // Check if we have all chunks
        if (Object.keys(photoChunks.chunks).length === totalChunks) {
            assembleAndSavePhoto();
        }
    } catch (error) {
        console.error('Error handling photo chunk:', error);
    }
}

function assembleAndSavePhoto() {
    try {
        // Assemble chunks in order
        let completeBase64 = '';
        for (let i = 0; i < photoChunks.totalChunks; i++) {
            completeBase64 += photoChunks.chunks[i];
        }
        
        // Convert base64 to buffer
        const imageBuffer = Buffer.from(completeBase64, 'base64');
        
        // Save photo with timestamp
        const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
        const filename = `photo_${timestamp}.jpg`;
        const filepath = path.join(PHOTOS_DIR, filename);
        
        fs.writeFileSync(filepath, imageBuffer);
        
        const photoInfo = {
            filename: filename,
            filepath: `/photos/${filename}`,
            timestamp: new Date().toISOString(),
            size: imageBuffer.length,
            metadata: photoChunks.metadata
        };
        
        // Log photo info
        const photoLogFile = path.join(DATA_DIR, 'photos.jsonl');
        fs.appendFileSync(photoLogFile, JSON.stringify(photoInfo) + '\n');
        
        // Broadcast to clients
        io.emit('new_photo', photoInfo);
        
        console.log(`Photo saved: ${filename} (${imageBuffer.length} bytes)`);
        
        // Clear chunks
        photoChunks = {};
    } catch (error) {
        console.error('Error assembling photo:', error);
    }
}

// Socket.IO connection handling
io.on('connection', (socket) => {
    console.log('Client connected');
    
    // Send recent events on connection
    socket.emit('recent_events', getRecentEvents());
    socket.emit('recent_photos', getRecentPhotos());
    
    socket.on('disconnect', () => {
        console.log('Client disconnected');
    });
});

function getRecentEvents(limit = 50) {
    try {
        const logFile = path.join(DATA_DIR, 'events.jsonl');
        if (!fs.existsSync(logFile)) return [];
        
        const lines = fs.readFileSync(logFile, 'utf8').trim().split('\n');
        return lines
            .slice(-limit)
            .map(line => JSON.parse(line))
            .reverse();
    } catch (error) {
        console.error('Error reading events:', error);
        return [];
    }
}

function getRecentPhotos(limit = 10) {
    try {
        const photoLogFile = path.join(DATA_DIR, 'photos.jsonl');
        if (!fs.existsSync(photoLogFile)) return [];
        
        const lines = fs.readFileSync(photoLogFile, 'utf8').trim().split('\n');
        return lines
            .slice(-limit)
            .map(line => JSON.parse(line))
            .reverse();
    } catch (error) {
        console.error('Error reading photos:', error);
        return [];
    }
}

// API Endpoints
app.get('/api/events', (req, res) => {
    res.json(getRecentEvents());
});

app.get('/api/photos', (req, res) => {
    res.json(getRecentPhotos());
});

// Start server
server.listen(PORT, () => {
    console.log(`Smart Cam Dashboard running on port ${PORT}`);
    console.log(`MQTT Broker: ${MQTT_BROKER}`);
});
```

### public/index.html (Frontend)
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Smart Cam Security Dashboard</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: #1a1a1a;
            color: #ffffff;
            min-height: 100vh;
        }
        
        .container {
            max-width: 1200px;
            margin: 0 auto;
            padding: 20px;
        }
        
        .header {
            text-align: center;
            margin-bottom: 30px;
            padding: 20px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            border-radius: 10px;
        }
        
        .header h1 {
            font-size: 2.5em;
            margin-bottom: 10px;
        }
        
        .status-indicator {
            display: inline-block;
            width: 12px;
            height: 12px;
            border-radius: 50%;
            margin-left: 10px;
        }
        
        .status-online { background: #4CAF50; }
        .status-offline { background: #f44336; }
        
        .dashboard {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 20px;
            margin-bottom: 20px;
        }
        
        .panel {
            background: #2d2d2d;
            border-radius: 10px;
            padding: 20px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        }
        
        .panel h2 {
            margin-bottom: 15px;
            color: #4CAF50;
            border-bottom: 2px solid #4CAF50;
            padding-bottom: 5px;
        }
        
        .current-photo {
            text-align: center;
        }
        
        .current-photo img {
            max-width: 100%;
            height: auto;
            border-radius: 8px;
            margin-bottom: 10px;
        }
        
        .photo-placeholder {
            width: 100%;
            height: 200px;
            background: #3d3d3d;
            border-radius: 8px;
            display: flex;
            align-items: center;
            justify-content: center;
            color: #888;
            margin-bottom: 10px;
        }
        
        .events-list {
            max-height: 400px;
            overflow-y: auto;
        }
        
        .event-item {
            background: #3d3d3d;
            margin-bottom: 10px;
            padding: 12px;
            border-radius: 6px;
            border-left: 4px solid #4CAF50;
        }
        
        .event-time {
            font-size: 0.8em;
            color: #888;
            margin-bottom: 5px;
        }
        
        .event-type {
            font-weight: bold;
            color: #4CAF50;
        }
        
        .event-message {
            margin-top: 5px;
            color: #ccc;
        }
        
        .photo-gallery {
            grid-column: 1 / -1;
        }
        
        .photo-thumbnails {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
            gap: 10px;
            max-height: 300px;
            overflow-y: auto;
        }
        
        .photo-thumb {
            position: relative;
            cursor: pointer;
            border-radius: 6px;
            overflow: hidden;
            transition: transform 0.2s;
        }
        
        .photo-thumb:hover {
            transform: scale(1.05);
        }
        
        .photo-thumb img {
            width: 100%;
            height: 100px;
            object-fit: cover;
        }
        
        .photo-thumb .photo-info {
            position: absolute;
            bottom: 0;
            left: 0;
            right: 0;
            background: rgba(0, 0, 0, 0.7);
            color: white;
            padding: 5px;
            font-size: 0.7em;
        }
        
        .controls {
            margin-bottom: 20px;
            text-align: center;
        }
        
        .btn {
            background: #4CAF50;
            color: white;
            border: none;
            padding: 10px 20px;
            border-radius: 5px;
            cursor: pointer;
            margin: 0 5px;
            transition: background 0.3s;
        }
        
        .btn:hover {
            background: #45a049;
        }
        
        .btn.danger {
            background: #f44336;
        }
        
        .btn.danger:hover {
            background: #da190b;
        }
        
        @media (max-width: 768px) {
            .dashboard {
                grid-template-columns: 1fr;
            }
            
            .photo-thumbnails {
                grid-template-columns: repeat(auto-fill, minmax(100px, 1fr));
            }
        }
        
        /* Loading animation */
        .loading {
            display: inline-block;
            width: 20px;
            height: 20px;
            border: 3px solid #f3f3f3;
            border-top: 3px solid #4CAF50;
            border-radius: 50%;
            animation: spin 1s linear infinite;
        }
        
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>Smart Cam Security Dashboard</h1>
            <p>ESP32-CAM MQTT Monitoring System <span id="connectionStatus" class="status-indicator status-offline"></span></p>
        </div>
        
        <div class="controls">
            <button class="btn" onclick="refreshData()">Refresh Data</button>
            <button class="btn" onclick="clearEvents()">Clear Events</button>
            <button class="btn danger" onclick="clearPhotos()">Clear Photos</button>
        </div>
        
        <div class="dashboard">
            <div class="panel">
                <h2>Latest Photo</h2>
                <div class="current-photo">
                    <div id="currentPhotoContainer">
                        <div class="photo-placeholder">
                            <span>Waiting for photo...</span>
                        </div>
                    </div>
                    <p id="photoInfo">No recent photos</p>
                </div>
            </div>
            
            <div class="panel">
                <h2>Recent Events</h2>
                <div id="eventsList" class="events-list">
                    <div class="event-item">
                        <div class="event-time">System starting...</div>
                        <div class="event-type">SYSTEM</div>
                        <div class="event-message">Dashboard initialized</div>
                    </div>
                </div>
            </div>
            
            <div class="panel photo-gallery">
                <h2>Photo Gallery</h2>
                <div id="photoGallery" class="photo-thumbnails">
                    <!-- Photos will be loaded here -->
                </div>
            </div>
        </div>
    </div>

    <script src="/socket.io/socket.io.js"></script>
    <script>
        // Socket.IO connection
        const socket = io();
        let isConnected = false;
        
        // Connection status
        socket.on('connect', () => {
            isConnected = true;
            updateConnectionStatus();
            console.log('Connected to server');
        });
        
        socket.on('disconnect', () => {
            isConnected = false;
            updateConnectionStatus();
            console.log('Disconnected from server');
        });
        
        function updateConnectionStatus() {
            const indicator = document.getElementById('connectionStatus');
            if (isConnected) {
                indicator.className = 'status-indicator status-online';
            } else {
                indicator.className = 'status-indicator status-offline';
            }
        }
        
        // Event handlers
        socket.on('status_update', (status) => {
            addEventToList(status);
        });
        
        socket.on('new_photo', (photoInfo) => {
            updateCurrentPhoto(photoInfo);
            addPhotoToGallery(photoInfo);
        });
        
        socket.on('recent_events', (events) => {
            displayEvents(events);
        });
        
        socket.on('recent_photos', (photos) => {
            displayPhotos(photos);
            if (photos.length > 0) {
                updateCurrentPhoto(photos[0]);
            }
        });
        
        function addEventToList(event) {
            const eventsList = document.getElementById('eventsList');
            const eventItem = createEventElement(event);
            
            // Add to top of list
            if (eventsList.firstChild) {
                eventsList.insertBefore(eventItem, eventsList.firstChild);
            } else {
                eventsList.appendChild(eventItem);
            }
            
            // Keep only last 20 events
            const events = eventsList.children;
            if (events.length > 20) {
                eventsList.removeChild(events[events.length - 1]);
            }
        }
        
        function createEventElement(event) {
            const div = document.createElement('div');
            div.className = 'event-item';
            
            const time = new Date(event.received_at || Date.now()).toLocaleString();
            
            div.innerHTML = `
                <div class="event-time">${time}</div>
                <div class="event-type">${event.event.toUpperCase()}</div>
                <div class="event-message">${event.message}</div>
            `;
            
            return div;
        }
        
        function displayEvents(events) {
            const eventsList = document.getElementById('eventsList');
            eventsList.innerHTML = '';
            
            events.forEach(event => {
                eventsList.appendChild(createEventElement(event));
            });
        }
        
        function updateCurrentPhoto(photoInfo) {
            const container = document.getElementById('currentPhotoContainer');
            const info = document.getElementById('photoInfo');
            
            container.innerHTML = `<img src="${photoInfo.filepath}" alt="Latest capture" />`;
            
            const time = new Date(photoInfo.timestamp).toLocaleString();
            const size = (photoInfo.size / 1024).toFixed(1);
            info.textContent = `Captured: ${time} (${size} KB)`;
        }
        
        function addPhotoToGallery(photoInfo) {
            const gallery = document.getElementById('photoGallery');
            const photoThumb = createPhotoThumbnail(photoInfo);
            
            // Add to beginning of gallery
            if (gallery.firstChild) {
                gallery.insertBefore(photoThumb, gallery.firstChild);
            } else {
                gallery.appendChild(photoThumb);
            }
            
            // Keep only last 50 photos
            const photos = gallery.children;
            if (photos.length > 50) {
                gallery.removeChild(photos[photos.length - 1]);
            }
        }
        
        function createPhotoThumbnail(photoInfo) {
            const div = document.createElement('div');
            div.className = 'photo-thumb';
            div.onclick = () => showFullPhoto(photoInfo);
            
            const time = new Date(photoInfo.timestamp).toLocaleString();
            const size = (photoInfo.size / 1024).toFixed(1);
            
            div.innerHTML = `
                <img src="${photoInfo.filepath}" alt="Photo thumbnail" />
                <div class="photo-info">
                    ${time}<br>${size} KB
                </div>
            `;
            
            return div;
        }
        
        function displayPhotos(photos) {
            const gallery = document.getElementById('photoGallery');
            gallery.innerHTML = '';
            
            photos.forEach(photo => {
                gallery.appendChild(createPhotoThumbnail(photo));
            });
        }
        
        function showFullPhoto(photoInfo) {
            // Create modal to show full-size photo
            const modal = document.createElement('div');
            modal.style.cssText = `
                position: fixed;
                top: 0;
                left: 0;
                width: 100%;
                height: 100%;
                background: rgba(0, 0, 0, 0.9);
                display: flex;
                align-items: center;
                justify-content: center;
                z-index: 1000;
                cursor: pointer;
            `;
            
            const img = document.createElement('img');
            img.src = photoInfo.filepath;
            img.style.cssText = `
                max-width: 90%;
                max-height: 90%;
                border-radius: 10px;
            `;
            
            modal.appendChild(img);
            modal.onclick = () => document.body.removeChild(modal);
            
            document.body.appendChild(modal);
        }
        
        // Control functions
        function refreshData() {
            socket.emit('refresh_data');
            location.reload();
        }
        
        function clearEvents() {
            if (confirm('Clear all events? This cannot be undone.')) {
                fetch('/api/clear-events', { method: 'POST' })
                    .then(() => location.reload());
            }
        }
        
        function clearPhotos() {
            if (confirm('Clear all photos? This cannot be undone.')) {
                fetch('/api/clear-photos', { method: 'POST' })
                    .then(() => location.reload());
            }
        }
        
        // Auto-refresh every 30 seconds
        setInterval(() => {
            if (isConnected) {
                socket.emit('heartbeat');
            }
        }, 30000);
    </script>
</body>
</html>
```

---

## 4. Docker Deployment

### docker-compose.yml
```yaml
version: '3.8'

services:
  # MQTT Broker (Eclipse Mosquitto)
  mosquitto:
    image: eclipse-mosquitto:2.0
    container_name: smart-cam-mqtt
    restart: unless-stopped
    ports:
      - "1883:1883"  # MQTT port
      - "9001:9001"  # WebSocket port
    volumes:
      - ./docker/mosquitto/config:/mosquitto/config
      - ./docker/mosquitto/data:/mosquitto/data
      - ./docker/mosquitto/log:/mosquitto/log
    environment:
      - MOSQUITTO_USERNAME=
      - MOSQUITTO_PASSWORD=
    networks:
      - smart-cam-network

  # Web Dashboard
  dashboard:
    build: 
      context: .
      dockerfile: Dockerfile
    container_name: smart-cam-dashboard
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - MQTT_BROKER=mqtt://mosquitto:1883
      - PORT=3000
    volumes:
      - ./data:/app/data
    depends_on:
      - mosquitto
    networks:
      - smart-cam-network

networks:
  smart-cam-network:
    driver: bridge

volumes:
  mosquitto-data:
  mosquitto-log:
```

### Dockerfile (untuk Dashboard)
```dockerfile
# Use Node.js official image
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application files
COPY . .

# Create data directory
RUN mkdir -p data/photos

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/api/events || exit 1

# Start application
CMD ["npm", "start"]
```

### docker/mosquitto/config/mosquitto.conf
```conf
# Mosquitto configuration for Smart Cam System

# General settings
persistence true
persistence_location /mosquitto/data/
log_dest file /mosquitto/log/mosquitto.log
log_dest stdout
log_type error
log_type warning
log_type notice
log_type information

# Network settings
listener 1883
protocol mqtt

# WebSocket listener (optional)
listener 9001
protocol websockets

# Security settings (uncomment and configure for production)
# password_file /mosquitto/config/password.txt
# acl_file /mosquitto/config/acl.txt

# Anonymous access (disable in production)
allow_anonymous true

# Connection limits
max_connections 100
max_inflight_messages 20
max_queued_messages 1000

# Message size limits
message_size_limit 10485760  # 10MB for large photos

# Keep alive
keepalive_interval 60

# Persistence settings
autosave_interval 1800
autosave_on_changes false

# Bridge settings (if connecting to external broker)
# connection external-broker
# address external-broker.example.com:1883
# topic security/# both 0
```

### Setup Scripts

#### setup.sh (Linux/macOS)
```bash
#!/bin/bash

echo "=== Smart Cam ESP32-CAM MQTT System Setup ==="

# Create directory structure
mkdir -p docker/mosquitto/{config,data,log}
mkdir -p data/photos
mkdir -p public

# Set permissions
chmod -R 755 docker/mosquitto
chmod -R 755 data

# Create .env file
cat > .env << EOF
MQTT_BROKER=mqtt://localhost:1883
PORT=3000
NODE_ENV=production
EOF

echo "Directory structure created successfully!"

# Build and start services
echo "Building and starting Docker containers..."
docker-compose up -d --build

echo "Waiting for services to start..."
sleep 10

# Check if services are running
if docker-compose ps | grep -q "Up"; then
    echo "✅ Services are running successfully!"
    echo ""
    echo "📊 Dashboard: http://localhost:3000"
    echo "🦟 MQTT Broker: localhost:1883"
    echo ""
    echo "Next steps:"
    echo "1. Configure your ESP32-CAM with WiFi credentials"
    echo "2. Update MQTT broker IP in ESP32 code"
    echo "3. Upload code to ESP32-CAM"
    echo "4. Open dashboard in browser"
else
    echo "❌ Some services failed to start. Check logs with:"
    echo "docker-compose logs"
fi
```

#### setup.bat (Windows)
```batch
@echo off
echo === Smart Cam ESP32-CAM MQTT System Setup ===

REM Create directory structure
if not exist "docker\mosquitto\config" mkdir docker\mosquitto\config
if not exist "docker\mosquitto\data" mkdir docker\mosquitto\data
if not exist "docker\mosquitto\log" mkdir docker\mosquitto\log
if not exist "data\photos" mkdir data\photos
if not exist "public" mkdir public

REM Create .env file
echo MQTT_BROKER=mqtt://localhost:1883 > .env
echo PORT=3000 >> .env
echo NODE_ENV=production >> .env

echo Directory structure created successfully!

REM Build and start services
echo Building and starting Docker containers...
docker-compose up -d --build

echo Waiting for services to start...
timeout /t 10 /nobreak > nul

echo Services setup complete!
echo.
echo Dashboard: http://localhost:3000
echo MQTT Broker: localhost:1883
echo.
echo Next steps:
echo 1. Configure your ESP32-CAM with WiFi credentials
echo 2. Update MQTT broker IP in ESP32 code
echo 3. Upload code to ESP32-CAM
echo 4. Open dashboard in browser

pause
```

---

## 5. Instruksi Penggunaan

### Persiapan Hardware
1. **ESP32-CAM Module**: Pastikan menggunakan ESP32-CAM AI-Thinker
2. **RCWL Motion Sensor**: Hubungkan ke GPIO 13 (VCC=3.3V, GND=GND, OUT=GPIO13)
3. **LED Indicators**: 
   - LED Flash pada GPIO 4 (built-in)
   - LED Red pada GPIO 15 (untuk indikator motion)
4. **Power Supply**: 5V untuk ESP32-CAM (jangan gunakan 3.3V untuk kamera)

### Konfigurasi Software

#### 1. Setup Development Environment
```bash
# Install Arduino IDE dan ESP32 board package
# Add board manager URL: https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json

# Install required libraries:
# - PubSubClient by Nick O'Leary
# - ArduinoJson by Benoit Blanchon
```

#### 2. Konfigurasi ESP32-CAM
```cpp
// Edit bagian ini di kode ESP32-CAM:
const char* ssid = "NAMA_WIFI_ANDA";
const char* password = "PASSWORD_WIFI_ANDA";
const char* mqtt_server = "192.168.1.100";  // IP server Docker
```

#### 3. Deploy Dashboard
```bash
# Clone atau download project
git clone <project-url>
cd smart-cam-system

# Run setup script
chmod +x setup.sh
./setup.sh

# Atau manual:
docker-compose up -d --build
```

#### 4. Verifikasi Koneksi
1. **Check MQTT Broker**:
   ```bash
   docker-compose logs mosquitto
   ```

2. **Check Dashboard**:
   ```bash
   docker-compose logs dashboard
   ```

3. **Test MQTT Connection**:
   ```bash
   # Install mosquitto-clients
   mosquitto_sub -h localhost -t "security/cam/+"
   ```

### Konfigurasi Network

#### Mendapatkan IP Address Server
```bash
# Linux/macOS
ifconfig | grep inet

# Windows
ipconfig

# Atau check di Docker
docker network inspect smart-cam-system_smart-cam-network
```

#### Port Forwarding (jika diperlukan)
- **MQTT**: Port 1883
- **Dashboard**: Port 3000
- **WebSocket MQTT**: Port 9001

### Troubleshooting

#### ESP32-CAM Issues
```cpp
// Jika kamera gagal init, coba:
config.frame_size = FRAMESIZE_VGA;  // Reduce resolution
config.jpeg_quality = 20;           // Reduce quality

// Jika WiFi tidak connect:
WiFi.mode(WIFI_STA);
WiFi.setAutoReconnect(true);

// Jika RCWL sensor terlalu sensitif:
#define MOTION_COOLDOWN 2000  // Increase cooldown period

// Jika LED tidak berfungsi:
pinMode(PIN_LED_RED, OUTPUT);
pinMode(PIN_LED_FLASH, OUTPUT);
```

#### MQTT Connection Issues
```bash
# Check broker status
docker-compose exec mosquitto mosquitto_pub -t test -m "hello"

# Check network connectivity
ping <mqtt_broker_ip>
telnet <mqtt_broker_ip> 1883
```

#### Dashboard Issues
```bash
# Check logs
docker-compose logs dashboard

# Restart services
docker-compose restart

# Rebuild if needed
docker-compose down
docker-compose up -d --build
```

### Monitoring & Maintenance

#### Log Files
- **Events**: `./data/events.jsonl`
- **Photos**: `./data/photos.jsonl`
- **MQTT Logs**: `./docker/mosquitto/log/mosquitto.log`

#### Backup Data
```bash
# Backup photos and logs
tar -czf backup-$(date +%Y%m%d).tar.gz data/

# Restore
tar -xzf backup-YYYYMMDD.tar.gz
```

#### Performance Tuning
```yaml
# docker-compose.yml - Add resource limits
services:
  dashboard:
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M
```

### Akses Dashboard
1. **Local**: http://localhost:3000
2. **Network**: http://[SERVER_IP]:3000
3. **Mobile**: Responsif untuk smartphone/tablet

### Fitur Dashboard
- ✅ Real-time event monitoring
- ✅ Live photo updates
- ✅ Photo gallery dengan thumbnails
- ✅ Connection status indicator
- ✅ Mobile-responsive design
- ✅ Auto-refresh capabilities

---

## Catatan Keamanan
⚠️ **Untuk Production Environment**:
1. Enable MQTT authentication
2. Use HTTPS/WSS untuk dashboard
3. Configure firewall rules
4. Regular backup data
5. Monitor disk usage untuk foto
6. Implement rate limiting

## Dukungan
Jika mengalami masalah, check:
1. Docker logs: `docker-compose logs [service]`
2. ESP32 Serial Monitor
3. Network connectivity
4. MQTT topic subscriptions

Sistem ini siap untuk deployment dan dapat dikustomisasi sesuai kebutuhan!
