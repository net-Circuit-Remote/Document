# net*CIRCUIT Remote — Kiến trúc chính thức được đề xuất

> **Bản cập nhật kiến trúc v2**  
> Bản này bổ sung bốn quyết định đã chốt:
> 1. Tự động dò kết nối/vị trí vật lý trên breadboard theo **Phương án C — Active Electrical Topology Discovery**.
> 2. Mini PC hỗ trợ cả **mạch số và mạch tương tự** ở tầng phần mềm/mô phỏng; FPGA phiên bản hiện tại chỉ chạy **Physical Digital Circuit**.
> 3. Hỗ trợ **30–50 người dùng kết nối**, còn thí nghiệm vật lý được điều phối theo execution slot/queue thay vì giả định tất cả Run cùng một thời điểm.
> 4. EP4CE10 tích hợp logic điều khiển **Oscilloscope 2 kênh + Function Generator 2 kênh**, với ADC/DAC/AFE/SDRAM bên ngoài FPGA.


## 1. Mục tiêu kiến trúc

**net*CIRCUIT Remote** được định hướng là một nền tảng **Remote Electronics Laboratory** theo mô hình **Web-CAD / Workbench**, phát triển theo hướng **digital-first nhưng mixed-signal-ready**.

Ở tầng Mini PC, kiến trúc phần mềm phải hỗ trợ được cả:
- **Mạch số**.
- **Mạch tương tự**.
- Khả năng mở rộng mixed-signal trong tương lai.

Ở tầng phần cứng hiện tại:
- FPGA/Physical Hardware phiên bản V1 chỉ chạy **mạch số**.
- Oscilloscope và Function Generator vẫn được hỗ trợ như các **instrument** sử dụng ADC/DAC/AFE bên ngoài FPGA.
- Khi phát triển thêm Physical Analog Hardware trong tương lai, kiến trúc Mini PC và frontend không phải thay đổi nền tảng.

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

## 8.1 Signal Domain

Circuit Graph phải mô tả loại miền tín hiệu để cùng một kiến trúc Mini PC có thể phục vụ cả digital và analog.

```text
SignalDomain
│
├── DIGITAL
├── ANALOG
├── POWER
├── GROUND
├── CLOCK
└── MIXED
```

Ví dụ:

```text
74HC08.A1     → DIGITAL
LM358.IN+     → ANALOG
VCC           → POWER
GND           → GROUND
CLK           → CLOCK
```

`SignalDomain` là contract ở tầng Circuit/Compiler, không phụ thuộc việc FPGA hiện tại đã hỗ trợ analog hay chưa.

### Trạng thái hỗ trợ

```text
Mini PC Virtual Digital        → SUPPORTED
Mini PC Virtual Analog         → ARCHITECTURALLY SUPPORTED
Physical Digital on FPGA V1    → SUPPORTED TARGET
Physical Analog Routing V1     → NOT SUPPORTED
Physical Analog Future         → ADD NEW HARDWARE CAPABILITY, NO PC ARCHITECTURE REWRITE
```

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

## 9.4 Physical Placement & Automatic Topology Discovery

Với breadboard 830 được thiết kế PCB riêng, mỗi nhóm lỗ điện tương đương phải có một `BreadboardNodeID` cố định.

Ví dụ:

```text
A1 ─┐
B1  │
C1  ├── BreadboardNode N001
D1  │
E1 ─┘
```

PCB ánh xạ node này tới hạ tầng scan/routing:

```text
Breadboard Hole
      ↓
BreadboardNodeID
      ↓
Scan Matrix / Sense Front-End
      ↓
Routing Resource
      ↓
FPGA
```

### Phương án được chọn: Active Electrical Topology Discovery

Không yêu cầu sinh viên khai báo vị trí linh kiện thủ công.

Khi người dùng bấm `Run`, hệ thống thực hiện:

```text
SAFE_STATE
   ↓
Disconnect normal drive paths
   ↓
Topology Scan
   ↓
DetectedTopologyGraph
   ↓
Component / Connection Resolution
   ↓
Compare with Reference Circuit
   ↓
ERC
   ↓
Physical Compiler
   ↓
Routing Solver
   ↓
Execute
```

### Các module cần bổ sung

