# End-to-End Workflow Diagrams

## Complete Application Lifecycle (Linux)

```mermaid
flowchart TD
    subgraph "Installation"
        I1[User downloads .AppImage or .deb] --> I2{Which format?}
        I2 -->|AppImage| I3[chmod +x, run directly]
        I2 -->|.deb| I4[sudo dpkg -i<br/>apt installs: libgtk-3-0, libnss3,<br/>pciutils, python3, git]
        I3 --> I5[First Launch]
        I4 --> I5
    end

    subgraph "First Launch"
        I5 --> F1[Shell wrapper passes --no-sandbox]
        F1 --> F2[Electron starts, disables HW accel]
        F2 --> F3{Read-only bundle?}
        F3 -->|Yes| F4[Seed ~/.local/share/ai-playground/<br/>Copy resources, restore chmod]
        F3 -->|No - Dev| F5[Use project dir]
        F4 --> F6[Initialize Service Registry]
        F5 --> F6
        F6 --> F7[Assign dynamic ports to 5 services]
        F7 --> F8[Detect hardware:<br/>lspci, ldconfig, /sys/bus/pci]
        F8 --> F9[Show UI - no backends set up yet]
    end

    subgraph "Backend Setup (one-time per backend)"
        F9 -->|User clicks Setup| S1[Check system deps]
        S1 --> S2{Missing apt packages?}
        S2 -->|Yes| S3[pkexec apt-get install<br/>or terminal sudo fallback]
        S2 -->|No| S4[Continue]
        S3 --> S4
        S4 --> S5[uv venv --project backend<br/>Create Python virtualenv]
        S5 --> S6[uv sync --project backend<br/>--extra xpu/cpu/cuda]
        S6 --> S7[Download native binaries<br/>llama-server / OVMS from GitHub]
        S7 --> S8[Backend is set up ✓]
    end

    subgraph "Normal Usage"
        S8 --> U1[Service auto-starts on boot]
        U1 --> U2[Health check passes]
        U2 --> U3[User interacts with AI features]
        U3 --> U4{Which feature?}
        U4 -->|Chat| U5[LlamaCPP or OpenVINO]
        U4 -->|Image Gen| U6[ComfyUI]
        U4 -->|Telegram/Slack| U7[Home Agent → LLM]
    end
```

---

## Complete Model Download & Inference Pipeline

```mermaid
flowchart TD
    subgraph "1. Model Discovery"
        D1[models.json catalog loaded<br/>from bundle or GitHub] --> D2[UI shows available models<br/>with types and capabilities]
        D2 --> D3[User selects model]
    end

    subgraph "2. Model Download"
        D3 --> DL1[POST /api/downloadModel<br/>to AI Backend Flask service]
        DL1 --> DL2[HfFileSystem: enumerate files in repo]
        DL2 --> DL3{GGUF split shards?}
        DL3 -->|Yes| DL4[Auto-queue all shard files]
        DL3 -->|No| DL5[Queue single file]
        DL4 --> DL6[ThreadPoolExecutor: 4 threads]
        DL5 --> DL6
        DL6 --> DL7[HTTP GET with Range headers<br/>Resume support, retry logic]
        DL7 --> DL8[Write to <sha256>_tmp/ directory]
        DL8 --> DL9{All downloads complete?}
        DL9 -->|No| DL7
        DL9 -->|Yes| DL10[Atomic rename to final path<br/>namespace---repo/filename.gguf]
        DL10 --> DL11[SSE: status=completed]
    end

    subgraph "3. Model Loading"
        DL11 --> ML1[User clicks Start/Chat]
        ML1 --> ML2{Which backend?}
        ML2 -->|LlamaCPP| ML3[Spawn llama-server<br/>--model path.gguf<br/>--port dynamic<br/>--gpu-layers 999]
        ML2 -->|OpenVINO| ML4[Configure OVMS<br/>with model directory]
        ML2 -->|ComfyUI| ML5[Load into ComfyUI<br/>checkpoint/lora pipeline]
        ML3 --> ML6[Poll /health until ready<br/>120 attempts × 1s]
        ML4 --> ML6
        ML5 --> ML6
    end

    subgraph "4. Inference"
        ML6 --> INF1[Model ready for requests]
        INF1 --> INF2[POST /v1/chat/completions<br/>with auth token]
        INF2 --> INF3[SSE stream: token by token]
        INF3 --> INF4[UI renders streaming response]
    end
```

