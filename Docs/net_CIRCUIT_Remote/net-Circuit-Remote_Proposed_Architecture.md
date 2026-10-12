# net*CIRCUIT Remote — Kiến trúc chính thức được đề xuất

## 1. Mục tiêu kiến trúc

**net*CIRCUIT Remote** được định hướng là một nền tảng **Remote Digital Electronics Laboratory** theo mô hình **Web-CAD / Workbench**, cho phép người dùng thiết kế, mô phỏng, chạy thí nghiệm và quan sát tín hiệu của mạch điện tử số từ trình duyệt.

Kiến trúc chính thức được đề xuất theo nguyên tắc:

> **Web-first, Simulation-first, Contract-driven, Virtual-to-Physical**

Mục tiêu cốt lõi:

- Dùng **Circuit Graph** làm nguồn sự thật duy nhất của thiết kế.
- Hoàn thiện **Virtual Hardware** trước.
- Giữ chung workflow giữa Virtual và Physical.
- Khi chuyển sang phần cứng thật, frontend không cần viết lại.
- Mini PC làm Edge Server / Controller.
- FPGA xử lý các tác vụ deterministic và timing-critical.
- Routing hardware kết nối FPGA tới IC logic thật.
- Kiến trúc đủ mở rộng cho nhiều user, nhiều station và nhiều loại instrument về sau.

---

# 2. Cấu hình hệ thống mục tiêu

## 2.1 Edge Server

Sử dụng Mini PC:

- CPU: **Intel Xeon W-1370**
- RAM: **16 GB**
- SSD: **256 GB**
- Hệ điều hành khuyến nghị: Linux
- Vai trò:
  - Web Server
  - API Backend
  - Circuit Validation
  - Circuit Compiler
  - Digital Simulator
  - Experiment Manager
  - Station Manager
  - Database
  - Hardware Service
  - FPGA communication
  - Waveform processing
  - Session / lease / queue management
  - Storage metadata

## 2.2 Client

Trình duyệt của người dùng chịu trách nhiệm:

- Render 3D bằng Three.js/WebGL.
- Hiển thị circuit workbench.
- Tương tác kéo thả linh kiện.
- Điều khiển instrument UI.
- Hiển thị waveform, LED state, logic state.
- Gửi thao tác người dùng đến backend.

> Mini PC không render 3D thay cho toàn bộ client.

---

# 3. Kiến trúc tổng thể

Kiến trúc chính thức:

> **Web SPA + Modular Monolith Backend + Isolated Hardware Service + Contract-Driven FPGA**

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
└─────────┼─────────────────────────────────────────────────────┘
          │
          │ REST / HTTPS
          │ WebSocket
          ▼
┌──────────────────────── MINI PC ───────────────────────────────┐
│                                                               │
│                         Nginx                                 │
│                           │                                   │
│           ┌───────────────┴───────────────┐                   │
│           ▼                               ▼                   │
│     Static Web Files                  FastAPI                 │
│                                       Backend                 │
│                                           │                   │
│    ┌──────────────────────────────────────┼────────────────┐  │
│    │                                      │                │  │
│    ▼                                      ▼                ▼  │
│ Circuit Domain                    Experiment Manager      Auth │
│ Validation                       Session / Lease          DB   │
│ Compiler                             │                         │
│    │                                 │                         │
│    ├─────────────┐                   │                         │
│    ▼             ▼                   │                         │
│ Virtual       Physical               │                         │
│ Compiler      Compiler               │                         │
│    │             │                   │                         │
│    ▼             ▼                   │                         │
│ Simulator   ExecutionPlan            │                         │
│    │             │                   │                         │
│    └─────────────┴──────────┬────────┘                         │
│                             ▼                                  │
│                       Hardware Service                          │
│                       separate process                         │
│                             │                                  │
│                      gRPC / Protobuf                           │
│                             │                                  │
│                       FPGA Host Driver                          │
│                             │                                  │
└─────────────────────────────┼──────────────────────────────────┘
                              │
                       USB / USB3
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

