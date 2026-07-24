# AI Stack

## Summary

The homelab currently supports an AI-related workflow centered on a self-hosted LLM and a personal avatar project. The stack is intentionally split across multiple machines based on hardware capability.

The LLM runs on the homelab, while other components remain on the personal workstation.

## Current Architecture

The AI workflow is currently divided as follows:

### synthia

`synthia` hosts the LLM component of the stack. It is VMID 100 on `virt`, named
`ia` at the hypervisor level.

Models currently present:

| Model | Size | Notes |
|---|---|---|
| `llama3.1:8b-instruct-q5_K_M` | 5.7 GB | Primary instruct model |
| `Brain:latest` | 5.7 GB | Custom model |
| `avatar_brain:latest` | 4.7 GB | Custom model for the avatar project |
| `llama3:8b` | 4.7 GB | Earlier base model |

Runtime:
- Ollama

Role:
- provide the language model backend for the broader project

### Personal Workstation

The personal workstation hosts the other active components of the project, including:

- avatar rendering with Three.js
- speech-to-text
- text-to-speech
- Whisper
- Kokoro
- associated local integration logic

This separation exists primarily because available hardware does not allow all components to run comfortably on the same system.

## Functional Goal

The purpose of the stack is to support an avatar-oriented AI project in which a rendered character interacts through LLM-backed responses and supporting speech systems.

## Deployment Model

The current model is hybrid:

- LLM hosted on the homelab
- rendering and supporting voice pipeline hosted on the personal workstation

## Infrastructure Notes

- the AI VM is hosted on `virt` as VMID 100
- full GPU passthrough is configured for the RTX 2060 via `hostpci0: 0000:01:00`
- the VM has 2 vCPUs and 4 GB RAM, with 34 GB of 117 GB disk in use

## GPU Status

**The GPU is currently not usable from inside the VM.** `nvidia-smi` fails with:

```
Failed to initialize NVML: Driver/library version mismatch
NVML library version: 580.173
```

This means the kernel module loaded in the guest does not match the installed
userspace driver libraries, which typically happens after a driver package
upgrade without a subsequent reboot. Until this is resolved, Ollama falls back to
CPU inference on 2 vCPUs, which makes response latency far worse than the
hardware allows.

Resolving this is the highest-value fix in the AI stack: the passthrough is
already configured correctly at the hypervisor level, so the problem is confined
to the guest.

## Current Limitations

- GPU acceleration is broken in the guest, so inference runs on CPU
- the VM has only 2 vCPUs, which is a poor fallback for CPU inference
- the stack is split across multiple systems
- not all components are consolidated into the homelab
- hardware capacity limits how much can be moved to a single node
- the architecture remains practical and functional, but still evolving

## Future Directions

Possible future work includes:

- refining the separation of AI components
- improving operational structure around the model-serving layer
- experimenting further with automation and orchestration
- evaluating whether more of the stack can be moved into the homelab over time