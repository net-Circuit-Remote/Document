# KIẾN TRÚC TỔNG THỂ DỰ ÁN net*CIRCUIT Remote

## 1. Tổng quan

**net*CIRCUIT Remote** là một nền tảng **Remote Digital Electronics Laboratory** theo mô hình **Web-CAD/Workbench**, cho phép người dùng thiết kế, mô phỏng, điều khiển và quan sát các thí nghiệm điện tử số từ trình duyệt.

Định hướng kiến trúc chính:

- **Circuit Graph** là mô hình trung gian và nguồn sự thật duy nhất của thiết kế.
- Phát triển **Virtual Hardware trước**, hoàn thiện toàn bộ workflow phần mềm.
- Sau đó thay `VirtualHardwareStation` bằng `PhysicalHardwareStation` mà không cần viết lại frontend.
- Sử dụng **Mini PC** làm Edge Server/Controller.
- FPGA chịu trách nhiệm cho các tác vụ **deterministic / timing-critical**.
- Routing Hardware kết nối FPGA với các IC logic thật.
- Hệ thống hướng đến khả năng phục vụ nhiều người dùng, mở rộng nhiều station và phát triển thành nền tảng phòng thí nghiệm số hoàn chỉnh.

### Cấu hình Edge Server hiện tại

- CPU: **Intel Xeon W-1370**
- RAM: **16 GB**
- SSD: **256 GB**
- Vai trò:
  - Web Server
  - Backend API
  - Simulator
  - Experiment Manager
  - Database
  - Hardware Service
  - FPGA communication
  - Waveform processing
  - Session / lease / queue management

> **Lưu ý:** mô hình 3D được render trên trình duyệt của từng người dùng bằng WebGL/Three.js. Mini PC không render 3D thay cho tất cả người dùng, trừ khi nó đồng thời được dùng như một máy trạm trực tiếp.

---

# 2. Kiến trúc tổng thể được lựa chọn

Kiến trúc đề xuất:

> **Web SPA + Modular Monolith Backend + Isolated Hardware Service + Contract-Driven FPGA**

Không sử dụng full microservices ở giai đoạn hiện tại.

```text
┌───────────────────────────────────────────────────────────────┐
│                         USER BROWSER                          │
│                                                               │
│  Vue 3 + TypeScript + Three.js                               │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐ │
│  │ Workbench UI │  │ CircuitGraph │  │ Three.js Scene      │ │
│  │ Instruments  │  │ Editor       │  │ Renderer / Picking  │ │
│  │ Properties   │  │ Undo/Redo    │  │ Gizmo / Wires       │ │
│  └──────┬───────┘  └──────────────┘  └─────────────────────┘ │
│         │                                                     │
└─────────┼─────────────────────────────────────────────────────┘
          │
          │ HTTPS REST
          │ WebSocket
          ▼
┌──────────────────────── MINI PC ───────────────────────────────┐
│ Xeon W-1370 / 16 GB RAM / SSD                                │
│                                                               │
│                     Reverse Proxy                             │
│                         Nginx                                 │
│                           │                                   │
│           ┌───────────────┴───────────────┐                   │
│           ▼                               ▼                   │
│     Static Web Files                  FastAPI                 │
│                                       Backend                 │
│                                           │                   │
│     ┌─────────────────────────────────────┼────────────────┐  │
│     │                                     │                │  │
│     ▼                                     ▼                ▼  │
│ Circuit Domain                    Experiment Manager      Auth │
│ Validation                       Session / Lease          DB   │
│ Compiler                             │                         │
│     │                                │                         │
│     ├─────────────┐                  │                         │
│     ▼             ▼                  │                         │
│ Virtual        Physical              │                         │
│ Compiler       Compiler              │                         │
│     │             │                  │                         │
│     ▼             │                  │                         │
│ Simulator         │                  │                         │
│                   ▼                  │                         │
│             ExecutionPlan            │                         │
│                   │                  │                         │
│                   └─────────┬────────┘                         │
│                             ▼                                  │
│                       Hardware Service                          │
│                       separate process                         │
│                             │                                  │
│                      gRPC / Protobuf                           │
│                             │                                  │
│                      FPGA Host Driver                          │
│                             │                                  │
└─────────────────────────────┼──────────────────────────────────┘
                              │
                     USB / USB3 / Ethernet
                              │
                              ▼
┌───────────────────────────────────────────────────────────────┐
│                            FPGA                               │
│                                                               │
│ Control Engine                                                │
│ Clock / Trigger                                               │
│ Routing Controller                                            │
│ Logic Analyzer                                                │
│ DDS / Generator                                               │
│ Acquisition                                                   │
│ SDRAM Controller                                              │
└───────────────────────┬───────────────────────────────────────┘
                        │
                        ▼
               MUX / CROSSPOINT MATRIX
                        │
                        ▼
               PHYSICAL LOGIC ICs
                        │
                        ▼
                 REAL EXPERIMENT
```