# 4. Nguyên tắc kiến trúc bắt buộc

## 4.1 CircuitGraph là nguồn sự thật

```text
CircuitGraph ≠ Three.js Scene
```

Three.js chỉ render trạng thái từ CircuitGraph.

## 4.2 CircuitGraph không phụ thuộc FPGA

```text
CircuitGraph ≠ FPGA Registers
```

Giữa CircuitGraph và FPGA luôn phải có:

```text
Validation
Compiler
ExecutionPlan
Hardware Driver
```

## 4.3 Virtual và Physical dùng chung abstraction

```text
HardwareStation
│
├── VirtualHardwareStation
└── PhysicalHardwareStation
```

Application không được phụ thuộc trực tiếp implementation bên dưới.

## 4.4 Frontend không biết chi tiết phần cứng

Frontend không chứa:

- MUX channel.
- Crosspoint address.
- FPGA register.
- SPI command.
- USB endpoint.
- Physical socket route.

Các thông tin đó thuộc backend/hardware domain.

---

# 5. Circuit Data Model

Circuit được chia thành ba tầng.

```text
DesignGraph
     ↓
ValidatedGraph
     ↓
ExecutionPlan
```

---

# 6. DesignGraph

DesignGraph là mô hình circuit do người dùng trực tiếp chỉnh sửa.

Bao gồm:

```text
Project
Circuit
ComponentInstance
Port
Connection
Property
Position
Rotation
Label
Metadata
```

Dùng cho:

- Save/Open.
- Undo/Redo.
- Circuit editing.
- 3D rendering.
- User interaction.

Ví dụ:

```text
SW1.OUT → AND1.A
AND1.Y  → LED1.IN
```

---

# 7. ValidatedGraph

Backend nhận DesignGraph và thực hiện normalize + validate.

Kiểm tra:

```text
component existence
port existence
port direction
signal width
net resolution
multiple drivers
required power
electrical constraints
unsupported devices
invalid connections
```

Ví dụ:

```text
SW1.OUT ───── Net0001 ───── AND1.A
```

ValidatedGraph là logical circuit đã hợp lệ.

---

# 8. ExecutionPlan

ExecutionPlan là output của compiler.

Có hai loại:

```text
VirtualExecutionPlan
PhysicalExecutionPlan
```

## VirtualExecutionPlan

Chứa thông tin phục vụ simulator:

```text
device model mapping
net dependency
event schedule metadata
initial state
timing parameters
```

## PhysicalExecutionPlan

Chứa:

```text
logical-to-physical binding
socket mapping
routing resources
crosspoint route
FPGA resource allocation
instrument channel allocation
```

ExecutionPlan không phải dữ liệu do người dùng chỉnh sửa trực tiếp.

---

# 9. Domain Separation

## 9.1 Circuit Domain

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

## 9.2 Rendering Domain

```text
Rendering Domain
│
├── SceneObject
├── Transform
├── Mesh
├── Material
├── Camera
├── Selection
└── Gizmo
```

## 9.3 Physical Domain

```text
Physical Domain
│
├── Station
├── HardwareProfile
├── Socket
├── RoutingResource
├── Crosspoint
├── FPGAResource
├── InstrumentChannel
└── CalibrationProfile
```

Ba domain không được trộn lẫn.

---

# 10. Component Library

Mỗi component definition chứa:

```text
identity
display metadata
logical ports
logic model
electrical constraints
3D asset
physical package
pin mapping
simulation model
```

Ví dụ:

```text
74HC08
│
├── logic model
├── DIP14 package
├── pins
├── power pins
├── simulation model
└── GLB model
```

Routing resource không lưu trong component definition.

Routing thuộc Station / Hardware Profile.

---

# 11. Frontend Architecture

Frontend sử dụng:

```text
Vue 3
TypeScript
Vite
Pinia
Three.js
```

Kiến trúc module:

```text
Web Application
│
├── Workspace Shell
├── Circuit Editor
├── Component Library UI
├── 3D Renderer
├── Instrument UI
├── Project Manager
├── Property Inspector
├── Toolbar / Tool State
├── API Client
└── WebSocket Client
```

