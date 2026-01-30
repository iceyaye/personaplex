# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

PersonaPlex is an NVIDIA research project implementing a real-time, full-duplex speech-to-speech conversational model with voice and role control. It's a finetuned version of the Moshi architecture for interactive, low-latency voice conversations with consistent persona control.

- **Backend**: Python/PyTorch (moshi package)
- **Frontend**: TypeScript/React with Vite
- **Model**: PersonaPlex 7B v1 from Hugging Face (`nvidia/personaplex-7b-v1`)

## Commands

### Backend (Python)

```bash
# Install package
pip install moshi/.

# For CPU-only evaluation
pip install moshi/. --index-url https://download.pytorch.org/whl/cpu

# Run server (interactive web UI)
SSL_DIR=$(mktemp -d); python -m moshi.server --ssl "$SSL_DIR"

# Run server with CPU offloading (low VRAM)
SSL_DIR=$(mktemp -d); python -m moshi.server --ssl "$SSL_DIR" --cpu-offload

# Offline inference (single file)
python -m moshi.offline --voice-prompt "NATF2.pt" --input-wav "input.wav" --output-wav "output.wav" --text-prompt "Your prompt"
```

### Frontend (client/)

```bash
npm run dev          # Start dev server
npm run build        # Build for production
npm run lint         # ESLint check
npm run lint:fix     # Auto-fix linting
npm run prettier     # Format code
```

### Docker

```bash
docker-compose up    # Full stack with GPU
```

## Architecture

```
personaplex/
├── moshi/                    # Python backend
│   └── moshi/
│       ├── server.py         # WebSocket server (aiohttp) - main entry
│       ├── offline.py        # Batch inference script
│       ├── models/           # ML models (LMModel, MimiModel)
│       │   ├── lm.py         # Language model and generation
│       │   ├── compression.py # Audio codec (Mimi)
│       │   └── loaders.py    # HF hub model loading
│       ├── modules/          # Building blocks
│       │   ├── transformer.py # StreamingTransformer
│       │   ├── seanet.py     # Audio encoder/decoder
│       │   └── streaming.py  # Streaming state management
│       └── utils/            # Logging, sampling, compilation
├── client/                   # React/TypeScript frontend
│   └── src/
│       ├── app.tsx           # Main React app
│       ├── protocol/         # Binary WebSocket protocol
│       │   ├── types.ts      # Message types
│       │   └── encoder.ts    # Binary encoding
│       ├── pages/Conversation/
│       │   ├── Conversation.tsx
│       │   ├── SocketContext.ts
│       │   └── hooks/        # useSocket, useUserAudio, useServerAudio
│       └── decoder/          # Web Worker for Opus decoding
└── assets/test/              # Example audio files
```

## Key Patterns

### Backend
- **Async I/O**: Heavy `asyncio` usage for WebSocket handling
- **Streaming**: Models support incremental generation via `StreamingModule` and `StreamingStateDict`
- **Device handling**: Auto-detection via `torch_auto_device()`

### Frontend
- **Custom hooks**: `useSocket`, `useUserAudio`, `useServerAudio` for state management
- **Web Workers**: Opus decoding in separate thread
- **Binary protocol**: Custom message encoding in `protocol/encoder.ts`
- **Tailwind + daisyUI**: CSS framework

### Audio
- 24kHz Opus streaming over WebSocket
- 16 pre-packaged voices: NATF0-3, NATM0-3 (natural), VARF0-4, VARM0-4 (variety)

## Environment

- Requires `HF_TOKEN` environment variable for model downloads
- Opus codec library required: `libopus-dev` (Linux) or `brew install opus` (macOS)
- Python 3.10+, Node.js for frontend
- GPU recommended; `--cpu-offload` for low VRAM (requires `accelerate` package)

## macOS / Apple Silicon Support

The project supports MPS (Metal Performance Shaders) for GPU acceleration on macOS:

```bash
# Auto-detects MPS on Apple Silicon
python -m moshi.server --ssl "$SSL_DIR"

# Explicit device selection
python -m moshi.server --ssl "$SSL_DIR" --device mps

# Enable fallback for unsupported MPS operations
PYTORCH_ENABLE_MPS_FALLBACK=1 python -m moshi.server --ssl "$SSL_DIR"
```

Requirements:
- macOS 12.3+
- Apple Silicon or AMD GPU
- PyTorch with MPS support (included in standard PyTorch builds)

Note: CUDA graphs are automatically disabled on MPS. CPU offloading (`--cpu-offload`) is not available on MPS.
