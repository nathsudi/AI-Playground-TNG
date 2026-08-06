# Key File Reference

Quick lookup table of every important file and its role in the Linux implementation.

---

## Electron Main Process

| File | Purpose | Linux Relevance |
|------|---------|-----------------|
| `WebUI/electron/main.ts` | App entry point, IPC handlers, window creation | Lines 197-203: disables GPU, adds --no-sandbox |
| `WebUI/electron/preload.ts` | Context bridge exposing ~80 IPC methods | Platform-agnostic |
| `WebUI/electron/aipgRoot.ts` | Resolves writable resources root | Core Linux: handles read-only AppImage/deb |
| `WebUI/electron/remoteUpdates.ts` | Fetches backend version updates from GitHub | Platform-agnostic |
| `WebUI/electron/pathsManager.ts` | Scans filesystem for downloaded models | Platform-agnostic |

---

## Backend Service Managers

| File | Purpose | Linux Specifics |
|------|---------|-----------------|
| `WebUI/electron/subprocesses/apiServiceRegistry.ts` | Central registry, port assignment, lifecycle | Platform-agnostic |
| `WebUI/electron/subprocesses/service.ts` | Base service class, process spawning, health checks | Git path: `/usr/bin/git` on Linux |
| `WebUI/electron/subprocesses/llamaCppBackendService.ts` | LlamaCPP binary management | Vulkan detection → build selection |
| `WebUI/electron/subprocesses/openVINOBackendService.ts` | OVMS binary management | libpython3.12, LD_LIBRARY_PATH setup |
| `WebUI/electron/subprocesses/comfyUIBackendService.ts` | ComfyUI Python process management | Level Zero detection, oneAPI paths |
| `WebUI/electron/subprocesses/aiBackendService.ts` | Flask model download service | Standard uv-based setup |
| `WebUI/electron/subprocesses/homeAgentBackendService.ts` | Home Agent service management | Standard uv-based setup |

---

## Linux-Specific Modules

| File | Purpose | Key Functions |
|------|---------|---------------|
| `WebUI/electron/subprocesses/deviceDetection.ts` | GPU/runtime detection | `linuxHasLevelZeroRuntime()`, `linuxHasVulkanLoader()`, `linuxHasIntelGpuPciDevice()` |
| `WebUI/electron/subprocesses/linuxPackageInstaller.ts` | apt-get integration | `hasAptGet()`, `resolvePackageList()`, `runPkexecInstall()`, `waitForTerminalInstall()` |
| `WebUI/electron/subprocesses/hardwareDiscovery.ts` | GPU enumeration via lspci | `detectIntelGpusViaLspci()` |
| `WebUI/electron/subprocesses/tools.ts` | Archive extraction, permissions | `tar -xf` on Linux, `restoreTreeWritePermissions()` |

---

## Python Environment Management

| File | Purpose | Key Functions |
|------|---------|---------------|
| `WebUI/electron/subprocesses/uvBasedBackends/uv.ts` | uv binary wrapper | `ensureManagedPython()`, `installBackend()`, `installBackendWithExtra()`, `checkBackend()`, `checkBackendWithDetails()`, `ensureBackendVenv()` |

---

## Python Backends

### Service (Model Downloads)

| File | Purpose |
|------|---------|
| `service/pyproject.toml` | Dependencies: Flask, apiflask, huggingface_hub |
| `service/uv.lock` | Pinned dependency tree |
| `service/web_api.py` | Flask routes: download, cancel, model checks |
| `service/model_downloader.py` | HFPlaygroundDownloader: multi-threaded downloads |
| `service/model_download_adpater.py` | SSE progress streaming adapter |
| `service/file_downloader.py` | Single-file HTTP downloader with resume |
| `service/config.py` | Model path configuration |
| `service/utils.py` | Model existence checks, SHA256 caching |

### ComfyUI Dependencies

| File | Purpose |
|------|---------|
| `comfyui-deps/pyproject.toml` | 80+ deps with xpu/cpu/cuda extras |
| `comfyui-deps/uv.lock` | Pinned dependency tree (4,049 lines) |
| `comfyui-deps/pyproject-flexible-venv.toml` | Minimal toml for custom git refs |
| `comfyui-deps/custom_nodes/aipg-auth/` | Loopback authentication middleware |
| `comfyui-deps/custom_nodes/OpenAICompatibleImageGen/` | OpenAI API-compatible image gen |
| `comfyui-deps/custom_nodes/OpenVINOImageUpscale/` | RealESRGAN upscaling |
| `comfyui-deps/custom_nodes/SafetyChecker/` | NSFW content detection |

### Home Agent

| File | Purpose |
|------|---------|
| `home-agent/pyproject.toml` | Dependencies: Flask, telegram-bot, slack-bolt |
| `home-agent/uv.lock` | Pinned dependency tree |
| `home-agent/web_api.py` | Flask routes for channel management |
| `home-agent/llm_proxy.py` | Proxies requests to upstream LLM |
| `home-agent/channels/telegram.py` | Telegram bot implementation |
| `home-agent/channels/slack.py` | Slack bolt app implementation |
| `home-agent/channels/audio.py` | Audio I/O channel |
| `home-agent/channels/registry.py` | Channel discovery and lifecycle |