---

# 12. Frontend State Management

Phân chia state:

```text
UI State
    Pinia

Circuit State
    CircuitGraph Engine

Renderer State
    SceneManager

Remote State
    API / WebSocket Client

Hardware State
    Server authoritative
```

Pinia chỉ quản lý UI/application state.

Không dùng Pinia làm:

- Simulator.
- Database.
- FPGA state machine.
- 3D engine.

---

# 13. 3D Rendering Architecture

Three.js tiếp tục là 3D engine chính.

Các module:

```text
SceneManager
AssetManager
WireLayer
SelectionManager
PickingManager
TransformGizmo
CameraController
PlacementGhost
ViewportController
```

Three.js không phải source of truth.

CircuitGraph thay đổi:

```text
CircuitGraph
    ↓
Scene sync
    ↓
Three.js
```

---

# 14. 3D Asset Pipeline

Pipeline chuẩn:

```text
CAD / Blender
      ↓
GLB
      ↓
Mesh Optimization
      ↓
Meshopt
      ↓
KTX2 / Basis Texture
      ↓
Browser
```

Áp dụng:

- GLB.
- Meshopt.
- KTX2.
- Asset cache.
- Lazy loading.
- InstancedMesh.
- Level of Detail.

---

# 15. 3D Performance Strategy

## Instancing

Các chi tiết lặp lại như breadboard hole sử dụng InstancedMesh.

## Asset Cache

Model cùng loại chỉ tải một lần.

## Lazy Loading

Instrument hoặc component chưa dùng thì chưa tải model.

## LOD

```text
near camera → high detail
far camera  → low detail
```

## Renderer isolation

3D renderer không được chứa logic simulation hoặc hardware logic.

---

# 16. Undo / Redo

Giai đoạn đầu tiếp tục sử dụng snapshot history có giới hạn.

Quy tắc:

- Một thao tác drag hoàn chỉnh tương ứng một history entry.
- Không tạo history liên tục theo mỗi frame.
- Giới hạn số snapshot.
- Giới hạn dung lượng history.

Khi circuit lớn mới chuyển sang command-based history.

---

# 17. Backend Architecture

Backend sử dụng:

> **FastAPI Modular Monolith**

Module:

```text
Backend
│
├── Auth
├── Project Service
├── Circuit Service
├── Validation Service
├── Simulation Service
├── Experiment Manager
├── Station Manager
├── Capture Manager
├── Persistence
├── WebSocket Gateway
└── Hardware Service Client
```

Không tách full microservices trong V1.

---

# 18. API Communication

Browser ↔ Backend:

```text
REST
+
WebSocket
```

## REST

Dùng cho:

```text
authentication
projects
circuits
save/open
station info
experiment creation
configuration
metadata
```

## WebSocket

Dùng cho:

```text
simulation events
logic states
LED updates
experiment state
hardware telemetry
logic analyzer results
oscilloscope capture
```

---

# 19. Hardware Service

Hardware Service chạy dưới process riêng.

```text
FastAPI
   │
   │ gRPC
   ▼
Hardware Service
   │
   ▼
PhysicalHardwareStation
   │
   ▼
FPGA Driver
   │
   ▼
Transport
   │
   ▼
FPGA
```

Mục tiêu:

- Cách ly hardware failure.
- Quản lý driver.
- Quản lý protocol.
- Quản lý safety.
- Không để lỗi USB/FPGA làm chết API.

---

# 20. HardwareStation Abstraction

Interface logic:

```text
get_capabilities()
validate_configuration()
apply_circuit()
set_input()
configure_clock()
configure_generator()
configure_trigger()
arm_capture()
run()
stop()
read_capture()
safe_state()
reset()
```

Implementation:

```text
HardwareStation
│
├── VirtualHardwareStation
└── PhysicalHardwareStation
```

Experiment Manager chỉ giao tiếp qua abstraction này.

---

# 21. Simulator Architecture