---

# 3. So sánh các phương án kiến trúc Backend

## 3.1 Monolith duy nhất

**Ưu điểm**
- Đơn giản.
- Ít process.
- Dễ triển khai.

**Nhược điểm**
- Hardware driver lỗi có thể kéo toàn hệ thống xuống.
- Khó cách ly lỗi thiết bị.
- Khó bảo trì khi hardware subsystem ngày càng phức tạp.

## 3.2 Full Microservices

**Ưu điểm**
- Dễ scale độc lập.
- Phù hợp hệ thống lớn.

**Nhược điểm**
- Tăng mạnh độ phức tạp:
  - service discovery
  - distributed logging
  - distributed transaction
  - message broker
  - network failure handling
  - container orchestration
- Không phù hợp với một hệ thống ban đầu chạy trên một Mini PC.

## 3.3 Modular Monolith + Hardware Service

Đây là phương án được lựa chọn.

```text
netcircuit-api
│
├── auth
├── projects
├── circuits
├── simulation
├── experiments
├── stations
├── captures
└── persistence

hardware-service
│
├── station abstraction
├── physical driver
├── transport
├── FPGA protocol
└── safety controller
```

**Lợi ích**
- Giữ phần mềm đơn giản.
- Vẫn tách biệt hardware khỏi application backend.
- Có thể phát triển thành microservices sau này nếu thật sự cần.

---

# 4. Hardware Service phải là process riêng

Hardware có các dạng lỗi đặc thù:

```text
USB disconnect
FPGA timeout
SPI error
CRC error
driver hang
FPGA reset
device unavailable
```

Không nên để các lỗi này làm chết:

```text
Web
API
Database
Virtual simulation
Circuit editing
```

Do đó:

```text
FastAPI
   │
   │ gRPC
   ▼
Hardware Service
   │
   ▼
FPGA Driver
   │
   ▼
FPGA
```

Nếu Hardware Service gặp lỗi thì Web/API/Database/Virtual Simulation vẫn phải tiếp tục hoạt động.

---

# 5. Circuit Graph là trung tâm của toàn hệ thống

```text
                    CircuitGraph
                         │
           ┌─────────────┼──────────────┐
           │             │              │
           ▼             ▼              ▼
       Three.js       Simulator     Hardware Compiler
```

Circuit Graph không phụ thuộc:

- tọa độ mesh 3D;
- FPGA register;
- MUX channel;
- SPI packet;
- crosspoint routing.

Ví dụ:

```text
SW1.OUT → AND1.A
AND1.Y  → LED1.IN
```

Frontend chỉ biết logical connection.

---

# 6. Ba tầng dữ liệu Circuit

```text
DesignGraph
     ↓
ValidatedGraph
     ↓
ExecutionPlan
```

## 6.1 DesignGraph

Là mô hình do người dùng chỉnh sửa.

Bao gồm:

- Component
- Port
- Connection
- Property
- Position
- Rotation
- Label
- Metadata

Dùng cho Save/Open/Undo/Redo/3D Render.

## 6.2 ValidatedGraph

Backend chuẩn hóa và kiểm tra:

```text
resolve component
resolve port
resolve net
check direction
check width
check drivers
check power domain
check electrical constraints
```

Ví dụ:

```text
SW1.OUT ────── Net0001 ───── AND1.A
```

## 6.3 ExecutionPlan

Compiler sinh ra tùy target:

```text
VirtualExecutionPlan
PhysicalExecutionPlan
```

Ví dụ Physical Execution Plan:

```text
Net0001

Source:
SW bank channel 4

Destination:
74HC08 socket pin 1

Routing:
Crosspoint X17 → Y05
```

ExecutionPlan là dữ liệu nội bộ backend/hardware. Frontend không cần biết.

---

# 7. Phân chia Domain

## 7.1 Circuit Domain

