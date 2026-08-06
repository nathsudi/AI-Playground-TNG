# Architecture Overview

## System Design Philosophy

AI Playground is a **multi-process desktop application** that orchestrates AI inference on local hardware. The key architectural decisions:

1. **Process isolation** - Each AI engine runs as a separate OS process for crash isolation
2. **Platform abstraction in TypeScript** - All Linux-specific logic lives in the Electron main process (zero shell scripts)
3. **Runtime dependency installation** - Python backends are NOT pre-installed; `uv` creates venvs on first launch
4. **Loopback-only security** - All local APIs are bound to 127.0.0.1 with per-session HMAC tokens

---

## High-Level Architecture Diagram

```mermaid
graph TB
    subgraph "User Interface"
        R[Vue.js Renderer<br/>Vite + TailwindCSS]
    end

    subgraph "Electron Main Process"
        M[Main Process<br/>electron/main.ts]
        P[Preload Bridge<br/>electron/preload.ts]
        LC[LangChain Subprocess<br/>subprocesses/langchain.ts]
        SR[Service Registry<br/>apiServiceRegistry.ts]
        UV[UV Manager<br/>uvBasedBackends/uv.ts]
        DD[Device Detection<br/>deviceDetection.ts]
        LPI[Linux Package Installer<br/>linuxPackageInstaller.ts]
        AR[AIPG Root Manager<br/>aipgRoot.ts]
    end

    subgraph "Backend Services (Separate Processes)"
        AI[AI Backend<br/>Flask - Port 59000-59999<br/>Model Downloads]
        LLAMA[LlamaCPP Backend<br/>llama-server binary<br/>Port 39000-39999]
        OV[OpenVINO Backend<br/>OVMS binary<br/>Port 29000-29999]
        CUI[ComfyUI Backend<br/>Python - torch.xpu<br/>Port 49000-49999]
        HA[Home Agent Backend<br/>Flask - Port 58000-58999]
    end

    subgraph "External Resources"
        HF[HuggingFace Hub<br/>Model Downloads]
        GH[GitHub Releases<br/>Binary Downloads]
    end

    R <-->|IPC via preload| P
    P <-->|ipcMain.handle| M
    M --> SR
    SR --> AI
    SR --> LLAMA
    SR --> OV
    SR --> CUI
    SR --> HA
    M --> UV
    M --> DD
    M --> LPI
    M --> AR
    M --> LC
    AI --> HF
    LLAMA --> GH
    OV --> GH
    CUI --> GH
```

---

## Component Interaction Flow

```mermaid
sequenceDiagram
    participant User
    participant Renderer as Vue.js Renderer
    participant Preload as Preload Bridge
    participant Main as Electron Main
    participant Registry as Service Registry
    participant UV as UV Manager
    participant Backend as Backend Process

    User->>Renderer: Interact with UI
    Renderer->>Preload: window.electronAPI.method()
    Preload->>Main: ipcRenderer.invoke('channel')
    Main->>Registry: getService('backend-name')
    
    alt Service not set up
        Main->>UV: installBackend('backend')
        UV->>UV: uv venv (create virtualenv)
        UV->>UV: uv sync (install dependencies)
        UV-->>Main: Installation complete
    end
    
    Main->>Registry: service.start()
    Registry->>Backend: spawn process (dynamic port)
    Backend-->>Registry: Health check OK
    Registry-->>Main: Service running
    Main-->>Preload: {port, token, status}
    Preload-->>Renderer: Result
    Renderer->>Backend: HTTP REST API (X-AIPG-Auth header)
    Backend-->>Renderer: Response/SSE stream
```

---

## Application Startup Sequence (Linux)

```mermaid
flowchart TD
    A[App Launch] --> B{Is Linux?}
    B -->|Yes| C[Disable HW Acceleration]
    C --> D[Add --no-sandbox flag]
    D --> E{Is Packaged?}
    E -->|Yes| F[Seed writable resources<br/>from read-only bundle<br/>to ~/.local/share/ai-playground]
    E -->|No| G[Use project directory directly]
    F --> H[Initialize Service Registry]
    G --> H
    H --> I[Assign dynamic ports<br/>ai-backend: 59000-59999<br/>openvino: 29000-29999<br/>comfyui: 49000-49999<br/>llamacpp: 39000-39999]
    I --> J[Check which services are set up]
    J --> K[Auto-start all set-up services]
    K --> L[Create BrowserWindow]
    L --> M[Load Vue.js renderer]
    M --> N[App Ready]
```