Simulator sử dụng:

> **Server-side Python Discrete Event Simulation**

Không dùng fixed timestep toàn hệ thống.

Luồng:

```text
input change
    ↓
event
    ↓
device evaluate
    ↓
propagation delay
    ↓
output event
    ↓
dependent devices
```

Ví dụ:

```text
t = 0 ns
SW1 = 1

t = 8 ns
AND1.Y = 1

t = 8 ns
LED1 = ON
```

---

# 22. Simulation Modules

```text
Simulation
│
├── Event Scheduler
├── Net Resolver
├── Device Model Registry
├── Clock Engine
├── Digital State Engine
├── Timing Model
├── Virtual Generator
├── Virtual Oscilloscope
├── Virtual Logic Analyzer
└── Waveform Builder
```

---

# 23. Virtual Hardware Flow

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
VirtualExecutionPlan
 ↓
Discrete Event Simulator
 ↓
VirtualHardwareStation
 ↓
Unified Result Model
 ↓
WebSocket
 ↓
Browser Instruments
```

Pipeline này phải được hoàn thiện trước khi bắt đầu Physical Hardware hoàn chỉnh.

---

# 24. Physical Hardware Flow

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
PhysicalExecutionPlan
 ↓
PhysicalHardwareStation
 ↓
FPGA
 ↓
Routing Fabric
 ↓
Real IC
 ↓
Capture / Telemetry
 ↓
Unified Result Model
 ↓
WebSocket
 ↓
Browser Instruments
```

---

# 25. Physical Compiler

Physical Compiler chịu trách nhiệm:

```text
logical component
        ↓
physical socket
        ↓
physical pin
        ↓
routing resource
        ↓
FPGA channel
```

Ví dụ:

```text
74HC08.Y1
→ socket U3 pin 3

LED1.IN
→ LED channel 7
```

---

# 26. Routing Solver

Routing Solver giải bài toán constrained graph routing.

Đầu vào:

```text
logical nets
hardware topology
available routing resources
electrical constraints
reserved resources
```

Đầu ra:

```text
RoutingPlan
```

Thuật toán nền:

- BFS.
- Dijkstra.
- Backtracking.
- Resource reservation.

Mục tiêu:

```text
valid route
+
no conflict
+
electrical safety
+
reasonable route length
```

---

# 27. Electrical Rule Checker

ERC chạy trước Physical Execution.

Kiểm tra:

```text
output-output short
VCC-GND short
multiple drivers
invalid voltage
missing power
floating required pin
unsupported component
unsupported connection
fan-out violation
routing impossible
```

Nếu không đạt:

```text
Reject Experiment
```

Không cấu hình FPGA.

---

# 28. Hardware Safety State Machine

State machine chính thức:

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
disable digital outputs
stop clocks
disable generator
tri-state FPGA outputs
disable capture
release unsafe resources
```

---

# 29. Mini PC ↔ FPGA Transport

Do Mini PC không có GPIO/SPI trực tiếp như Raspberry Pi, transport được abstraction hóa.

```text
FpgaTransport
│
├── SerialTransport
├── SpiUsbTransport
├── Usb3Transport
└── EthernetTransport
```

## Prototype

Khuyến nghị:

```text
USB-UART
hoặc
USB-SPI
```

## Final high-throughput

Khuyến nghị:

```text
USB 3 FIFO
```

đặc biệt khi triển khai:

- Logic Analyzer.
- Oscilloscope.
- Large capture.

---

# 30. FPGA Protocol

Protocol frame:

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

Command:

```text
GET_CAPABILITY
SET_CLOCK
SET_ROUTE
WRITE_OUTPUT
CONFIGURE_TRIGGER
ARM_CAPTURE
START_CAPTURE
READ_CAPTURE
STOP
RESET
SAFE_STATE
```

Mỗi command có:

```text
sequence
timeout
response
error_code
```

---

# 31. FPGA Responsibilities

FPGA xử lý:

```text
Clock Generation
Deterministic Switching
Timestamping
Routing Control
Logic Capture
Trigger
Digital IO
DDS / Generator
ADC Interface
SDRAM Controller
Acquisition
```

FPGA không xử lý:

- User management.
- Project management.
- Circuit editing.
- Database.
- WebSocket.
- Authentication.
- Circuit persistence.

---

# 32. Mini PC Responsibilities

Mini PC xử lý:

```text
Web Server
API
Authentication
Database
Circuit Validation
Virtual Simulation
Physical Compiler
Routing Solver
Experiment Lifecycle
Session / Lease
Capture Management
Waveform Processing
Hardware Coordination
Storage
Logging
```

---

# 33. Capability Discovery

Frontend không hard-code số lượng channel/resource.

Station trả:

```text
StationCapabilities
```

Ví dụ:

```text
digitalInputs
digitalOutputs
logicAnalyzerChannels
oscilloscopeChannels
generatorChannels
maxSampleRate
routingResources
supportedComponents
supportedFeatures
firmwareVersion
```

Frontend tự điều chỉnh UI dựa trên capabilities.

---

# 34. Logic Analyzer Architecture

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
WebSocket Binary
     ↓
Browser
```