```text
Circuit Domain
│
├── Project
├── Circuit
├── ComponentInstance
├── ComponentDefinition
├── Port
├── Net
├── Property
└── CircuitMetadata
```

## 7.2 Rendering Domain

```text
Rendering Domain
│
├── SceneObject
├── Transform
├── Mesh
├── Material
└── Camera
```

## 7.3 Physical Domain

```text
Physical Domain
│
├── Station
├── HardwareProfile
├── RoutingResource
├── Socket
├── Crosspoint
├── FPGAResource
├── InstrumentChannel
└── CalibrationProfile
```

Không trộn lẫn ba domain này.

---

# 8. Component Library

Mỗi component definition nên chứa:

```text
identity
display metadata
ports
logic model
electrical constraints
3D asset reference
physical package
pin mapping
simulation model
```

Ví dụ:

```text
74HC08
│
├── logical model
├── DIP14 package
├── pins
├── power pins
├── simulation behavior
└── GLB model
```

Routing channel không nằm trong component definition; routing thuộc Station / Hardware Profile.

---

# 9. Kiến trúc Simulator

Đối với điện tử số, lựa chọn:

> **Discrete Event Simulation**

Không dùng fixed timestep cho toàn bộ hệ thống.

```text
t = 0 ns
SW1 changes 0 → 1
        ↓
event
AND1.A = 1
        ↓
propagation delay = 8 ns
        ↓
t = 8 ns
AND1.Y recalculated
        ↓
event
LED1 = ON
```

**Lợi ích**
- Không tính toán khi trạng thái không thay đổi.
- Hiệu quả CPU.
- Dễ mô hình hóa propagation delay.
- Phù hợp nhiều user đồng thời.

---

# 10. Simulator chạy ở đâu?

## Browser-side
Ưu điểm:
- tương tác nhanh;
- giảm tải server.

Nhược điểm:
- khó đồng bộ implementation;
- logic simulation nằm trên client;
- khó đảm bảo reproducibility.

## Server-side
Ưu điểm:
- một implementation duy nhất;
- dễ quản lý version;
- phù hợp Remote Lab;
- dễ đồng bộ Virtual và Physical workflow.

Nhược điểm:
- phụ thuộc network latency.

## Rust/WASM Shared Engine
Ưu điểm:
- native server + WASM browser dùng chung engine.

Nhược điểm:
- tăng độ phức tạp;
- chưa cần thiết.

### Lựa chọn

> **Server-side Python Discrete Event Simulator**

Chỉ cân nhắc Rust/WASM khi Python simulator trở thành bottleneck thực tế.

---

# 11. Frontend Framework

Tiếp tục:

```text
Vue 3
TypeScript
Vite
Pinia
Three.js
```

| Framework | Ưu điểm | Nhược điểm |
|---|---|---|
| React | Ecosystem lớn | Không đủ lý do để rewrite |
| Angular | Structure mạnh | Nặng, nhiều boilerplate |
| Svelte | Nhẹ | CAD ecosystem nhỏ hơn |
| **Vue 3** | Cân bằng tốt | Phù hợp hệ thống hiện tại |
| Electron | Desktop | Không phù hợp mục tiêu Web Remote Lab |

**Lựa chọn:** Vue 3 + TypeScript.

---

# 12. Three.js hay Babylon.js?

## Babylon.js
Mạnh về:
- game engine;
- physics;
- GUI;
- asset loader;
- debugging.

Nhưng tương đối nặng cho CAD/workbench kỹ thuật.

## Three.js
Cho phép tự kiểm soát:

```text
SceneManager
WireLayer
Picking
Transform Gizmo
Selection
Asset Manager
```

### Lựa chọn

> **Tiếp tục Three.js**

Không có lý do kỹ thuật đủ lớn để rewrite sang Babylon.js.

---

# 13. Asset Pipeline cho mô hình 3D

```text
CAD / Blender
      ↓
GLB
      ↓
Mesh optimization
      ↓
Meshopt
      ↓
KTX2 textures
      ↓
Browser
```

Ưu tiên:

- GLTF / GLB
- Meshopt
- KTX2 / Basis
- Asset caching
- Lazy loading
- Level of Detail

---

# 14. Tối ưu Three.js

## Instancing

Breadboard có nhiều lỗ giống nhau không nên tạo hàng trăm Mesh độc lập.

```text
1 geometry
+
InstancedMesh
```

## Asset caching

