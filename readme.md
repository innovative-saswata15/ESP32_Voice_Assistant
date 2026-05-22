# ESP32 Voice Assistant (Alexa-Style Assistant)

## Overview

This project is an ESP32-S3 based Wi-Fi voice assistant system using:

* ESP32-S3
* INMP441 I2S MEMS Microphone
* Flask Python Server
* WAV Audio Recording
* Voice Activity Detection (VAD)
* Wi-Fi Audio Upload

The system records speech using the INMP441 microphone, converts it into WAV format, and uploads the audio to a local Flask server over Wi-Fi.

This project is the foundation for building:

* Alexa-like assistants
* Smart speakers
* Voice-controlled IoT systems
* Embedded AI assistants
* Speech-to-text systems

---

# Current Features

## ESP32 Side

* Wi-Fi connectivity
* INMP441 microphone support
* I2S audio capture
* Voice detection
* Automatic recording
* WAV file generation
* HTTP audio upload
* Silence timeout handling

## Server Side

* Flask upload server
* WAV file reception
* Audio file storage
* Real-time upload monitoring

---

# Hardware Used

| Component                  | Purpose             |
| -------------------------- | ------------------- |
| ESP32-S3                   | Main controller     |
| INMP441                    | I2S MEMS microphone |
| USB Cable                  | Programming + power |
| Wi-Fi Network              | Audio upload        |
| Bluetooth Speaker (future) | Audio output        |

---

# Wiring Diagram

## INMP441 → ESP32-S3

| INMP441 | ESP32-S3 |
| ------- | -------- |
| VCC     | 3.3V     |
| GND     | GND      |
| WS      | GPIO 5   |
| SCK     | GPIO 4   |
| SD      | GPIO 6   |
| L/R     | GND      |

---

# Project Architecture

```text
INMP441 Microphone
        ↓
ESP32-S3 Audio Capture
        ↓
Voice Detection
        ↓
WAV File Creation
        ↓
Wi-Fi Upload
        ↓
Flask Server
        ↓
received_audio.wav
```

---

# Folder Structure

```text
ESP32-Voice-Assistant/
│
├── README.md
├── .gitignore
├── esp32_voice_assistant.ino
│
├── server/
│   └── server.py
│
├── audio_samples/
│   └── received_audio.wav
│
└── docs/
    └── setup_guide.md
```

---

# ESP32 Arduino Code

Save the ESP32 code as:

```text
esp32_voice_assistant.ino
```

Main libraries used:

```cpp
#include <driver/i2s.h>
#include <WiFi.h>
#include <HTTPClient.h>
```

---

# Latest Working ESP32 Code