```text
Physical Placement Layer
│
├── BreadboardNodeMap
├── ScanMatrixController
├── SafeStimulusController
├── HighImpedanceSenseEngine
├── DetectedTopologyGraph
├── ComponentSignatureLibrary
├── PlacementResolver
├── TopologyMatcher
└── BoardOccupancyMap
```

### Cơ chế scan đề xuất

1. Đưa toàn bộ station về `SAFE_STATE`.
2. Ngắt các output có thể gây xung đột.
3. Dùng nguồn test giới hạn dòng/điện áp để kích từng node.
4. Dùng ngõ đo trở kháng cao để đọc các node còn lại.
5. Xây dựng graph các kết nối dẫn điện đã phát hiện.
6. Với IC thuộc thư viện hỗ trợ, thực hiện test-vector/signature có kiểm soát để hỗ trợ suy luận loại IC và orientation.
7. Đối chiếu `DetectedTopologyGraph` với sơ đồ tham chiếu.
8. Nếu topology tương đương thì cho phép compile/run dù vị trí vật lý khác hoàn toàn.

### Nguyên tắc so sánh

Không so sánh:

```text
Teacher Position == Student Position
```

Mà so sánh:

```text
Teacher Logical Topology
        ≡
Student Detected Electrical Topology
```

Ví dụ cùng một `U1 = 74HC08`:

```text
Teacher: U1 at E10/F10
Student: U1 at E35/F35
```

vẫn hợp lệ nếu các net điện tương đương.

### Giới hạn kỹ thuật cần giữ rõ

Tự động scan có thể xác định topology dẫn điện rất tốt, nhưng **không được giả định có thể nhận dạng chắc chắn mọi IC tùy ý chỉ bằng đo điện**.

V1 nên:
- Giới hạn thư viện IC được hỗ trợ.
- Biết trước danh sách linh kiện từ bài thực hành/reference circuit.
- Dùng `ComponentSignatureLibrary` cho các IC số đã xác nhận.
- Nếu có nhiều cách ánh xạ cùng hợp lệ hoặc kết quả scan mơ hồ, hệ thống phải trả `AMBIGUOUS_TOPOLOGY` và không được tự đoán.

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

Mini PC sử dụng kiến trúc simulator kép nhưng cùng contract.

```text
ValidatedGraph
      │
      ├───────────────┐
      │               │
      ▼               ▼
Digital Compiler   Analog Compiler
      │               │
      ▼               ▼
Discrete Event     SPICE Adapter
Simulator          / Analog Engine
      │               │
      └───────┬───────┘
              ▼
      Unified Result Model
```

## 21.1 Digital Simulation

Sử dụng:

> **Server-side Python Discrete Event Simulation**

Không dùng fixed timestep toàn hệ thống.

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

## 21.2 Analog Simulation

Kiến trúc Mini PC phải có interface:

```text
AnalogSimulationEngine
```

Implementation khuyến nghị:

```text
SPICE-compatible adapter
```

Luồng:

```text
ValidatedGraph
      ↓
Analog Compiler
      ↓
SPICE Netlist / Analog Model
      ↓
AnalogSimulationEngine
      ↓
Waveform
      ↓
Unified Result Model
```

Không tự viết một SPICE solver hoàn chỉnh trong V1 nếu không có nhu cầu nghiên cứu riêng về numerical solver.

## 21.3 Mixed-Signal Coordinator

Khi cần mixed-signal trong tương lai:

```text
Digital Simulator
       │
       ├── Digital-to-Analog Bridge
       │
       └── Analog-to-Digital Bridge
                 │
                 ▼
          Analog Simulator
```

Đây là mở rộng ở Mini PC, không làm thay đổi frontend hoặc CircuitGraph contract.

---

# 22. Simulation Modules

```text
Simulation
│
├── Simulation Manager
├── Worker Pool
│
├── Digital
│   ├── Event Scheduler
│   ├── Net Resolver
│   ├── Device Model Registry
│   ├── Clock Engine
│   ├── Digital State Engine
│   └── Timing Model
│
├── Analog
│   ├── Analog Compiler
│   ├── SPICE Adapter
│   └── Analog Waveform Adapter
│
├── Mixed-Signal Coordinator
│
├── Virtual Generator
├── Virtual Oscilloscope
├── Virtual Logic Analyzer
└── Waveform Builder
```

### Simulation Worker Model

Để phục vụ 30–50 user:

```text
FastAPI
   ↓
Simulation Manager
   ↓
Process Worker Pool
   ├── Worker 1
   ├── Worker 2
   ├── ...
   └── Worker N
```

CPU-bound simulation không chạy trực tiếp trong event loop của FastAPI.

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
Signal Domain Dispatch
 ├───────────────┐
 ↓               ↓
Digital       Analog
Compiler      Compiler
 ↓               ↓
Discrete      SPICE /
Event Sim     Analog Engine
 └───────┬───────┘
         ↓
VirtualHardwareStation
         ↓
Unified Result Model
         ↓
WebSocket
         ↓
Browser Instruments
```

Pipeline Mini PC này được thiết kế ngay từ đầu để không phải thay đổi kiến trúc nền khi bổ sung physical analog hardware về sau.

---

# 24. Physical Hardware Flow

Phiên bản phần cứng hiện tại chỉ thực thi **mạch số**.

```text
User
 ↓
Circuit Editor
 ↓
Reference DesignGraph
 ↓
Backend
 ↓
Validation
 ↓
ValidatedGraph
 ↓
SAFE_STATE
 ↓
Automatic Electrical Topology Discovery
 ↓
DetectedTopologyGraph
 ↓
Topology Matcher
 ↓
Digital Physical Compiler
 ↓
Digital ERC
 ↓
Routing Solver
 ↓
PhysicalExecutionPlan
 ↓
PhysicalHardwareStation
 ↓
FPGA
 ↓
Digital Routing Fabric
 ↓
Real Digital IC
 ↓
Capture / Telemetry
 ↓
Unified Result Model
 ↓
WebSocket
 ↓
Browser Instruments
```

### Physical Analog Future

Khi phần cứng analog được phát triển về sau:

```text
ValidatedGraph
      ↓
Analog Physical Compiler
      ↓
Analog ERC
      ↓
Analog Routing Profile
      ↓
Future Physical Analog Hardware
```

Mini PC vẫn dùng các abstraction hiện có:

```text
HardwareStation
StationCapabilities
ExecutionPlan
Unified Result Model
```

Do đó không cần thay đổi kiến trúc nền của Mini PC.

---

# 25. Physical Compiler

Physical Compiler phiên bản V1 xử lý **digital physical execution**.

Đầu vào không chỉ có logical graph mà còn có topology thực tế đã tự dò:

```text
ValidatedGraph
      +
DetectedTopologyGraph
      +
HardwareProfile
      ↓
Physical Compiler
```

Physical Compiler chịu trách nhiệm:

```text
logical component
        ↓
resolved physical placement
        ↓
breadboard electrical node
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

## 31.1 Phạm vi Physical Circuit hiện tại

FPGA phiên bản V1 chỉ chịu trách nhiệm thực thi **mạch số vật lý**.

```text
Physical Digital Circuit → SUPPORTED
Physical Analog Circuit  → FUTURE
```

Điều này không ngăn Mini PC mô phỏng analog.

## 31.2 Các chức năng FPGA V1/Final Digital Platform

```text
Clock Generation
Deterministic Switching
Timestamping
Digital Routing Control
Logic Capture
Trigger
Digital IO
Topology Scan Coordination
DDS / Function Generator Control
ADC Interface / Oscilloscope Acquisition
SDRAM Controller
Acquisition
```

Oscilloscope và Function Generator là **instrument subsystem**:
- Tín hiệu analog đi qua ADC/DAC/AFE bên ngoài.
- FPGA chỉ xử lý control, timing, DDS, trigger, acquisition và buffering.
- Việc có ADC/DAC instrument không đồng nghĩa FPGA V1 đã hỗ trợ arbitrary analog circuit routing.

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

Digital Virtual Simulation
Analog Virtual Simulation
Mixed-Signal Coordination

Digital Physical Compiler V1
Future Analog Physical Compiler Adapter