20 IC cùng loại chỉ tải model một lần:

```text
Asset
  ↓
cache
  ↓
clone / instance
```

## Lazy loading

Không tải toàn bộ model/instrument khi người dùng chưa sử dụng.

## Level of Detail

```text
Zoom gần → high-poly
Zoom xa  → low-poly
```

---

# 15. State Management

Pinia quản lý:

```text
UI state
selection
tool state
project state
preferences
```

Không dùng Pinia làm simulator/database/hardware state machine.

Phân chia:

```text
UI State
    Pinia

Circuit Domain
    Circuit Graph Engine

Renderer State
    SceneManager

Remote State
    API / WebSocket Client

Hardware State
    Server Authoritative
```

---

# 16. Undo / Redo

Giai đoạn hiện tại:
- Snapshot history vẫn phù hợp.
- Giới hạn số lượng và dung lượng snapshot.

Khi circuit lớn hơn có thể chuyển sang:

```text
MoveComponentCommand
AddWireCommand
DeleteComponentCommand
ChangePropertyCommand
```

Không cần rewrite ngay.

---

# 17. Backend Framework

Lựa chọn:

> **FastAPI**

Lý do:
- phù hợp simulation;
- waveform/data processing;
- FPGA/hardware tooling;
- Python ecosystem mạnh;
- Xeon W-1370 đủ khả năng xử lý.

Không có lý do kỹ thuật đủ lớn để chuyển sang Node/NestJS.

---

# 18. Hardware Service Language

Lựa chọn:

> **Python Hardware Service cho V1**

Host không thực hiện hard-real-time; FPGA chịu trách nhiệm timing-critical.

Nếu data path trở thành bottleneck:

```text
Hardware Service
      ↓
Rust/C++ Native Driver
```

mà không cần thay kiến trúc phía trên.

---

# 19. FastAPI ↔ Hardware Service

Lựa chọn:

> **gRPC + Protobuf**

RPC điển hình:

```text
GetCapabilities
Reserve
Validate
ApplyCircuit
SetInput
ConfigureClock
ConfigureGenerator
ConfigureTrigger
ArmCapture
Run
Stop
ReadCapture
SafeState
Reset
```

---

# 20. Browser ↔ Backend

Sử dụng:

```text
REST
+
WebSocket
```

## REST
Dùng cho:

```text
login
projects
circuits
save
open
station information
experiment creation
configuration
```

## WebSocket
Dùng cho:

```text
simulation events
LED state
switch state
experiment state
logic analyzer
oscilloscope capture
hardware telemetry
```

---

# 21. Waveform Transport

Không truyền waveform lớn bằng JSON.

Sử dụng:

> **WebSocket Binary Frame**

Header:

```text
capture_id
channel
sample_rate
sample_count
timestamp
encoding
```

Payload:

```text
Int16
Uint16
Float32
```

tùy loại instrument.

---

# 22. Oscilloscope Architecture

Không stream raw ADC liên tục vào browser.

```text
ADC
 ↓
FPGA
 ↓
Trigger
 ↓
Capture Buffer
 ↓
SDRAM
 ↓
Decimation / Window
 ↓
Mini PC
 ↓
Browser
```

Browser chỉ nhận số sample cần thiết cho viewport hiện tại.

---

# 23. Logic Analyzer Architecture

```text
Digital Inputs
     ↓
FPGA
     ↓
Trigger
     ↓
Sampling
     ↓
SDRAM
     ↓
Compression
     ↓
Mini PC
     ↓
Browser
```

Digital waveform có thể dùng RLE để giảm dữ liệu.

---

# 24. Phân chia trách nhiệm Mini PC và FPGA

## Mini PC

```text
Web Server
API
Database
Circuit Compiler
Simulator
Routing Algorithm
Experiment Lifecycle
User / Session Management
Waveform Processing
Storage
```

## FPGA

```text
Clock Generation
Precise Switching
Timestamping
Logic Capture
Trigger
DDS
Routing Control
Digital IO
ADC Interface
SDRAM Controller
```

---

# 25. PC Mini ↔ FPGA

Mini PC không có GPIO/SPI tích hợp như Raspberry Pi, do đó phải dùng transport phù hợp.

## USB-UART
- rẻ;
- dễ debug;
- throughput thấp;
- phù hợp prototype.

## FT2232H / USB-SPI
- nhanh hơn UART;
- hỗ trợ JTAG/SPI;
- phù hợp command/register.