---

## GPU Detection & Backend Selection Decision Tree

```mermaid
flowchart TD
    A[App starts on Linux] --> B[Scan /sys/bus/pci/devices<br/>for Intel GPU: vendor=0x8086, class=0x03*]
    
    B --> C{Intel GPU PCI device found?}
    C -->|No| D[No Intel GPU hardware]
    C -->|Yes| E[Check libze_loader.so<br/>ldconfig -p + hardcoded paths]
    
    E --> F{Level Zero loader found?}
    F -->|No| G[Level Zero not installed]
    F -->|Yes| H[Check libze_intel_gpu.so<br/>GPU driver library]
    
    H --> I{GPU driver found?}
    I -->|No| J[Driver missing]
    I -->|Yes| K[✓ Full Intel XPU stack available]
    
    D --> L[CPU-only mode for all backends]
    G --> L
    J --> L
    
    K --> M{Which backend needs GPU?}
    M -->|ComfyUI| N[uv sync --extra xpu<br/>PyTorch XPU + Level Zero]
    M -->|OpenVINO| O[OVMS with GPU device<br/>OPENVINO_DEVICE=GPU]
    
    A --> P[Check libvulkan.so<br/>ldconfig -p + hardcoded paths]
    P --> Q{Vulkan loader found?}
    Q -->|Yes| R[LlamaCPP: ubuntu-vulkan-x64<br/>GPU via Vulkan compute]
    Q -->|No| S[LlamaCPP: ubuntu-x64<br/>CPU-only inference]
```

---

## Python Environment Management Workflow

```mermaid
flowchart TD
    subgraph "Setup Phase"
        A[Backend setup requested] --> B[Locate bundled uv binary<br/>build/resources/uv.exe]
        B --> C[uv venv --project backend<br/>--allow-existing --relocatable]
        C --> D[Creates backend/.venv/]
        D --> E{Which install method?}
        E -->|Standard| F[uv sync --project backend]
        E -->|With extra| G[uv sync --project backend<br/>--extra xpu]
        E -->|Flexible| H[uv pip install -r requirements.txt]
        F --> I[All packages from uv.lock installed]
        G --> I
        H --> I
    end

    subgraph "Verification Phase"
        I --> J[uv sync --check --project backend<br/>--output-format json]
        J --> K{Exit code 0?}
        K -->|Yes| L[Environment in sync ✓]
        K -->|No| M{What's wrong?}
        M -->|action=create| N[Venv doesn't exist]
        M -->|action=sync| O[Packages out of date]
        N --> P[Re-run setup]
        O --> P
    end

    subgraph "Error Recovery"
        I -->|Hash mismatch| Q[Detected cache corruption]
        Q --> R[Log warning]
        R --> S[Retry with --no-cache flag]
        S --> I
    end
```

---

## Service Registry: Port Assignment & Startup