Topology Discovery Orchestration
Topology Matching
Routing Solver
Experiment Lifecycle
Session / Lease / Queue
Capture Management
Waveform Processing
Hardware Coordination
Storage
Logging
```

Nguyên tắc quan trọng:

> Việc FPGA V1 chỉ hỗ trợ physical digital circuit không làm thay đổi kiến trúc Mini PC. Khi phần cứng analog được thêm sau này, chỉ cần thêm capability, compiler adapter và driver tương ứng.

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

virtualDigital = true
virtualAnalog = true

physicalDigital = true
physicalAnalogRouting = false   # V1

physicalExecutionSlots = 1      # mặc định với một shared physical fabric

logicAnalyzerChannels
oscilloscopeChannels = 2
generatorChannels = 2

maxSampleRate
routingResources
topologyScanSupported = true
supportedComponents
supportedFeatures
firmwareVersion
```

Frontend và Experiment Manager tự điều chỉnh behavior dựa trên capabilities.

Khi nâng cấp hardware analog trong tương lai:

```text
physicalAnalogRouting = true
```

mà không phải thay đổi kiến trúc frontend/Mini PC.

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

Phiên bản hiện tại hỗ trợ:

> **2 Oscilloscope Channels**

```text
CH1 Probe ─→ AFE1 ─→ ADC CH1 ─┐
                               │
CH2 Probe ─→ AFE2 ─→ ADC CH2 ─┼─→ FPGA Acquisition
                               │
                               ▼
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

FPGA chịu trách nhiệm:

```text
sample timing
trigger
timestamp
channel synchronization
buffer control
SDRAM arbitration
capture state machine
```

AFE và ADC nằm ngoài FPGA.

Không stream toàn bộ raw ADC liên tục lên browser; chỉ truyền capture/window cần thiết.

---

# 36. Function Generator Architecture

Phiên bản hiện tại hỗ trợ:

> **2 Function Generator Channels**

```text
                     ┌─→ DDS CH1 ─→ DAC1 ─→ Filter/Output Stage ─→ OUT1
Browser Control ─→ FPGA
                     └─→ DDS CH2 ─→ DAC2 ─→ Filter/Output Stage ─→ OUT2
```

Luồng điều khiển:

```text
Browser
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

FPGA quản lý riêng từng channel:

```text
frequency
phase
waveform
timing
trigger synchronization
enable/disable
```

DAC, reconstruction filter và output amplifier nằm ngoài FPGA.

## 36.1 FPGA Instrument Resource Architecture

Oscilloscope 2CH, Function Generator 2CH và Logic Analyzer phải là các module độc lập.

```text
                    EP4CE10
┌────────────────────────────────────────────┐
│ Protocol / Control                         │
│        │                                   │
│        ├──────────── Instrument Bus ────────┤
│        │              │          │         │
│        ▼              ▼          ▼         │
│ Logic Analyzer   Scope 2CH    DDS 2CH      │
│        │              │          │         │
│        └──────┬───────┘          │         │
│               ▼                  │         │
│          SDRAM Arbiter           │         │
│               ↓                  │         │
│        SDRAM Controller          │         │
│                                  │         │
│ Clock / PLL / Trigger / Timestamp│         │
└────────────────────────────────────────────┘
          │                         │
       ADC/AFE                    DAC/AFE
```

### SDRAM arbitration

Không cho Scope, Logic Analyzer và Host Readback tự điều khiển SDRAM độc lập.

```text
Scope Capture ──────┐
Logic Analyzer ─────┼──→ SDRAM Arbiter ─→ SDRAM
Host Readback ──────┘
```

### Clock Domain Crossing

Các clock domain phải tách rõ:

```text
System Clock
ADC Sample Clock
DDS Clock
SDRAM Clock
USB Interface Clock
```

Sử dụng CDC FIFO/synchronizer tại các boundary.

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

Mục tiêu:

> **30–50 users có thể kết nối và sử dụng hệ thống.**

Cần phân biệt rõ:

```text
30–50 Connected Users
        ≠
30–50 Simultaneous Physical Executions
```

## 41.1 Virtual Users

```text
30–50 users
    ↓
independent logical sessions
    ↓
Simulation Manager
    ↓
Process Worker Pool
```

Virtual simulation có thể chạy song song trong giới hạn CPU/RAM.

## 41.2 Physical Users

Nhiều người dùng có thể đồng thời:

- đăng nhập;
- chỉnh sửa circuit;
- validate;
- chuẩn bị experiment;
- xem trạng thái;
- gửi yêu cầu Run.

Nhưng FPGA chỉ nhận số lượng experiment vật lý tương ứng với:

```text
StationCapabilities.physicalExecutionSlots
```

Với một shared breadboard/routing fabric V1:

```text
physicalExecutionSlots = 1
```

Do đó:

```text
30–50 Physical Users
        ↓
Experiment Scheduler
        ↓
Run Queue
        ↓
Active Execution Slot
        ↓
FPGA
```

Việc người dùng không bấm `Run` cùng một thời điểm giúp giảm contention thực tế, nhưng kiến trúc **không được dựa vào giả định này để đảm bảo correctness**.

Nếu nhiều người bấm Run gần như đồng thời:

```text
Admission Control
      ↓
Queue
      ↓
Serialized Physical Execution
```

FPGA xử lý experiment đang active; Mini PC chịu trách nhiệm scheduling, ownership, timeout và queue.

## 41.3 Khả năng mở rộng tương lai

Nếu FPGA + routing hardware sau này được partition thành nhiều vùng điện cách ly:

```text
physicalExecutionSlots = N
```

thì Scheduler có thể chạy N experiment vật lý đồng thời mà frontend không đổi.

---

# 42. Station Lease & Run Scheduler

Mỗi experiment vật lý có lease ngắn hạn.

```text
User A ─┐
User B ─┼──→ Run Queue ─→ Execution Slot 0 ─→ FPGA
User C ─┘
```

Lifecycle:

```text
QUEUED
  ↓
RESERVED
  ↓
TOPOLOGY_SCAN
  ↓
VALIDATING
  ↓
CONFIGURING
  ↓
RUNNING
  ↓
CAPTURING
  ↓
COMPLETE
  ↓
SAFE_STATE
  ↓
RELEASE
```

Nếu client mất kết nối hoặc timeout:

```text
STOP
 ↓
SAFE_STATE
 ↓
release routing/resources
 ↓
next queued experiment
```

### Scheduler policy V1

Ưu tiên đơn giản:

```text
FIFO
+
per-user rate limit
+
maximum execution time
```

Không cần message broker trong V1.

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
| Digital Simulation | Python Discrete Event Engine |
| Analog Simulation | SPICE-compatible Adapter on Mini PC |
| Simulation Concurrency | Process Worker Pool |
| Database | PostgreSQL |
| Realtime Browser | WebSocket |
| Waveform Transport | WebSocket Binary |
| Reverse Proxy | Nginx |
| Service Management | systemd |
| Prototype FPGA | EP4CE6 |
| Final FPGA | EP4CE10 + SDRAM |
| Physical Circuit V1 | Digital only |
| Oscilloscope | 2 channels, external AFE/ADC |
| Function Generator | 2 channels, external DAC/output stage |
| Breadboard Discovery | Active Electrical Topology Discovery |
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
SignalDomain
ExecutionPlan
HardwareStation
StationCapabilities
Unified Result Model
BreadboardNodeMap
DetectedTopologyGraph
```

---

## Phase B — Virtual Digital Circuit Engine

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
Simulation Worker Pool
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

## Phase C2 — Mini PC Analog Simulation Foundation

Bổ sung mà không thay đổi frontend architecture:

```text
SignalDomain
Analog Compiler
SPICE-compatible Adapter
Analog Waveform Adapter
Mixed-Signal Coordinator boundary
```

Mục tiêu:

```text
Virtual Digital → supported
Virtual Analog  → supported
Physical Digital → target V1
Physical Analog → future hardware
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

## Phase H — Breadboard Auto-Discovery + Physical Digital Routing

Triển khai:

```text
BreadboardNodeMap
Scan Matrix
Safe Stimulus Front-End
High-Impedance Sense
DetectedTopologyGraph
Component Signature Library
Topology Matcher

MUX / Crosspoint Digital Routing
Hardware Profile
Physical Compiler
Digital ERC
Routing Solver
```

Mục tiêu:

> Sinh viên không cần lắp linh kiện đúng vị trí giáo viên; hệ thống so sánh topology điện đã tự dò.

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

## Phase K — Function Generator 2CH

Thêm:

```text
DDS CH1
DDS CH2
DAC CH1
DAC CH2
frequency
phase
waveform
synchronization
output enable
```

---

## Phase L — Oscilloscope 2CH

Thêm:

```text
AFE CH1 / CH2
ADC CH1 / CH2
shared trigger engine
channel synchronization
SDRAM capture
SDRAM arbiter
decimation
waveform transport
```

---