## USB 3 FIFO
- throughput cao;
- phù hợp waveform, Logic Analyzer, Oscilloscope.

## Ethernet
- isolation tốt;
- station có thể đặt xa;
- FPGA network stack phức tạp hơn.

## PCIe
- hiệu năng cao nhất;
- PCB/driver/DMA/FPGA IP phức tạp;
- không phù hợp V1.

### Lựa chọn

Prototype:

```text
USB-UART hoặc USB-SPI
```

Final high-throughput:

```text
USB 3 FIFO
```

---

# 26. FPGA Transport Abstraction

```text
FpgaTransport
│
├── SerialTransport
├── SpiUsbTransport
├── Usb3Transport
└── EthernetTransport
```

Application không phụ thuộc transport cụ thể.

---

# 27. FPGA Protocol

Không cho PC ghi register tùy ý.

Protocol:

```text
SOF
Version
Command
Sequence
Length
Payload
CRC
EOF
```

Command ví dụ:

```text
GET_CAPABILITY
SET_CLOCK
SET_ROUTE
WRITE_OUTPUT
ARM_CAPTURE
START_CAPTURE
READ_CAPTURE
RESET
SAFE_STATE
```

Mỗi command có sequence, timeout, response và error code.

---

# 28. Capability Discovery

Không hard-code khả năng station trên frontend.

Station trả:

```text
StationCapabilities
```

Ví dụ:

```text
digitalInputs: 32
digitalOutputs: 32
logicAnalyzerChannels: 16

oscilloscope:
  channels: 2
  maxSampleRate: ...

generator:
  channels: 2

routing:
  ...
```

Frontend bật/tắt chức năng dựa trên capability.

---

# 29. Physical Compiler

```text
Circuit Graph
      ↓
Physical Compiler
      ↓
Routing Graph
      ↓
Routing Solver
      ↓
Routing Plan
      ↓
FPGA
```

Compiler ánh xạ logical port → physical socket/resource.

---

# 30. Routing Solver

Không cần AI.

Bản chất là:

> **Graph Routing / Constrained Path Finding**

Có thể dùng:

```text
BFS
Dijkstra
Backtracking
Resource Reservation
```

Mục tiêu:

```text
shortest valid route
+
no resource conflict
+
electrical constraints
```

---

# 31. Electrical Rule Checker — ERC

Trước khi chạy hardware thật:

```text
Circuit
  ↓
ERC
```

Kiểm tra:

```text
output ↔ output short
VCC ↔ GND
multiple drivers
invalid voltage
floating required pins
power missing
unsupported IC
unsupported connection
fan-out
routing impossible
```

Nếu lỗi:

```text
Reject Experiment
```

Không gửi xuống FPGA.

---

# 32. Hardware Safety State Machine

```text
OFFLINE
   ↓
IDLE
   ↓
RESERVED
   ↓
CONFIGURING
   ↓
READY
   ↓
ARMED
   ↓
RUNNING
   ↓
STOPPING
   ↓
IDLE
```

Lỗi:

```text
FAULT
  ↓
SAFE_STATE
```

SAFE_STATE:

```text
disable routing
disable outputs
stop clocks
disable generator
tri-state FPGA outputs
```

---

# 33. Multi-user Experiment Lease

Virtual:

```text
50 users
→ 50 virtual sessions
```

Physical:

```text
1 station
→ 1 experiment owner
```

Backend quản lý:

- reservation;
- lease;
- timeout;
- queue;
- ownership;
- recovery.

---

# 34. Message Broker

Giai đoạn đầu chưa cần:

```text
Kafka
RabbitMQ
NATS
Redis Cluster
```

Một Mini PC có thể dùng:

```text
PostgreSQL
+
Experiment Manager
```

Chỉ thêm broker khi có nhiều server/station và có nhu cầu thực tế.

---

# 35. Database

Prototype có thể giữ SQLite.

Production/multi-user nên dùng:

> **PostgreSQL**

Lưu:

```text
users
projects
circuits
experiments
station leases
queues
capture metadata
audit logs
```

---

# 36. Waveform Storage

Không lưu waveform lớn trực tiếp trong PostgreSQL.

Database chỉ lưu metadata:

```text
capture_id
experiment_id
channel
sample_rate
timestamp
file_path
format
```

Waveform lưu trên filesystem:

```text
/data/captures/
```

Sau này có thể thay bằng object storage.

