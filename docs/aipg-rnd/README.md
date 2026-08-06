# AI Playground - Linux Implementation Analysis

> Comprehensive technical documentation for carrying forward Linux development of Intel AI Playground.

## Document Index

| Document | Description |
|----------|-------------|
| [Architecture Overview](01-architecture-overview.md) | High-level system design, component map, and Mermaid diagrams |
| [Linux Implementation Deep Dive](02-linux-implementation.md) | All Linux-specific code paths, GPU detection, package installation |
| [Backend Services](03-backend-services.md) | How LlamaCPP, OpenVINO, and ComfyUI backends work on Linux |
| [Dependency & Version Management](04-dependency-management.md) | How Python/JS deps are pinned, installed, and updated |
| [Build & Distribution](05-build-distribution.md) | How the Linux AppImage/deb packages are built |
| [IPC & Communication](06-ipc-communication.md) | Electron-to-backend communication, auth tokens, health checks |
| [Development Workflow](07-development-workflow.md) | How to set up, run, and debug on Linux |
| [Key File Reference](08-key-file-reference.md) | Quick lookup table of important files and their roles |

## Quick Context

- **App Version:** 3.1.2-beta
- **Electron:** 42
- **Python:** 3.12.13 (managed via `uv`)
- **Backend Versions:** LlamaCPP b9763 | ComfyUI v0.25.1 | OpenVINO 2026.3.0
- **Linux Targets:** AppImage (portable) + .deb (Debian/Ubuntu)
- **GPU Acceleration:** Vulkan (llama.cpp), Level Zero/XPU (ComfyUI/OpenVINO)