```cpp
#include <driver/i2s.h>
#include <WiFi.h>
#include <HTTPClient.h>

const char* ssid = "FASTNET 2.4 ghz @_2022";
const char* password = "SASBAI@2712";

String serverUrl = "http://192.168.0.184:5000/upload";

#define I2S_WS   5
#define I2S_SD   6
#define I2S_SCK  4

#define I2S_PORT I2S_NUM_0
#define BUFFER_LEN 256

int32_t rawBuffer[BUFFER_LEN];

#define SAMPLE_RATE 16000
#define RECORD_SECONDS 2
#define TOTAL_SAMPLES (SAMPLE_RATE * RECORD_SECONDS)

int16_t audioBuffer[TOTAL_SAMPLES];

int audioIndex = 0;

bool recording = false;

unsigned long lastVoiceTime = 0;

int voiceThreshold = 25000;

int voiceCounter = 0;

void createWavHeader(byte *header, int wavSize) {

  header[0] = 'R';
  header[1] = 'I';
  header[2] = 'F';
  header[3] = 'F';

  int fileSize = wavSize + 36;

  header[4] = (byte)(fileSize & 0xff);
  header[5] = (byte)((fileSize >> 8) & 0xff);
  header[6] = (byte)((fileSize >> 16) & 0xff);
  header[7] = (byte)((fileSize >> 24) & 0xff);

  header[8] = 'W';
  header[9] = 'A';
  header[10] = 'V';
  header[11] = 'E';

  header[12] = 'f';
  header[13] = 'm';
  header[14] = 't';
  header[15] = ' ';

  header[16] = 16;
  header[17] = 0;
  header[18] = 0;
  header[19] = 0;

  header[20] = 1;
  header[21] = 0;

  header[22] = 1;
  header[23] = 0;

  header[24] = SAMPLE_RATE & 0xff;
  header[25] = (SAMPLE_RATE >> 8) & 0xff;
  header[26] = (SAMPLE_RATE >> 16) & 0xff;
  header[27] = (SAMPLE_RATE >> 24) & 0xff;

  int byteRate = SAMPLE_RATE * 2;

  header[28] = byteRate & 0xff;
  header[29] = (byteRate >> 8) & 0xff;
  header[30] = (byteRate >> 16) & 0xff;
  header[31] = (byteRate >> 24) & 0xff;

  header[32] = 2;
  header[33] = 0;

  header[34] = 16;
  header[35] = 0;

  header[36] = 'd';
  header[37] = 'a';
  header[38] = 't';
  header[39] = 'a';

  header[40] = wavSize & 0xff;
  header[41] = (wavSize >> 8) & 0xff;
  header[42] = (wavSize >> 16) & 0xff;
  header[43] = (wavSize >> 24) & 0xff;
}

void setup() {

  Serial.begin(115200);

  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {

    delay(500);
    Serial.print(".");
  }

  Serial.println("WiFi Connected!");

  i2s_config_t i2s_config = {

    .mode = (i2s_mode_t)(I2S_MODE_MASTER | I2S_MODE_RX),
    .sample_rate = SAMPLE_RATE,
    .bits_per_sample = I2S_BITS_PER_SAMPLE_32BIT,
    .channel_format = I2S_CHANNEL_FMT_ONLY_LEFT,
    .communication_format = I2S_COMM_FORMAT_STAND_I2S,
    .intr_alloc_flags = ESP_INTR_FLAG_LEVEL1,
    .dma_buf_count = 8,
    .dma_buf_len = 64,
    .use_apll = false,
    .tx_desc_auto_clear = false,
    .fixed_mclk = 0
  };

  i2s_pin_config_t pin_config = {

    .bck_io_num = I2S_SCK,
    .ws_io_num = I2S_WS,
    .data_out_num = I2S_PIN_NO_CHANGE,
    .data_in_num = I2S_SD
  };

  i2s_driver_install(I2S_PORT, &i2s_config, 0, NULL);

  i2s_set_pin(I2S_PORT, &pin_config);

  i2s_zero_dma_buffer(I2S_PORT);

  Serial.println("System Ready");
}

void loop() {

  size_t bytesRead = 0;

  i2s_read(
    I2S_PORT,
    rawBuffer,
    sizeof(rawBuffer),
    &bytesRead,
    portMAX_DELAY
  );

  int samples = bytesRead / 4;

  long total = 0;

  for (int i = 0; i < samples; i++) {

    int32_t sample32 = rawBuffer[i];

    sample32 = sample32 >> 14;

    total += abs(sample32);
  }

  int avg = total / samples;

  Serial.print("Voice Level: ");
  Serial.println(avg);

  if (!recording) {

    if (avg > voiceThreshold) {
      voiceCounter++;
    }
    else {
      voiceCounter = 0;
    }

    if (voiceCounter >= 15) {

      recording = true;

      audioIndex = 0;

      lastVoiceTime = millis();

      Serial.println("========== RECORDING STARTED ==========");

      voiceCounter = 0;
    }
  }

  if (recording) {

    if (avg > voiceThreshold) {

      lastVoiceTime = millis();

      Serial.println("Speaking Detected");
    }
    else {

      Serial.println("Waiting for silence timeout...");
    }

    for (int i = 0; i < samples; i++) {

      int32_t sample32 = rawBuffer[i];

      sample32 = sample32 >> 14;

      if (sample32 > 32767) sample32 = 32767;
      if (sample32 < -32768) sample32 = -32768;

      int16_t sample = (int16_t)sample32;

      if (audioIndex < TOTAL_SAMPLES) {

        audioBuffer[audioIndex++] = sample;
      }
    }

    Serial.print("Recording Samples: ");
    Serial.println(audioIndex);

    if ((millis() - lastVoiceTime > 2000) ||
        (audioIndex >= TOTAL_SAMPLES)) {

      recording = false;

      Serial.println("========== RECORDING STOPPED ==========");

      int wavDataSize = audioIndex * 2;

      int totalWavSize = wavDataSize + 44;

      byte *wavFile = (byte *)malloc(totalWavSize);

      if (wavFile == NULL) {

        Serial.println("Memory Allocation Failed!");
        return;
      }

      createWavHeader(wavFile, wavDataSize);

      memcpy(
        wavFile + 44,
        (byte*)audioBuffer,
        wavDataSize
      );

      Serial.println("WAV File Created!");

      WiFiClient client;
      HTTPClient http;

      http.begin(client, serverUrl);

      http.addHeader("Content-Type", "audio/wav");

      int httpResponseCode = http.POST(
        wavFile,
        totalWavSize
      );

      Serial.print("HTTP Response Code: ");
      Serial.println(httpResponseCode);

      String response = http.getString();

      Serial.println(response);

      http.end();

      free(wavFile);

      Serial.println("WAV Upload Complete!");
    }
  }

  delay(10);
}
```