```mermaid
sequenceDiagram
    participant App as Electron App
    participant Reg as ApiServiceRegistry
    participant GP as get-port
    participant AI as AI Backend
    participant OV as OpenVINO
    participant CUI as ComfyUI
    participant LC as LlamaCPP
    participant HA as Home Agent

    App->>Reg: aiplaygroundApiServiceRegistry(win, settings)
    
    par Port Assignment
        Reg->>GP: getPort({port: portNumbers(59000, 59999)})
        GP-->>Reg: 59042
        Reg->>GP: getPort({port: portNumbers(29000, 29999)})
        GP-->>Reg: 29001
        Reg->>GP: getPort({port: portNumbers(49000, 49999)})
        GP-->>Reg: 49000
        Reg->>GP: getPort({port: portNumbers(39000, 39999)})
        GP-->>Reg: 39015
        Reg->>GP: getPort({port: portNumbers(58000, 58999)})
        GP-->>Reg: 58000
    end

    Reg->>Reg: Register all services

    Note over Reg: startAllSetUpServices()
    
    par Check setup status
        Reg->>AI: serviceIsSetUp()?
        AI-->>Reg: true
        Reg->>OV: serviceIsSetUp()?
        OV-->>Reg: true
        Reg->>CUI: serviceIsSetUp()?
        CUI-->>Reg: false (not set up)
        Reg->>LC: serviceIsSetUp()?
        LC-->>Reg: true
    end

    par Auto-start set-up services
        Reg->>AI: detectDevices() then start()
        Reg->>OV: detectDevices() then start()
        Reg->>LC: detectDevices() then start()
    end

    Note over CUI: Skipped - not set up
```

---

## Linux Package Installation Decision Flow

```mermaid
flowchart TD
    A[Backend needs system library<br/>e.g. libze_loader.so missing] --> B[Build alternatives list<br/>e.g. libze1 OR level-zero]
    
    B --> C[For each alternative set:<br/>apt-cache show pkg]
    C --> D[Pick first available pkg name]
    D --> E[Check which are already installed<br/>dpkg-query -W]
    E --> F{Any missing?}
    
    F -->|No| G[All deps satisfied ✓]
    F -->|Yes| H{command -v apt-get?}
    
    H -->|No| I[Error: not an apt-based system<br/>User must install manually]
    H -->|Yes| J{command -v pkexec?}
    
    J -->|Yes| K[pkexec bash -lc<br/>apt-get install -y pkg1 pkg2<br/>Shows graphical auth dialog]
    J -->|No| L[x-terminal-emulator -e bash -lc<br/>sudo apt-get install -y pkg1 pkg2<br/>Opens terminal for password]
    
    K --> M{Exit code 0?}
    L --> M
    M -->|Yes| G
    M -->|No| N[Installation failed<br/>Show error to user]
```

---

## Remote Version Update Flow

```mermaid
flowchart TD
    A[App starts] --> B[Read local backend-versions.json<br/>comfyui: v0.25.1<br/>llamacpp: b9763<br/>openvino: 2026.3.0]
    
    B --> C[Fetch remote backend-versions.json<br/>from GitHub raw.githubusercontent.com]
    
    C --> D{Network available?}
    D -->|No| E[Use local versions]
    D -->|Yes| F[Compare remote vs local]
    
    F --> G{Any newer versions?}
    G -->|No| E
    G -->|Yes| H[Store updated versions in memory]
    
    H --> I{Backend already set up?}
    I -->|No| J[Will install new version when set up]
    I -->|Yes| K{Current installed matches remote?}
    K -->|Yes| L[Up to date ✓]
    K -->|No| M[Mark as needing update<br/>Show update available in UI]
    
    M --> N[User triggers update]
    N --> O[Download new binary version]
    O --> P[Write installed-version.json marker]
```

---

## Complete Build-to-Distribution Pipeline