Có thể dùng:

```text
RLE
```

để giảm dung lượng digital capture.

---

# 35. Oscilloscope Architecture

```text
Analog Input
    ↓
AFE
    ↓
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
Decimation / Windowing
    ↓
Mini PC
    ↓
WebSocket Binary
    ↓
Browser
```

Không stream toàn bộ raw ADC liên tục lên browser.

---

# 36. Function Generator Architecture

```text
Browser Control
      ↓
Backend
      ↓
Hardware Service
      ↓
FPGA DDS
      ↓
DAC / Output Stage
      ↓
Physical Circuit
```

FPGA quản lý:

```text
frequency
phase
waveform
timing
trigger synchronization
```

---

# 37. Waveform Transport

Không dùng JSON cho capture lớn.

Dùng:

> **WebSocket Binary Frame**

Header:

```text
capture_id
instrument
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
RLE blocks
```

tùy loại dữ liệu.

---

# 38. Database

Production/multi-user sử dụng:

> **PostgreSQL**

Lưu:

```text
users
roles
projects
circuits
experiments
stations
station leases
queues
capture metadata
audit logs
settings
```

SQLite chỉ phù hợp prototype.

---

# 39. Capture Storage

Không lưu waveform lớn trực tiếp trong PostgreSQL.

PostgreSQL lưu metadata.

Capture data lưu:

```text
/data/captures/
```

Có thể nâng thành object storage trong tương lai.

---

# 40. Authentication và Authorization

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

Tương lai có thể tích hợp:

```text
OIDC
SSO
University account
```

---

# 41. Multi-user Architecture

Virtual:

```text
many users
→ independent virtual sessions
```

Physical:

```text
1 station
→ 1 active owner
```

Cần:

```text
Reservation
Lease
Queue
Timeout
Ownership
Recovery
```

Experiment Manager chịu trách nhiệm quản lý toàn bộ lifecycle.

---

# 42. Station Lease

Ví dụ:

```text
User A
  ↓
Station 01
  ↓
Lease Active

User B
  ↓
Waiting Queue

User C
  ↓
Waiting Queue
```

Lease hết hạn hoặc client mất kết nối:

```text
STOP
↓
SAFE_STATE
↓
release resources
↓
next user
```

---

# 43. Deployment Architecture

Mini PC chạy:

```text
Linux
Nginx
FastAPI
Hardware Service
PostgreSQL
systemd
```

System services:

```text
nginx.service
postgresql.service
netcircuit-api.service
netcircuit-hardware.service
```

Docker Compose có thể dùng cho:

- Development.
- CI.
- Integration test.

Không bắt buộc cho production V1.

---

# 44. Storage Capacity

SSD 256 GB đủ cho giai đoạn đầu.

Khi có:

```text
large logic captures
oscilloscope captures
history
logs
many projects
3D assets
```

nên nâng lên:

```text
512 GB hoặc 1 TB NVMe
```

---

# 45. Unified Result Model

Virtual và Physical đều trả kết quả theo cùng logical contract.

Ví dụ:

```text
ExperimentResult
│
├── state
├── digitalSignals
├── analogCaptures
├── timestamps
├── instrumentStatus
├── warnings
└── errors
```

Frontend không cần biết dữ liệu đến từ:

```text
Virtual Simulator
```

hay:

```text
Physical FPGA
```

---

# 46. Component Architecture chính thức

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
│   ├── Property Inspector
│   ├── API Client
│   └── WebSocket Client
│
├── Circuit Domain
│   ├── DesignGraph
│   ├── ValidatedGraph
│   ├── Net Resolver
│   ├── Validation
│   └── Serialization
│
├── Backend
│   ├── Auth
│   ├── Project Service
│   ├── Circuit Service
│   ├── Validation Service
│   ├── Experiment Manager
│   ├── Station Manager
│   ├── Capture Manager
│   ├── Persistence
│   └── WebSocket Gateway
│
├── Simulation
│   ├── Event Scheduler
│   ├── Device Models
│   ├── Net Engine
│   ├── Clock Engine
│   ├── Timing Model
│   ├── Virtual Generator
│   ├── Virtual Logic Analyzer
│   └── Virtual Oscilloscope
│
├── Physical Compiler
│   ├── Hardware Binding
│   ├── ERC
│   ├── Routing Solver
│   └── PhysicalExecutionPlan
│
├── Hardware Service
│   ├── HardwareStation
│   ├── VirtualHardwareStation
│   ├── PhysicalHardwareStation
│   ├── FPGA Driver
│   ├── FPGA Protocol
│   ├── FPGA Transport
│   └── Safety Controller
│
├── FPGA
│   ├── Protocol Engine
│   ├── Control Engine
│   ├── Routing Controller
│   ├── Clock Generator
│   ├── Trigger Engine
│   ├── Logic Analyzer
│   ├── DDS
│   ├── ADC Interface
│   ├── Acquisition
│   └── SDRAM Controller
│
└── Infrastructure
    ├── PostgreSQL
    ├── Capture Storage
    ├── Nginx
    ├── systemd
    └── Logging
```

---

# 47. Technology Stack chính thức

| Layer | Công nghệ |
|---|---|
| Frontend | Vue 3 |
| Frontend Language | TypeScript |
| Build Tool | Vite |
| State Management | Pinia |
| 3D Engine | Three.js |
| 3D Asset | GLB + Meshopt + KTX2 |
| Backend | FastAPI |
| Validation | Pydantic / JSON Schema |
| Hardware RPC | gRPC + Protobuf |
| Simulation | Python Discrete Event Engine |
| Database | PostgreSQL |
| Realtime Browser | WebSocket |
| Waveform Transport | WebSocket Binary |
| Reverse Proxy | Nginx |
| Service Management | systemd |
| Prototype FPGA | EP4CE6 |
| Final FPGA | EP4CE10 + SDRAM |
| Prototype PC-FPGA Transport | USB-UART / USB-SPI |
| High-throughput PC-FPGA Transport | USB 3 FIFO |
| Edge Server | Mini PC Xeon W-1370 |

---

# 48. Công nghệ không dùng trong V1

Không sử dụng nếu chưa có nhu cầu thực tế:

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
PCIe FPGA
Server-side 3D Rendering
Cloud-only Database
```

Mục tiêu là tránh overengineering.

---

# 49. Roadmap phát triển chính thức

## Phase A — Architecture Stabilization

Chuẩn hóa:

```text
DesignGraph
ValidatedGraph
ExecutionPlan
HardwareStation
StationCapabilities
Unified Result Model
```

---

## Phase B — Virtual Circuit Engine

Xây:

```text
Event Scheduler
Logic Propagation
Clock
Basic Gates
LED
Switch
7-Segment
Flip-Flop
Counter
Register
```

Mục tiêu:

```text
Web
 ↓
Simulator
 ↓
Web
```

