# Third-party notices

## Upstream repo

The StudioClaw team's original repo (linked from the Credits section of README.md) has no licence file, and none of its files has a licence header or copyright notice, so there is no upstream notice to
reproduce here. No other repos are bundled in this copy.

## Software the demo uses (not included)

The demo calls the following software, which you install yourself. None of it is copied into this repo. Each
keeps its own licence and notices, which you can find in that project's own distribution:

| Component | Used for | Where to find it |
|---|---|---|
| OpenClaw 2026.7.1 | Agents, channels, sessions and skills | https://openclaw.ai (npm package `openclaw`) |
| vLLM | Local model server for reasoning and vision | https://github.com/vllm-project/vllm |
| ComfyUI | Local image generation | https://github.com/comfyanonymous/ComfyUI |
| Qwen3.6-35B NVFP4 (`nvidia/Qwen3.6-35B-A3B-NVFP4`) | Reasoning and vision model | The model's own model card and licence |
| SDXL base 1.0 and Z-Image (optional) | Image models loaded by your ComfyUI workflows | Each model's own model card and licence |
| OpenClaw WhatsApp plugin (optional) | WhatsApp channel | `clawhub:@openclaw/whatsapp` |

Before you use a component, check that its licence allows your use, in particular for model weights.