---

## Process Architecture (Linux Runtime)

```mermaid
graph LR
    subgraph "PID 1: Electron Main"
        E[AI Playground<br/>--no-sandbox]
    end

    subgraph "Renderer Processes"
        R1[Chrome Renderer<br/>Vue.js UI]
    end

    subgraph "Backend Processes"
        B1[python3.12<br/>Flask AI Backend]
        B2[llama-server<br/>Vulkan GPU / CPU]
        B3[ovms<br/>OpenVINO Model Server]
        B4[python3.12<br/>ComfyUI + torch.xpu]
        B5[python3.12<br/>Home Agent]
    end

    subgraph "Utility Process"
        U1[Node.js<br/>LangChain Worker]
    end

    E --> R1
    E --> B1
    E --> B2
    E --> B3
    E --> B4
    E --> B5
    E --> U1
```

---

## Data Flow: Model Inference Request

```mermaid
sequenceDiagram
    participant UI as Vue.js UI
    participant Main as Electron Main
    participant LLama as llama-server

    UI->>Main: getBackendAuthToken('llamacpp-backend')
    Main-->>UI: token (32 random bytes, hex)
    
    UI->>LLama: POST /v1/chat/completions<br/>Headers: X-AIPG-Auth: <token>
    
    alt Streaming
        LLama-->>UI: SSE: data: {"choices":[...]}
        LLama-->>UI: SSE: data: {"choices":[...]}
        LLama-->>UI: SSE: data: [DONE]
    else Non-streaming
        LLama-->>UI: JSON: {"choices":[...]}
    end
```

---

## Data Flow: Model Download

```mermaid
sequenceDiagram
    participant UI as Vue.js UI
    participant Main as Electron Main
    participant AIBackend as AI Backend (Flask)
    participant HF as HuggingFace Hub

    UI->>Main: IPC: downloadModel(modelData)
    Main->>AIBackend: POST /api/downloadModel
    AIBackend->>HF: hf_hub_url() + HTTP GET (Range headers)
    
    loop Progress Updates
        HF-->>AIBackend: File chunks
        AIBackend-->>UI: SSE: {progress, speed, downloaded}
    end
    
    AIBackend->>AIBackend: Atomic rename from temp dir
    AIBackend-->>UI: SSE: {status: "completed"}
```

---

## Directory Layout (Key Paths)

```
AI-Playground/
├── WebUI/                          # Electron + Vue.js frontend
│   ├── electron/                   # Main process TypeScript
│   │   ├── main.ts                 # Entry point, IPC handlers
│   │   ├── preload.ts              # Context bridge (~80 IPC methods)
│   │   ├── aipgRoot.ts             # Linux read-only bundle workaround
│   │   └── subprocesses/           # Backend service managers
│   │       ├── apiServiceRegistry.ts
│   │       ├── llamaCppBackendService.ts
│   │       ├── openVINOBackendService.ts
│   │       ├── comfyUIBackendService.ts
│   │       ├── aiBackendService.ts
│   │       ├── homeAgentBackendService.ts
│   │       ├── deviceDetection.ts       # Linux GPU detection
│   │       ├── linuxPackageInstaller.ts # apt-get integration
│   │       └── uvBasedBackends/
│   │           └── uv.ts           # Python env management
│   ├── src/                        # Vue.js renderer (components, stores)
│   ├── build/                      # Build configs and scripts
│   │   ├── build-config.json       # electron-builder config
│   │   └── scripts/                # Build automation
│   ├── external/                   # Bundled configs and catalogs
│   │   ├── backend-versions.json   # Pinned backend versions
│   │   ├── models.json             # Model catalog
│   │   └── model_config.json       # Model path configuration
│   └── package.json                # npm project definition
├── service/                        # Python: model download service
├── comfyui-deps/                   # Python: ComfyUI dependencies + custom nodes
├── home-agent/                     # Python: Telegram/Slack bridge
├── OpenVINO/                       # Python: device detection utility
├── modes/                          # Product mode configs (studio/essentials/nvidia)
└── .python-version                 # Pins Python 3.12.13
```
