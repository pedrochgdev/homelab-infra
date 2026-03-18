# AI Stack

## Summary

The homelab currently supports an AI-related workflow centered on a self-hosted LLM and a personal avatar project. The stack is intentionally split across multiple machines based on hardware capability.

The LLM runs on the homelab, while other components remain on the personal workstation.

## Current Architecture

The AI workflow is currently divided as follows:

### synthia

`synthia` hosts the LLM component of the stack.

Current model:
- `llama3.1:8b-instruct-q5_K_M`

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

- the AI VM is hosted on `virt`
- full GPU passthrough is enabled for `synthia`
- current workload behavior suggests VRAM is the main hardware constraint rather than system RAM

## Current Limitations

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