---

# 37. SSD Capacity

SSD 256 GB đủ cho:
- OS;
- backend;
- database;
- source;
- 3D assets;
- logs cơ bản.

Khi lưu nhiều capture, nên nâng:

> **512 GB hoặc 1 TB NVMe**

---

# 38. Authentication

Giai đoạn đầu:

```text
username/password
server-side session
HTTP-only secure cookie
RBAC
```

Role:

```text
Student
Instructor
Administrator
```

Sau này có thể tích hợp OIDC/SSO.

---

# 39. Deployment trên Mini PC

Không dùng Kubernetes.

Stack:

```text
Linux
Nginx
FastAPI
Hardware Service
PostgreSQL
systemd
```

Services:

```text
nginx.service
netcircuit-api.service
netcircuit-hardware.service
postgresql.service
```

Docker Compose có thể dùng cho development/CI/test, không bắt buộc production.

---

# 40. Vì sao không Kubernetes?

Hệ thống ban đầu chỉ có một Mini PC.

Kubernetes tạo thêm:

```text
container orchestration
network overlay
ingress
service discovery
persistent volumes
Helm
```

nhưng chưa giải quyết bài toán thực tế tương xứng với độ phức tạp.

---

# 41. Đánh giá Mini PC Xeon W-1370

Phù hợp với:

```text
FastAPI
PostgreSQL
Discrete Event Simulator
Circuit Compiler
Routing Solver
Hardware Service
Waveform Processing
30–50 WebSocket Connections
```

RAM 16 GB phù hợp V1.

Có thể nâng 32 GB nếu cần nhiều simulation session, build/IDE và xử lý waveform lớn trên cùng máy.

---

# 42. Render 3D thực tế chạy ở đâu?

```text
Laptop A GPU → User A
Laptop B GPU → User B
Laptop C GPU → User C
```

Mini PC chủ yếu phục vụ:

```text
HTML
JavaScript
GLB Assets
API Data
WebSocket Data
```

Do đó tải render 3D chủ yếu nằm ở client.

---

# 43. Component Architecture chính thức

```text
net-Circuit-Remote
│
├── Web Application
│   ├── Workspace Shell
│   ├── Circuit Editor
│   ├── Component Library UI
│   ├── 3D Renderer
│   ├── Instrument UI
│   ├── Project Manager
│   └── API Client
│
├── Circuit Domain
│   ├── DesignGraph
│   ├── Validation
│   ├── Net Resolver
│   └── Serialization
│
├── Backend
│   ├── Auth
│   ├── Project Service
│   ├── Circuit Service
│   ├── Experiment Manager
│   ├── Station Manager
│   ├── Capture Manager
│   └── WebSocket Gateway
│
├── Simulation
│   ├── Event Engine
│   ├── Device Models
│   ├── Clock
│   ├── Waveforms
│   └── Virtual Instruments
│
├── Physical Compiler
│   ├── Hardware Binding
│   ├── ERC
│   ├── Routing Solver
│   └── ExecutionPlan
│
├── Hardware Service
│   ├── HardwareStation
│   ├── VirtualHardwareStation
│   ├── PhysicalHardwareStation
│   ├── FPGA Driver
│   ├── Transport
│   └── Safety Controller
│
├── FPGA
│   ├── Protocol
│   ├── Control
│   ├── Routing
│   ├── Clock
│   ├── DDS
│   ├── Trigger
│   ├── Logic Analyzer
│   ├── Acquisition
│   └── SDRAM
│
└── Infrastructure
    ├── PostgreSQL
    ├── File Storage
    ├── Nginx
    ├── systemd
    └── Logging
```

---

# 44. Luồng Virtual Hardware

```text
User
 ↓
Circuit Editor
 ↓
DesignGraph
 ↓
Backend
 ↓
Validation
 ↓
ValidatedGraph
 ↓
Virtual Compiler
 ↓
Discrete Event Simulator
 ↓
VirtualHardwareStation
 ↓
Waveform / State Events
 ↓
WebSocket
 ↓
LED / Logic Analyzer / Oscilloscope
```

Mục tiêu đầu tiên là hoàn thiện pipeline này end-to-end.

---

# 45. Luồng Physical Hardware

