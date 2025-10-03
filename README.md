# 🔊 Surround-Sound: A Distributed Synchronous Audio System

This document outlines the implementation plan for the **Surround-Sound** project, a system designed to transform a group of mobile devices and a host computer into a single, cohesive, high-volume, and synchronized pseudo-surround sound system over a local Wi-Fi network.

## 🎯 Project Goal

*   **Purpose**: To address the limitation of low-power, built-in laptop speakers for group viewing experiences by creating a powerful, distributed audio system.
*   **Target User**: Friends, families, or small groups watching movies, presentations, or listening to music together where the primary audio source (e.g., a laptop) is inadequate.

## ✨ Key Features

-   **Perfect Synchronization**: Zero perceptible echo or lag between any connected device.
-   **Simple Setup**: Quick connection via QR code scan opening a web client in the browser. **No app installation required.**
-   **Host Control**: The laptop (server) controls the audio source, playback, and master volume.
-   **Individual Volume/Channel Control**: Client devices can adjust their individual volume or be assigned a specific audio channel (e.g., Left, Right, Center) for a true surround effect.

## 🛠️ Technical Implementation Plan

The core engineering challenge is achieving Inter-Destination Synchronization (IDS)—ensuring audio starts and stays perfectly aligned across multiple devices with different internal clocks and network paths.

### 1. System Architecture: Client-Server Model

The system will use a Client-Server architecture over the Local Area Network (LAN).

| Component | Device                       | Role                                                                                             | Technologies                                                                                             |
| :-------- | :--------------------------- | :----------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------- |
| **Server**  | Laptop (Windows/macOS)       | Runs a Python script to serve the web client, capture system audio, and stream it to all clients. | **Python**, **FastAPI/Flask**, **WebSockets**, **Soundcard**, **NumPy**, **Opuslib**.                        |
| **Client**  | Any device with a Web Browser | Distributed Speaker. Renders UI, receives audio stream, synchronizes, and plays audio via the browser. | **HTML5**, **CSS**, **JavaScript**, **Web Audio API**, **WebSockets**.                                       |

### 2. Detailed Tech Stack

A developer building this project must be proficient with the following technologies:

#### Backend (Server - Python)
*   **Web Framework**: **FastAPI** or **Flask**. Required to serve the HTML/CSS/JS client files and to handle WebSocket connections for control and synchronization.
*   **WebSocket Library**: `websockets` or the native support in **FastAPI**. Crucial for real-time, bidirectional communication with clients for time sync, channel assignment, and status updates.
*   **Audio Capture**: **`soundcard`**. A cross-platform library to capture system audio loopback (what you hear from your speakers). `PyAudio` can be used for microphone input during the proof-of-concept phase.
*   **Audio Processing & Encoding**:
    *   **`NumPy`**: For efficient handling of raw audio data as numerical arrays.
    *   **`opuslib`**: Python bindings for the **Opus** codec, essential for low-latency, high-quality audio compression before streaming.
*   **QR Code Generation**: **`qrcode`**. To generate the QR code containing the server's local IP address and port for easy client connection.
*   **Concurrency**: **`asyncio`**. Essential for managing multiple concurrent client connections (WebSockets, audio streaming) without blocking.

#### Frontend (Client - Web)
*   **Core Web Technologies**: **HTML5**, **CSS3**, **JavaScript (ES6+)**. For building the client-side user interface.
*   **Web Audio API**: This is the **most critical** client-side technology. It provides the necessary tools for low-latency audio playback, creating a jitter buffer, and scheduling audio chunks to play at a precise, synchronized time.
*   **WebSockets API**: The browser's native API for establishing a WebSocket connection with the server to receive control messages and audio data.
*   **UI Framework (Optional)**: A lightweight framework like **Vue.js** or **Svelte** could simplify UI development for channel selection and volume controls, but is not strictly necessary for a minimal product.

### 2. Core Technology: Low-Latency Synchronization

A playback time difference of more than 5-20 milliseconds is perceivable as an echo. The following mechanisms will be used to achieve synchronization.

#### A. Time Synchronization Protocol