### OpenVINO Utilities

| File | Purpose |
|------|---------|
| `OpenVINO/pyproject.toml` | Single dep: openvino |
| `OpenVINO/uv.lock` | Pinned dependency tree |
| `OpenVINO/detect_devices.py` | Enumerates CPU/GPU/NPU devices |

---

## Build System

| File | Purpose |
|------|---------|
| `WebUI/package.json` | npm project, scripts, deps |
| `WebUI/package-lock.json` | Locked npm dependency tree |
| `WebUI/vite.config.mts` | Vite + Electron + TailwindCSS config |
| `WebUI/build/build-config.json` | electron-builder config (Linux targets) |
| `WebUI/build/scripts/build-paths.mts` | Platform-specific resource URLs |
| `WebUI/build/scripts/fetch-external-resources.mts` | Downloads uv, 7zip binaries |
| `WebUI/build/scripts/after-pack.cjs` | Linux --no-sandbox wrapper injection |
| `WebUI/build/scripts/ensure-electron.mjs` | Electron binary provisioning |
| `WebUI/build/scripts/ensure-precommit-hook.mjs` | Pre-commit hook setup |
| `WebUI/build/scripts/warn-bundled-wheels.mts` | Post-build .whl inclusion check |

---

## Configuration Files

| File | Purpose | When Used |
|------|---------|-----------|
| `WebUI/external/backend-versions.json` | Pinned backend binary versions | Runtime: version check + download |
| `WebUI/external/models.json` | Model catalog (types, capabilities) | Runtime: model selection UI |
| `WebUI/external/model_config.json` | Model storage paths | Production |
| `WebUI/external/model_config.dev.json` | Dev-mode model paths | Development |
| `WebUI/external/hardware-recommendations.json` | GPU → mode mapping | Runtime: auto-detect best mode |
| `WebUI/external/mcp.json` | MCP server definitions | Production |
| `WebUI/external/mcp-dev.json` | Dev-mode MCP servers | Development |
| `WebUI/build/settings.json` | Default app settings | Bundled into package |
| `WebUI/external/settings-dev.json` | Dev-mode settings | Development |
| `.python-version` | Global Python version pin (3.12.13) | uv reads this for resolution |
| `modes/*/mode.json` | Product mode definitions | Runtime: feature gating |

---

## CI/CD & Quality

| File | Purpose |
|------|---------|
| `.github/workflows/build-installer.yml` | Builds Linux + Windows installers |
| `.github/workflows/eslint-prettier.yml` | TypeScript lint CI |
| `.github/workflows/ruff.yml` | Python lint CI |
| `.github/workflows/bandit.yml` | Python security analysis |
| `.github/workflows/trivy.yml` | Vulnerability scanning + SBOM |
| `.github/workflows/stale.yml` | Auto-close stale issues |
| `.github/dependabot.yml` | Automated dependency PRs |
| `.pre-commit-config.yaml` | Local pre-commit hooks |

---

## Documentation

| File | Purpose |
|------|---------|
| `docs/linux-intel-gpu-setup.md` | Intel GPU driver setup guide (Ubuntu 24.04) |
| `CONTRIBUTING.md` | Contribution guidelines |
| `AGENTS.md` | AI agent coding guidelines |

---

## Runtime Paths (Linux Packaged)

| Path | Purpose |
|------|---------|
| `~/.local/share/ai-playground/resources/` | Writable resources root (seeded from bundle) |
| `~/.local/share/ai-playground/resources/python-interpreter/` | Managed CPython installations |
| `~/.local/share/ai-playground/resources/service/.venv/` | AI backend Python venv |
| `~/.local/share/ai-playground/resources/comfyui-deps/.venv/` | ComfyUI Python venv |
| `~/.local/share/ai-playground/resources/home-agent/.venv/` | Home Agent Python venv |
| `~/.local/share/ai-playground/resources/logs/` | Application logs |
| `~/models/` (configurable) | Downloaded AI models |

---

## Environment Variables (Linux Runtime)

| Variable | Set By | Purpose |
|----------|--------|---------|
| `AIPG_LOOPBACK_TOKEN` | Main process | Per-session auth token |
| `UV_NO_ENV_FILE` | uv.ts | Prevent uv reading .env |
| `UV_NO_CONFIG` | uv.ts | Prevent uv reading user config |
| `UV_PYTHON_INSTALL_DIR` | uv.ts | Where managed Python installs |
| `UV_PYTHON_PREFERENCE` | uv.ts | `only-managed` for OVMS |
| `GGML_VK_VISIBLE_DEVICES` | llamaCppBackendService.ts | Vulkan GPU selection |
| `ONEAPI_DEVICE_SELECTOR` | comfyUIBackendService.ts | Level Zero device selection |
| `ZE_FLAT_DEVICE_HIERARCHY` | comfyUIBackendService.ts | Level Zero topology |
| `LD_LIBRARY_PATH` | Various | Shared library resolution |
| `OPENVINO_DEVICE` | openVINOBackendService.ts | OpenVINO device selection |
| `XDG_DATA_HOME` | System | Writable root base path |