---

# Python Flask Server

Create:

```text
server/server.py
```

## Flask Server Code

```python
from flask import Flask, request

app = Flask(__name__)

@app.route("/upload", methods=["POST"])
def upload():

    audio_data = request.data

    with open("received_audio.wav", "wb") as f:
        f.write(audio_data)

    print("WAV file received!")
    print("Size:", len(audio_data))

    return "Upload Success", 200

app.run(host="0.0.0.0", port=5000)
```

---

# Installation

## 1. Install Arduino IDE Libraries

Required:

* ESP32 Board Package
* WiFi Library
* HTTPClient Library

---

## 2. Install Python Flask

Ubuntu:

```bash
pip3 install flask
```

---

# Running the Server

Open terminal:

```bash
python3 server.py
```

Expected output:

```text
Running on http://0.0.0.0:5000
```

---

# Finding Local IP

Ubuntu:

```bash
hostname -I
```

Example:

```text
192.168.0.184
```

Use this IP inside the ESP32 code:

```cpp
String serverUrl = "http://192.168.0.184:5000/upload";
```

---

# Uploading Code to ESP32

1. Open Arduino IDE
2. Select ESP32-S3 board
3. Select correct COM/USB port
4. Upload code
5. Open Serial Monitor
6. Speak near microphone

---

# Expected Serial Output

```text
WiFi Connected!
========== RECORDING STARTED ==========
Speaking Detected
Recording Samples: 16000
========== RECORDING STOPPED ==========
WAV File Created!
HTTP Response Code: 200
WAV Upload Complete!
```

---

# Expected Flask Output

```text
WAV file received!
Size: 32044
```

---

# Debugging Journey and Failure Analysis

One of the most important parts of this project was debugging the complete embedded audio pipeline.

This section documents the major failures encountered and the engineering fixes used.

---

## 1. Trackpad-Style Noise Values / Random Readings

### Problem

Initial microphone readings produced:

```text
7611
10697
8220
5651
```

with unstable random values.

### Cause

Raw 32-bit I2S audio was being read directly without proper scaling.

### Fix

Used bit shifting:

```cpp
sample = rawBuffer[i] >> 14;
```

This converted 32-bit audio into usable signed values.

---

## 2. False Voice Detection

### Problem

Silence was triggering recording automatically.

Example:

```text
Voice Level: 77003
```

while room was silent.

### Cause

Changing audio scaling changed signal amplitude dramatically.

### Fix

Adjusted:

```cpp
voiceThreshold
```

multiple times during debugging.

Example tested values:

```cpp
22000
120000
25000
```

Also improved stability using:

```cpp
if (voiceCounter >= 15)
```

instead of smaller trigger counts.

---

## 3. HTTPS Upload Failure

### Problem

ESP32 produced:

```text
HTTP Response Code: -5
```

when uploading to webhook.site.

### Cause

ESP32 had TLS/SSL handshake problems with HTTPS.

### Fix

Changed:

```text
https://
```

to:

```text
http://
```

for lightweight testing.

---

## 4. Large WAV Upload Failure

### Problem

ESP32 produced:

```text
HTTP Response Code: -3
```

while uploading WAV audio.

### Cause

WAV payload size was too large for ESP32 HTTP memory handling.

5-second recordings generated nearly 100 KB uploads.