A simplified version of the Network Time Protocol (NTP) will be implemented to align all client clocks with the server's master clock.

*   **Mechanism**: The Server acts as the time reference.
*   **Process**:
    1.  A Client periodically sends a time request packet to the Server.
    2.  The Server timestamps the packet upon receipt (`t2`) and again just before sending a response (`t3`).
    3.  The Client uses its original send time (`t1`) and final receipt time (`t4`), along with the server's timestamps, to calculate the network offset and synchronize its internal 'playback clock' to the server's time.

#### B. Stream Transmission and Playback

*   **Audio Capture & Encoding**: The Server captures the laptop's audio output. The audio is encoded using a low-latency codec like **Opus** or **AAC**.
*   **Streaming Protocol**: **WebSockets** will be used for both control and audio data. While UDP is traditionally lower latency, using WebSockets simplifies the architecture significantly (no need for a separate UDP path, better firewall traversal). Modern WebSockets can achieve very low latency on a local Wi-Fi network, and sending audio data in binary format is efficient.
*   **Timestamping**: The Server will embed a high-precision, synchronized timestamp in the header of every audio packet. This timestamp indicates the exact future moment the packet should be played.
*   **Client Jitter Buffer**: Each Client's JavaScript code will maintain a small Jitter Buffer (e.g., 50-100 ms). This buffer stores incoming audio packets and is sized to be slightly longer than the worst-case network latency.
*   **Synchronized Playback**: Using the **Web Audio API**, the client will schedule audio packets from the buffer to be played only when its local synchronized time matches the timestamp on the packet. This ensures that even if a packet arrives late due to network jitter, it is played at the correct moment, keeping all devices perfectly in sync.

### 3. Advanced Surround Capability

To create a true surround system, the Server will perform Channel Mixing and Routing.

1.  The Server must be able to read multi-channel audio sources (e.g., 5.1 or 7.1 movie tracks).
2.  The web client interface will allow the user to select a channel for their device (e.g., Left, Right, Center).
3.  The Server will extract the audio data for the assigned channel, encode it into a dedicated stream, and send it to the corresponding phone with the appropriate synchronization timestamp.

## 🚀 Development Roadmap

The project will be developed in three distinct phases.

### Phase 1: Proof of Concept (PC-to-PC, Basic Sync)

1.  **Develop Server**: Create a minimal Python server that captures microphone input and streams it via UDP with millisecond-accurate timestamps.
2.  **Develop Simple Client**: Create a second Python script that connects to the server, implements the NTP-like clock sync, receives the UDP stream, and plays back the audio using a jitter buffer based on packet timestamps.
3.  **Test and Tune**: Run the Server and Client on two PCs on the same Wi-Fi network. Measure latency and adjust the jitter buffer until any echo is eliminated to establish a baseline time tolerance.

### Phase 2: Web Client Implementation

1.  **Web Server & QR Code**: Enhance the Python server using **FastAPI/Flask** to serve a basic HTML/JS page. Implement logic to find the server's local IP, start the server, and display a **QR code** in the terminal or a simple GUI window that points to `http://<server-ip>:<port>`.
2.  **Web Client Development**: Create the client-side application using HTML, CSS, and JavaScript.
    *   Implement the time synchronization logic over a **WebSocket** connection.
    *   Use the **Web Audio API** to receive binary audio data, manage the jitter buffer, and schedule synchronized playback.
3.  **User Interface (UI)**: Build the web interface with controls for channel selection (Left, Right, Center) and individual volume.

### Phase 3: Feature Refinement and Distribution

1.  **Audio Interception**: Upgrade the Server to capture the system's internal audio output (e.g., movie sound) instead of just microphone input. This will require platform-specific audio capture libraries.
2.  **Multi-Channel Feature**: Implement the channel routing logic on the server and the channel selection UI on the clients to enable the full surround sound capability.
3.  **Packaging**: Bundle the Python server application and all its web assets (HTML, CSS, JS) into a single executable file for Windows and macOS using a tool like **PyInstaller** or **cx_Freeze** for easy distribution.