hoàn chỉnh.

---

## Phase C — Virtual Instruments

Xây:

```text
Virtual Generator
Virtual Logic Analyzer
Virtual Oscilloscope
```

---

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

Chuyển database:

```text
SQLite
  ↓
PostgreSQL
```

---

## Phase E — Mini PC Deployment

Triển khai:

```text
Nginx
FastAPI
PostgreSQL
Hardware Service
systemd
```

Stress test:

```text
30–50 users
```

---

## Phase F — FPGA Transport Emulator

Tạo Fake FPGA để test:

```text
command
response
timeout
CRC error
disconnect
reset
```

---

## Phase G — EP4CE6 Prototype

Triển khai:

```text
protocol
GPIO
clock
basic routing
small capture
safe state
```

Không thêm SDRAM ở prototype.

---

## Phase H — Physical Routing Fabric

Triển khai:

```text
MUX / Crosspoint Hardware
Hardware Profile
Physical Binding
Routing Solver
ERC
```

---

## Phase I — EP4CE10 Migration

```text
EP4CE6
   ↓
EP4CE10
```

Giữ protocol tương thích.

---

## Phase J — SDRAM + Logic Analyzer

Thêm:

```text
SDRAM
capture buffer
trigger
timestamp
compression
```

---

## Phase K — Function Generator

Thêm:

```text
DDS
frequency
phase
waveform
synchronization
```

---

## Phase L — Oscilloscope

Thêm:

```text
AFE
ADC
trigger
SDRAM capture
decimation
waveform transport
```

---

## Phase M — Multi-user Physical Lab

Thêm:

```text
Reservation
Lease
Queue
Permission
Recovery
Audit
```

---

# 50. Kiến trúc Virtual-to-Physical cuối cùng

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
 Virtual Compiler              Physical Compiler
          │                        │
          ▼                        ▼
VirtualExecutionPlan          ERC + Routing Solver
          │                        │
          ▼                        ▼
Discrete Event Simulator      PhysicalExecutionPlan
          │                        │
          ▼                        ▼
VirtualHardwareStation       PhysicalHardwareStation
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

# 51. Kết luận kiến trúc

Kiến trúc chính thức được đề xuất cho **net*CIRCUIT Remote** là:

> **Contract-Driven Virtual-to-Physical Laboratory Architecture**

Các quyết định kiến trúc cốt lõi:

1. **Vue 3 + TypeScript + Three.js** cho Web Workbench.
2. **CircuitGraph** là source of truth.
3. Tách dữ liệu thành **DesignGraph → ValidatedGraph → ExecutionPlan**.
4. **FastAPI Modular Monolith** làm backend.
5. **Hardware Service** là process riêng.
6. **gRPC + Protobuf** cho Backend ↔ Hardware Service.
7. **REST + WebSocket** cho Browser ↔ Backend.
8. **Python Discrete Event Simulator** cho Virtual Hardware.
9. `HardwareStation` là abstraction chung cho Virtual và Physical.
10. **Physical Compiler + ERC + Routing Solver** đứng giữa logical circuit và hardware.
11. **Mini PC Xeon W-1370** làm Edge Server/Controller.
12. **FPGA** chỉ đảm nhiệm các tác vụ deterministic/timing-critical.
13. **EP4CE6** dùng cho prototype.
14. **EP4CE10 + SDRAM** dùng cho bản final.
15. PC ↔ FPGA dùng abstraction transport; prototype dùng USB-UART/USB-SPI, high-throughput dùng USB 3 FIFO.
16. **PostgreSQL** dùng cho production/multi-user.
17. **WebSocket Binary** dùng cho waveform/capture data.
18. Không sử dụng các công nghệ phức tạp nếu chúng chưa giải quyết một nhu cầu thực tế.

Mục tiêu cuối cùng là cho phép cùng một circuit:

```text
DesignGraph
```

có thể chạy:

```text
Virtual Simulation
```

hoặc:

```text
Physical Experiment
```

mà frontend, workflow và logical model không cần thay đổi.
