# ⛳ EagleEye Setup Tool & Ecosystem

> **Public repository scope:** This repository contains project documentation and screenshots only. Source code, build files, and runnable binaries are not included in this checkout. Implementation details below describe the project and cannot be independently verified from this repository.

An integrated C# / .NET solution designed for controlling high-speed stereo vision sensors, broadcasting real-time ball detection telemetry via UDP, automated patch launcher operations, and managing cloud/server data resources.


## 📦 Key Application Components

### 1. **Patch Launcher (`LauncherForm`)**
* **Auto-Patching Engine:** Connects to the HTTP update server on startup to check `version_settingtool.txt`.
* **Deployment:** Automatically downloads `SettingTool.zip`, extracts assets, updates local version metadata (`versions.json`), and executes the main application.
* **Offline Fallback:** Supports offline launch or manual executable path selection if the update server is unreachable.

### 2. **EagleEye SettingTool (`ControlV`)**
* **Hardware Control:** Interface for adjusting camera parameters (Exposure, Gain, Canny Edge, Static/Dynamic Hough Thresholds) and 3-axis orientation alignment (Yaw, Pitch, Roll).
* **Communication Pipelines:** Dual connection support via RS-232 Serial (`9600-8-N-1`) and Ethernet TCP/IP.
* **Telemetry Data Processing:** Parses raw sensor UART logs (e.g., `HIT-`, `found:`, `disappear:`) in real time.
* **Server Data Management:** Archives image logs and metadata into `.zip` packages alongside user comments, uploading them directly to the remote server via HTTP Multipart API (`/upload`).

### 3. **UDP Local Simulator (`SimulatorForm`)**
* **Local Loopback Bridge:** Inter-process communication simulator operating on local UDP ports.
* **Bi-directional Protocol Testing:** Emulates third-party golf simulator applications by sending club mode commands (`DRIVER`, `IRON`, `ALL`) and listening for status events (`ready=1`, `ready=0`, `speed=XX.X`).

---

## 📡 API & Communication Protocol Specification

### 1. **Local UDP Protocol (Tool ↔ Simulator / 3rd Party App)**

Communication operates via local loopback sockets:
* **Tool Listening Port:** `7778`
* **Simulator Listening Port:** `7777`

#### **A. Inbound Commands (Sent to SettingTool on Port 7778)**

| Command | Format | Description |
| :--- | :--- | :--- |
| **Club Mode Selection** | `P_FIND_BALL:<MODE>` | Sets active tracking preset. `<MODE>` options: `DRIVER`, `IRON`, `ALL`. |
| **Force Ready (On)** | `ready=1` | Forces sensor into ready state for ball tracking. |
| **Force Ready (Off)** | `ready=0` | Cancels tracking/readiness state. |
| **Reset Command** | `coldreset` | Triggers hardware reboot sequence over serial. |

#### **B. Outbound Telemetry (Broadcast by SettingTool on Port 7777)**

| Event Key | Example Payload | Description |
| :--- | :--- | :--- |
| **Ready Status** | `ready=1` / `ready=0` | Notifies whether the ball is detected and stabilized in the ready zone. |
| **Shot Ball Speed** | `speed=68.5` | Broadcasts measured ball launch velocity in meters per second ($m/s$). |

---

### 2. **Hardware Serial Parsing Protocol (Sensor ➔ SettingTool)**

The `ControlV` module parses incoming newline-delimited strings from the camera sensor:

[Hardware Sensor] ─── (UART Line Output) ───► [ProcessReceivedLine()]


* **Ball Detection Payload:**
  * **Pattern:** `found:<X>,<Y>,<Z>(<FPS>/<Score>)`
  * **Action:** Parses 3D coordinates $(X, Y, Z)$ and sends `ready=1` over UDP if valid.
* **Impact & Shot Telemetry:**
  * **Pattern:** `HIT-SP:<Speed>,AZ:<Azimuth>,IN:<Inclination>,BS:<BackSpin>,SS:<SideSpin>,TS:<TotalSpin>,AX:<Axis>,ES:<EstFlag>`
  * **Action:** Logs detailed metrics (Velocity, Launch/Side Angles, Spin rates in RPM) and broadcasts `speed=<Speed>` to the simulator.
* **Disappearance / Miss Faults:**
  * **Pattern:** `disappear:<Reason>`
  * **Action:** Converts sensor fault codes (e.g., `Detect insufficient count`, `Measure insufficient speed`) into human-readable logs and sends `ready=0`.

---

### 3. **HTTP Server API Specification**

#### **A. Version Check Endpoint**
* **HTTP Method:** `GET`
* **URL:** `http://<ServerUrl>/version_settingtool.txt`
* **Response:** Plaintext semantic version string (e.g., `1.8.0`).

#### **B. Application Package Download**
* **HTTP Method:** `GET`
* **URL:** `http://<ServerUrl>/SettingTool.zip`
* **Response:** Archive file stream containing the compiled client application.

#### **C. Image & Log Data Upload**
* **HTTP Method:** `POST`
* **URL:** `http://<ServerUrl>/upload`
* **Content-Type:** `multipart/form-data`
* **Payload Structure:**
  * `file`: Compressed `.zip` binary stream containing captured frame images and an embedded `comment.txt` file.

---

## 🛠️ Usage with a Separately Supplied Application Package

The following workflow describes the implementation; the executables and configuration files are not included in this repository. IP addresses and server URLs are examples and must match the deployment.

### Prerequisites
* **OS:** Windows 10 / 11
* **Runtime:** .NET 8.0 Desktop Runtime
* **IDE:** Visual Studio 2022

### Running the Ecosystem

1. **Launch Sequence:**
   * Execute `Launcher.exe`.
   * The launcher checks `update_config.json` for the target server URL (default: `http://192.168.1.10:8000`).
   * If an update is detected, it handles patching automatically before launching `SettingTool.exe`.

2. **Connecting to Hardware:**
   * Open `SettingTool`.
   * Select the designated COM Port (RS-232) or specify the Ethernet IP (`192.168.1.10`) and click **연결 (Connect)**.
   * Adjust sensor gain, exposure, and alignment angles via the control sliders.

3. **Running the Simulator:**
   * Launch `SettingTool.Simulator`.
   * Test UDP state changes by clicking **DRIVER 모드 전환**, **Ready (1) 강제 송신**, or inspecting real-time telemetry logs in the terminal window.

4. **Server Data Management:**
   * Click **로그 전송 (Upload Logs)** within the `ControlV` interface.
   * Select the target image directory, enter contextual developer comments, and submit the archive to the configured upload endpoint.


<img width="1286" height="953" alt="image" src="https://github.com/user-attachments/assets/d5bb130f-3d89-45d6-a7b1-79863f34459d" />

## Integration scope

The static version-file and ZIP endpoints above describe this launcher workflow. The related [EagleEye web project](https://github.com/jelis86/EagleEye_web_public) documents a JSON release API; compatibility or migration between these update mechanisms is not established by the public documentation. Confirm the deployed launcher/server contract before integration.