## Phase M — Multi-user Physical Lab

Mục tiêu:

```text
30–50 connected users
1 physical execution slot V1
burst Run requests serialized by scheduler
```

Thêm:

```text
Reservation
Lease
Run Queue
Admission Control
Execution Slot
Timeout
Permission
Recovery
Audit
```

Sau này nếu hardware được partition an toàn:

```text
physicalExecutionSlots > 1
```

mà không thay đổi frontend architecture.

---

## Phase N — Future Physical Analog Hardware

Không nằm trong FPGA/Physical V1.

Khi phát triển:

```text
Analog Routing Fabric
Analog Switch / Relay Matrix
Analog ERC
Calibration Profiles
Future FPGA/AFE Extensions
```

Mini PC giữ nguyên:

```text
CircuitGraph
SignalDomain
Analog Compiler Interface
HardwareStation
StationCapabilities
Unified Result Model
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
                 ┌───────────┴─────────────┐
                 │                         │
                 ▼                         ▼
          VIRTUAL EXECUTION         PHYSICAL EXECUTION V1
                 │                         │
         SignalDomain Dispatch             ▼
          ┌──────┴──────┐              SAFE_STATE
          │             │                   │
          ▼             ▼                   ▼
 Digital Compiler   Analog Compiler   Electrical Topology Scan
          │             │                   │
          ▼             ▼                   ▼
 Discrete Event       SPICE         DetectedTopologyGraph
 Simulator            Adapter               │
          │             │                   ▼
          └──────┬──────┘             Topology Matcher
                 │                         │
                 ▼                         ▼
      VirtualHardwareStation        Digital Physical Compiler
                 │                         │
                 │                         ▼
                 │                   Digital ERC
                 │                         │
                 │                         ▼
                 │                   Routing Solver
                 │                         │
                 │                         ▼
                 │                PhysicalExecutionPlan
                 │                         │
                 │                         ▼
                 │                PhysicalHardwareStation
                 │                         │
                 │                         ▼
                 │                      EP4CE10
                 │                ┌────────┼─────────┐
                 │                │        │         │
                 │                ▼        ▼         ▼
                 │             Digital   Scope     Generator
                 │             Routing    2CH        2CH
                 │                │        │         │
                 │                ▼       ADC       DAC
                 │            Real Digital ICs      │
                 │                │                 │
                 └────────────────┴────────┬────────┘
                                          ▼
                                Unified Result Model
                                          │
                                          ▼
                                  WebSocket / Binary
                                          │
                                          ▼
                                         UI
```

### Future Physical Analog

```text
ValidatedGraph
      ↓
Analog Physical Compiler
      ↓
Analog ERC
      ↓
Future Analog Routing / AFE Hardware
      ↓
PhysicalHardwareStation
```

Đây là phần mở rộng hardware; không yêu cầu thay đổi kiến trúc nền Mini PC.

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
19. Breadboard dùng **Active Electrical Topology Discovery** để tự xác định topology vật lý; không bắt sinh viên đặt linh kiện giống vị trí giáo viên.
20. Mini PC hỗ trợ kiến trúc **Digital + Analog Simulation**, trong khi Physical FPGA V1 chỉ chạy mạch số.
21. Analog physical hardware tương lai được thêm qua capability/compiler/driver mới, không rewrite Mini PC/frontend.
22. 30–50 users được hỗ trợ ở mức kết nối/session; physical execution V1 dùng queue + execution slot thay vì giả định 30–50 Run đồng thời.
23. `physicalExecutionSlots` là capability; V1 mặc định bằng 1.
24. EP4CE10 quản lý **Oscilloscope 2CH + Function Generator 2CH**, dùng AFE/ADC/DAC/SDRAM bên ngoài.
25. Scope, Logic Analyzer và Host Readback dùng **SDRAM Arbiter** chung.

## Cấu hình chức năng V1 được chốt

```text
Mini PC:
  Virtual Digital      = YES
  Virtual Analog       = YES
  30–50 user sessions  = TARGET

FPGA / Physical:
  Digital Circuit      = YES
  Analog Circuit       = NO (future)
  Auto Topology Scan   = YES
  Oscilloscope         = 2 channels
  Function Generator   = 2 channels
  Logic Analyzer       = YES
  External SDRAM       = YES
  Physical Exec Slots  = 1 by default
```

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