### Fixes

Reduced RAM pressure:

```cpp
#define RECORD_SECONDS 1
#define BUFFER_LEN 256
```

This stabilized binary uploads.

---

## 5. Webhook.site Binary Upload Issues

### Problem

Webhook.site was unreliable for large binary WAV uploads.

### Cause

Webhook.site is optimized for webhooks and debugging, not embedded binary audio streaming.

### Fix

Built a local Flask server:

```python
@app.route("/upload", methods=["POST"])
```

This created a reliable local audio upload system.

---

## 6. Noise / Beep Audio Instead of Voice

### Problem

Recorded WAV file contained:

* static
* robotic sound
* beep-like audio

instead of understandable speech.

### Cause

INMP441 outputs 24-bit signed audio inside 32-bit I2S frames.

Incorrect bit alignment caused clipping and distortion.

### Debugging Steps Tested

Tested multiple bit shifts:

```cpp
>> 14
>> 11
>> 8
```

Each produced different audio behavior.

### Final Working Direction

Used safer signed conversion:

```cpp
int32_t sample32 = rawBuffer[i];
sample32 = sample32 >> 14;
```

plus clipping protection:

```cpp
if (sample32 > 32767) sample32 = 32767;
if (sample32 < -32768) sample32 = -32768;
```

This produced recognizable speech rhythm.

---

## 7. DMA Garbage Audio

### Problem

Initial recording sometimes contained random garbage/noise.

### Cause

ESP32 DMA buffer was not cleared.

### Fix

Added:

```cpp
i2s_zero_dma_buffer(I2S_PORT);
```

This stabilized startup audio.

---

## 8. Double Upload Triggering

### Problem

Server received:

```text
WAV file received!
```

twice.

### Cause

Residual sound/noise retriggered recording immediately after upload.

### Planned Future Fix

Future improvements:

* cooldown timer
* better VAD
* wake-word activation

---

## Engineering Lessons Learned

This debugging process taught:

* I2S digital audio handling
* ESP32 memory limitations
* WAV binary formatting
* TCP/HTTP upload behavior
* Digital audio scaling
* Signed PCM conversion
* Embedded networking
* Voice activity detection
* Real-time embedded debugging

---

# Current Limitations

* Audio still needs DSP optimization
* Noise filtering not implemented yet
* No wake word support yet
* No speech-to-text integration yet
* No Bluetooth speaker playback yet

---

# Upcoming Features

## Planned Improvements

* Better audio quality
* DSP filtering
* Noise suppression
* Wake word detection
* Whisper speech-to-text
* ChatGPT integration
* Bluetooth speaker response
* Real conversational assistant

---

# Future AI Pipeline

```text
Voice
 ↓
ESP32 Recording
 ↓
WAV Upload
 ↓
Whisper Speech-to-Text
 ↓
ChatGPT Processing
 ↓
Text Response
 ↓
Text-to-Speech
 ↓
Bluetooth Speaker Output
```

---

# Recommended Tools

| Tool        | Purpose             |
| ----------- | ------------------- |
| Arduino IDE | ESP32 programming   |
| Flask       | Audio upload server |
| VLC         | Audio playback      |
| Audacity    | Audio debugging     |
| GitHub      | Version control     |

---

# Git Commands

## Initialize Repo

```bash
git init
```

## Add Files

```bash
git add .
```

## Commit

```bash
git commit -m "Initial ESP32 Voice Assistant"
```

## Create GitHub Repo

```bash
git remote add origin YOUR_REPO_URL
```

## Push

```bash
git push -u origin main
```

---

# Suggested .gitignore

```gitignore
*.wav
*.pyc
__pycache__/
.vscode/
build/
```

---

# Learning Outcomes

This project teaches:

* Embedded systems
* ESP32 programming
* I2S digital audio
* MEMS microphones
* Wi-Fi networking
* HTTP communication
* WAV file structure
* Flask server development
* Voice detection systems
* Embedded AI architecture

---

# Project Status

## Current Status: WORKING

✅ Wi-Fi Connected

✅ Audio Recording

✅ Voice Detection

✅ WAV Generation

✅ Flask Upload

✅ Real Audio Transfer

---

# Author

Built by Saswata using ESP32-S3 + INMP441.

---

# License

MIT License