```mermaid
flowchart TD
    subgraph "Developer Machine"
        A[git clone + cd WebUI] --> B[npm run setup]
        B --> C[Development: npm run dev]
        C --> D[Make changes, test locally]
        D --> E[Pre-commit hooks run:<br/>Ruff + ESLint + Prettier]
        E --> F[git commit + push]
    end

    subgraph "CI/CD (GitHub Actions)"
        F --> G[Tag push: v3.1.2-beta]
        G --> H[build-installer.yml triggers]
        H --> I1[Linux runner: ubuntu-latest]
        H --> I2[Windows runner]
        
        I1 --> J1[npm ci --legacy-peer-deps]
        J1 --> K1[npm run fetch-external-resources<br/>Download linux uv + 7zip]
        K1 --> L1[npm run build:linux]
        
        L1 --> M1[vue-tsc: type check]
        M1 --> N1[vite build: compile TS + bundle Vue]
        N1 --> O1[electron-builder --linux --x64]
        O1 --> P1[after-pack.cjs: inject --no-sandbox wrapper]
        P1 --> Q1[Output: AppImage + .deb]
    end

    subgraph "Distribution"
        Q1 --> R[GitHub Release created]
        R --> S1[AI Playground-3.1.2-beta.AppImage]
        R --> S2[AI Playground-3.1.2-beta.deb]
    end

    subgraph "User Machine"
        S1 --> T1[chmod +x, run directly]
        S2 --> T2[sudo dpkg -i package.deb]
        T1 --> U[First launch: seed resources]
        T2 --> U
        U --> V[App ready to use]
    end
```

---

## Data Persistence & Storage Layout

```mermaid
flowchart TD
    subgraph "Read-Only Bundle (AppImage/deb install)"
        RO1[app.asar - Electron app code]
        RO2[resources/ - Initial seed data]
    end

    subgraph "Writable: ~/.local/share/ai-playground/resources/"
        W1[.aipg-seed-version - Version marker]
        W2[uv.exe - Package manager binary]
        W3[7zr.exe - Archive tool]
        W4[backend-versions.json]
        W5[models.json - Model catalog]
        W6[settings.json - User settings]
        
        subgraph "Python Environments"
            PE1[python-interpreter/ - Managed CPython 3.12]
            PE2[service/.venv/ - AI backend venv]
            PE3[comfyui-deps/.venv/ - ComfyUI venv]
            PE4[home-agent/.venv/ - Home Agent venv]
        end

        subgraph "Backend Installations"
            BI1[llamacpp-server/ - llama-server binary]
            BI2[openvino-server/ - OVMS binary]
            BI3[ComfyUI/ - Git clone + custom nodes]
        end
    end

    subgraph "User-Configurable: ~/models/ (default)"
        M1[LLM/ggufLLM/ - GGUF model files]
        M2[LLM/openVINO/ - OpenVINO IR models]
        M3[ComfyUI/models/ - Diffusion checkpoints]
        M4[embeddings/ - Embedding models]
    end

    RO2 -->|Seeded on first launch| W1
```

---

## Error Recovery Patterns

```mermaid
flowchart TD
    subgraph "UV Hash Mismatch Recovery"
        HM1[uv sync fails with hash mismatch] --> HM2[Detect via regex on error message]
        HM2 --> HM3[Log warning about cache corruption]
        HM3 --> HM4[Retry: uv sync --no-cache]
        HM4 --> HM5{Success?}
        HM5 -->|Yes| HM6[Continue normally]
        HM5 -->|No| HM7[Surface error to user]
    end

    subgraph "Backend Crash Recovery"
        CR1[Backend process exits unexpectedly] --> CR2[on-exit handler fires]
        CR2 --> CR3[Log exit code + stderr]
        CR3 --> CR4[Set status = 'failed']
        CR4 --> CR5[Send serviceInfoUpdate to renderer]
        CR5 --> CR6[UI shows error state]
        CR6 --> CR7[User can click Retry]
        CR7 --> CR8[service.start() again]
    end

    subgraph "Network Failure Recovery"
        NF1[Model download fails mid-stream] --> NF2[Download uses HTTP Range headers]
        NF2 --> NF3[On retry: resume from last byte]
        NF3 --> NF4[Temp dir preserved between retries]
    end

    subgraph "System Package Failure"
        SP1[pkexec install fails] --> SP2{Exit code?}
        SP2 -->|126 - dismissed| SP3[User cancelled auth dialog]
        SP2 -->|!= 0| SP4[Package not available or broken deps]
        SP3 --> SP5[Try terminal fallback]
        SP4 --> SP6[Parse apt output for specifics]
        SP6 --> SP7[Show actionable error to user]
    end
```