```text
User
 ↓
Circuit Editor
 ↓
DesignGraph
 ↓
Backend
 ↓
Validation
 ↓
ValidatedGraph
 ↓
Physical Compiler
 ↓
ERC
 ↓
Routing Solver
 ↓
ExecutionPlan
 ↓
PhysicalHardwareStation
 ↓
FPGA
 ↓
Routing Fabric
 ↓
Real Logic IC
 ↓
Capture / Telemetry
 ↓
WebSocket
 ↓
Browser Instruments
```

Frontend không phải viết lại khi chuyển từ Virtual sang Physical.

---

# 46. Unified Hardware Abstraction

```text
HardwareStation
│
├── VirtualHardwareStation
└── PhysicalHardwareStation
```

Application chỉ phụ thuộc `HardwareStation`.

```text
ExperimentManager
      ↓
HardwareStation
      ↓
Virtual hoặc Physical
```

---

# 47. Lộ trình phát triển

## Phase A — Architecture Stabilization

Giữ:

```text
Vue
Three.js
CircuitGraph
FastAPI
HardwareStation
```

Chuẩn hóa:

```text
DesignGraph
ValidatedGraph
ExecutionPlan
Capabilities
```

## Phase B — Complete Virtual Circuit Engine

Xây:

```text
event scheduler
logic propagation
clock
basic gates
LED
switch
7-segment
flip-flop
counter
register
```

Mục tiêu:

```text
Web → Simulator → Web
```

hoàn chỉnh.

## Phase C — Virtual Instruments

Thêm:

```text
virtual generator
virtual oscilloscope
virtual logic analyzer
```

## Phase D — Backend Lifecycle

Xây:

```text
Projects
Experiments
Sessions
Persistence
Station Manager
Capture Manager
```

Chuyển:

```text
SQLite → PostgreSQL
```

## Phase E — Mini PC Deployment

Triển khai:

```text
Nginx
FastAPI
PostgreSQL
Hardware Service
systemd
```

Stress test 30–50 users.

## Phase F — FPGA Transport Emulator

Tạo Fake FPGA để giả lập:

```text
command
response
timeout
CRC error
disconnect
device reset
```

## Phase G — EP4CE6 Prototype

Chỉ triển khai:

```text
protocol
GPIO
clock
basic routing
small capture
safe state
```

Chưa làm SDRAM.

## Phase H — Physical Routing Fabric

Xây:

```text
MUX / Crosspoint Hardware
Routing Solver
Physical Mapping
ERC
```

## Phase I — EP4CE10 Migration

```text
EP4CE6
   ↓
EP4CE10
```

Giữ protocol tương thích.

## Phase J — SDRAM + Logic Analyzer

Thêm:

```text
capture buffer
trigger
timestamp
compression
```

## Phase K — Function Generator

Thêm:

```text
DDS
frequency
amplitude
offset
waveform selection
```

## Phase L — Oscilloscope

Thêm:

```text
ADC
AFE
trigger
SDRAM capture
decimation
waveform transfer
```

Oscilloscope nên triển khai sau Logic Analyzer.

## Phase M — Multi-user Physical Lab

Thêm:

```text
reservation
lease
queue
permissions
recovery
audit
```

---

# 48. Technology Stack chính thức

| Layer | Công nghệ |
|---|---|
| Web Framework | Vue 3 |
| Frontend Language | TypeScript |
| Build Tool | Vite |
| UI State | Pinia |
| 3D | Three.js |
| 3D Asset | GLB + Meshopt + KTX2 |
| Backend API | FastAPI |
| Validation | Pydantic / JSON Schema |
| Hardware RPC | gRPC + Protobuf |
| Simulation | Python Discrete Event Engine |
| Database | PostgreSQL |
| Browser Realtime | WebSocket |
| Large Waveform | WebSocket Binary |
| Reverse Proxy | Nginx |
| Service Manager | systemd |
| FPGA Prototype | EP4CE6 |
| FPGA Final | EP4CE10 + SDRAM |
| Prototype Transport | USB-UART / USB-SPI |
| High-throughput Transport | USB 3 FIFO |
| Edge Server | Mini PC Xeon W-1370 |

---

# 49. Công nghệ chưa lựa chọn ở giai đoạn hiện tại

Không dùng ngay:

```text
Kubernetes
Kafka
RabbitMQ
Redis Cluster
Full Microservices
GraphQL
Electron
Unity
Unreal Engine
Rust Rewrite
WebRTC Data Plane
PCIe FPGA
Cloud Database bắt buộc
Server-side 3D Rendering
```

Lý do:

> Chưa giải quyết một vấn đề thực tế đủ lớn để bù cho độ phức tạp và chi phí bảo trì.

---

# 50. Ba nguyên tắc kiến trúc bất biến

## Nguyên tắc 1

```text
CircuitGraph ≠ Three.js Scene
```

CircuitGraph là source of truth; Three.js là rendering/projection layer.

## Nguyên tắc 2

```text
CircuitGraph ≠ FPGA Registers
```

Giữa chúng luôn phải có:

```text
Compiler
ExecutionPlan
Hardware Driver
```

## Nguyên tắc 3

```text
Virtual Hardware
        │
        └──── same abstraction ──── Physical Hardware
```

Thông qua:

```text
HardwareStation
```

Nhờ đó frontend không cần rewrite khi chuyển sang hardware thật.

---

# 51. Kiến trúc cuối cùng được chốt

Tên kiến trúc:

> **Contract-Driven Virtual-to-Physical Laboratory Architecture**

```text
                    USER
                      │
                      ▼
               Web CAD Workbench
                      │
                      ▼
                  DesignGraph
                      │
                      ▼
                 Validation
                      │
                      ▼
                ValidatedGraph
                      │
          ┌───────────┴────────────┐
          │                        │
          ▼                        ▼
 Virtual Execution             Physical Compiler
          │                        │
          ▼                        ▼
Discrete Event Simulator      Routing + ERC
          │                        │
          ▼                        ▼
VirtualHardwareStation       ExecutionPlan
          │                        │
          │                        ▼
          │                PhysicalHardwareStation
          │                        │
          │                        ▼
          │                      FPGA
          │                        │
          │                        ▼
          │                Routing Hardware
          │                        │
          │                        ▼
          │                    Real ICs
          │                        │
          └──────────────┬─────────┘
                         ▼
               Unified Result Model
                         │
                         ▼
                 WebSocket Stream
                         │
                         ▼
      LED / Logic Analyzer / Oscilloscope
```

---

# 52. Định hướng phát triển dài hạn

Kiến trúc cho phép mở rộng:

```text
Multiple Mini PCs
Multiple Physical Stations
Multiple FPGA Boards
Remote Scheduling
Instructor Dashboard
Automated Experiments
Experiment Recording
Replay
Hardware Calibration
Digital Twin
Cloud Management
Remote Firmware Update
Advanced Logic Analyzer
Oscilloscope
Function Generator
Protocol Decoder
AI-assisted Circuit Analysis
```

Mỗi tính năng mới phải đi qua contract/interface rõ ràng, không phá vỡ CircuitGraph và HardwareStation abstraction.

---

# 53. Kết luận

Kiến trúc chính thức đề xuất cho **net*CIRCUIT Remote**:

> **Web-first, Simulation-first, Contract-driven, Virtual-to-Physical Remote Digital Electronics Laboratory**

Các quyết định trọng tâm:

1. Giữ Vue 3 + TypeScript + Three.js.
2. CircuitGraph là source of truth.
3. Backend dùng FastAPI theo Modular Monolith.
4. Hardware Service tách process.
5. Hoàn thiện Virtual Hardware trước Physical Hardware.
6. Simulator dùng Discrete Event.
7. Hardware abstraction qua `HardwareStation`.
8. Physical Circuit đi qua Compiler + ERC + Routing Solver.
9. Mini PC Xeon W-1370 là Edge Server/Controller.
10. FPGA chỉ xử lý deterministic/timing-critical tasks.
11. Prototype dùng EP4CE6.
12. Final dùng EP4CE10 + SDRAM.
13. PC ↔ FPGA ban đầu dùng USB-UART/USB-SPI; tương lai dùng USB 3 FIFO nếu cần throughput lớn.
14. PostgreSQL thay SQLite khi bước sang production/multi-user.
15. Không dùng Kubernetes/full microservices khi chưa có nhu cầu thực tế.
16. Frontend không phụ thuộc FPGA register, MUX address hoặc transport vật lý.
17. Cùng một workflow chạy được trên `VirtualHardwareStation` và `PhysicalHardwareStation`.

Kiến trúc này cân bằng giữa:

- hiệu năng;
- độ ổn định;
- khả năng mở rộng;
- khả năng bảo trì;
- trải nghiệm người dùng;
- phạm vi luận văn;
- khả năng tiếp tục phát triển thành hệ thống thực tế sau luận văn.
