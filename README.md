# Studio Claw

**Local multi-agent image desk on Dell Pro Precision GB10**

People chat on Discord or WhatsApp, with text or text plus a reference image, and get images back that they can keep refining in conversation. Three local agents handle each request on one Dell Pro Precision GB10: a desk agent files a ticket, an artist agent generates the image through ComfyUI, and an independent reviewer agent checks it before anyone sees it, with reasoning, vision and image generation all running on the GB10 while Discord and WhatsApp only carry the messages. The requester stays in charge: they sign off on each image or ask for changes.

## Watch it first

The overview video is under [Releases](../../releases). It explains what the demo does and how the agents hand work to each other.

## How it works

| Stage | What happens | Check or human gate |
|---|---|---|
| 1. Request | A person posts a request in Discord or WhatsApp, with or without a reference image. The **desk** decides whether it is a new request or a change to an image it already delivered. | If the request is too vague to draw, the desk asks one short question. If it can't tell new from change, it asks: "Change T-… (v3), or start a new one?" |
| 2. Ticket and spec | The desk files a ticket (`ticket.py create`) and writes a spec: the request word for word, 2 to 4 must-haves taken only from what was asked, an avoid list and the reference image. | `ticket.py` allows only valid state changes and refuses anything else. |
| 3. Generate | The **artist** writes a short SDXL prompt, runs `comfy_gen.py` on the local ComfyUI and submits the image. On a change, it starts from the previous version's prompt word for word. | The artist doesn't review its own work. If generation fails twice, the ticket is marked `failed`. |
| 4. Review | The **reviewer** compares the image with the spec, and with the base image when it's a change, then records ACCEPT or REJECT with fixes. | Independent pass/fail checklist (`workspaces/reviewer/AGENTS.md`). A rejected image goes back to the artist with the fixes, up to `TICKET_MAX_ATTEMPTS` (default 3) per version. After that, the best attempt is posted. |
| 5. Deliver | The desk posts the ticket ID, the version and the image. | **Human gate:** the requester replies "accept" to sign off (then the ticket is closed) or says what to change. |
| 6. Iterate | "make the cap blue" or "go back to v1 but make it blue" creates a new version of the same ticket (`ticket.py revise`), and steps 3 to 5 run again. | Versions are unlimited unless you set `TICKET_MAX_VERSIONS`. "history", "status" and "stats" show every version and its verdict. |

```
Discord / WhatsApp ─► DESK  new request? → ticket.py create (v1) → "Filed T-… → artist"
                        │  iteration?    → ticket.py revise --change "…"  (v2, v3, …; "back to v2" works)
                        │  writes a spec: must-haves + avoid list (+ reference, or base image + change)
                        │
                        ├─► ARTIST   comfy_gen.py → ComfyUI → ticket.py submit
                        │
                        ├─► REVIEWER checks the image against the spec (and against the base, on an iteration)
                        │      ACCEPT → ticket.py accept
                        │      REJECT → ticket.py reject --reason "fix 1; fix 2"   (attempt +1)
                        │                 └─► desk re-spawns the artist with the feedback
                        │                      (3 rejects on one version → post best attempt, person keeps iterating)
                        │
                        └─► posts "T-… v2" + image → ticket.py close → "Reply with changes to iterate"

"history" → every version with its verdict   ·   "status" → the board   ·   "stats" → first-try acceptance, attempts, versions
```

Two loops run inside each other, so quality and iteration stay separate:

- **Versions** are the requester's loop: "make it bluer", "go back to v2 but bigger". Each round is a new version of the *same* ticket. It is built from the previous version's prompt and image, so the subject stays recognisable.
- **Attempts** are the reviewer's loop. Within one version, the reviewer can send work back up to `TICKET_MAX_ATTEMPTS` times. If the limit is reached, the version is marked `failed` and the desk posts the best attempt anyway. The limit resets on every new version.

### The three agents

Each agent runs in its own OpenClaw workspace with only the tools it needs, so the roles can't blur.

| Agent | Job | Tools |
|---|---|---|
| `desk` (default) | Talks to people, owns the tickets and runs the artist → reviewer → retry loop. It never draws and never judges. | `read`, `exec`, `view_image`, `message`, `agents_list`, `sessions_spawn`, `sessions_yield`, `subagents` |
| `artist` | Turns one spec into one image through ComfyUI, then submits it. It doesn't review its own work. | `read`, `exec`, `view_image`, `image_generate` |
| `reviewer` | Independently accepts or rejects each image against the spec. It can't generate images. | `read`, `exec`, `view_image` |

| OpenClaw feature | Used for |
|---|---|
| **Channels** (`channels.discord`, WhatsApp plugin) | Receiving messages and attachments, and posting images back |
| **Multi-agent** (`agents.list`, an `AGENTS.md` in each agent's workspace, tool allow/deny) | The three roles, each limited to its own tools |
| **Sub-agents** (`sessions_spawn` / `sessions_yield`) | The desk running the artist → reviewer → retry loop |
| **Sessions** (one per channel) | Conversation memory, so "make it bluer" knows what "it" is |
| **Skills** (`~/.openclaw/skills/tickets`) + `exec` | The ticket store the agents use |
| **Image provider** (`plugins.entries.comfy`) | Local ComfyUI on `127.0.0.1:8188` |
| **Model provider** (`models.providers.vllm`) | Local Qwen3.6 on `127.0.0.1:8000`, for both reasoning *and* vision |

## Hardware

**Built and run on:** every part of the demo was built and run on this Dell system.

| Component | Built and run on |
|---|---|
| System | Dell Pro Precision GB10 |
| GPU | NVIDIA GB10 Grace Blackwell superchip, 128 GB unified LPDDR5x (shared by CPU and GPU) |
| CPU | 20-core NVIDIA Grace Arm CPU |
| Memory | 128 GB unified LPDDR5x (shared by CPU and GPU) |
| Storage | 4 TB NVMe SSD |
| OS | NVIDIA DGX OS |
| Software | OpenClaw 2026.7.1; vLLM serving Qwen3.6-35B NVFP4 (reasoning and vision); ComfyUI; Python SQLite ticket store (stdlib) |

### Also meets the GPU requirement (not tested on these)

The demo needs an NVIDIA GPU with 128 GB of GPU-addressable memory running Linux, in a desktop form factor, to serve Qwen3.6-35B NVFP4 under vLLM alongside ComfyUI. This Dell system meets that requirement but has not been tested with this demo:

| System | Status |
|---|---|
| Dell Pro Precision GB300 | Meets the GPU requirement; not tested |

## What's in this repo

| Path | What |
|---|---|
| `README.md` | This file |
| `LICENSE` | The terms this copy is shared under |
| `THIRD_PARTY_NOTICES.md` | Third-party software the demo uses and its licences |
| `.gitignore` | Keeps models, outputs, `.env` and the generated config out of the repo |
| `ticket-desk/README.md` | Operations guide: install, tuning, WhatsApp, debugging |
| `ticket-desk/setup.sh` | Installs the skill and the workspaces, sets up the ticket database and applies the OpenClaw config |
| `ticket-desk/openclaw.patch.json5` | Config template. `setup.sh` replaces `__HOME__`, `__KIT__` and `__GUILD_ID__` with your own values and writes `openclaw.patch.generated.json5`, which is not committed. |
| `ticket-desk/workspaces/desk/AGENTS.md` | Instructions for the front desk and orchestrator |
| `ticket-desk/workspaces/artist/AGENTS.md` | How the artist writes prompts and submits images |
| `ticket-desk/workspaces/reviewer/AGENTS.md` | The reviewer's ACCEPT/REJECT checklist |
| `ticket-desk/skills/tickets/SKILL.md` | Teaches the agents the ticket commands |
| `ticket-desk/skills/tickets/ticket.py` | SQLite ticket store (Python standard library only) |
| `ticket-desk/skills/tickets/comfy_gen.py` | Queues one image on ComfyUI, waits for it, saves it and prints JSON |

## Setup (Linux, on the Dell Pro Precision GB10)

### Requirements

- A Dell Pro Precision GB10 running Linux (NVIDIA DGX OS), or another system that meets the GPU requirement above
- vLLM serving `nvidia/Qwen3.6-35B-A3B-NVFP4` on port `8000`, for both text **and** vision
- ComfyUI on port `8188`, with an SDXL API workflow (and optionally a Z-Image one) in `$KIT/workflows/` (see Known limits)
- Node.js for `openclaw@2026.7.1`, and Python 3 (standard library only) for the skill
- **A Discord bot token** for a bot with Message Content Intent turned on, and the ID of your Discord server (guild)
- **WhatsApp (optional):** the OpenClaw WhatsApp plugin and a phone you can link

### Steps

1. **Start the local model.** Serve `nvidia/Qwen3.6-35B-A3B-NVFP4` with vLLM on `127.0.0.1:8000`, OpenAI-compatible API at `/v1`, with tool calling on and the `qwen3_xml` tool parser. The config expects these settings:
   - model ID `nvidia/Qwen3.6-35B-A3B-NVFP4`, text and image input (the same model handles vision)
   - API key `vllm-local` (`setup.sh` writes `VLLM_API_KEY=vllm-local` to `~/.openclaw/.env`)
   - context window `131072` tokens, up to `8192` output tokens, request timeout 300 s

   Check it with `curl localhost:8000/v1/models`. If tool calls come back as text or with empty arguments, restart vLLM with the `qwen3_xml` tool parser.
2. **Start ComfyUI** on `127.0.0.1:8188` and check it with `curl localhost:8188/system_stats`. Export your workflows in ComfyUI's API format to `$KIT/workflows/` (`KIT` defaults to `~/kit`). The repo expects:
   - `sdxl_api.json` (default, `DESK_MODEL=sdxl`): prompt text node `6`, output node `9`
   - `zit_api.json` (optional Z-Image, `DESK_MODEL=zit`): prompt text node `27`
   - in each, a `KSampler` node whose `seed` input `comfy_gen.py` sets and whose `negative` input links to a text node (it fills that with `DESK_NEGATIVE`), and a `SaveImage` node
   - for a reference-image run (`comfy_gen.py --from-image`), a `CheckpointLoader` node whose VAE output (index 2) is used; the image is encoded and sampled at denoise `DESK_DENOISE` (default `0.65`)
3. **Install OpenClaw:** `npm i -g openclaw@2026.7.1`.
4. **Add your Discord bot token** to `~/.openclaw/.env` as `DISCORD_BOT_TOKEN=...`. Turn on the bot's Message Content Intent, and invite it to your server with the `bot` and `applications.commands` scopes.
5. **Find your Discord server ID** (in Discord, turn on Developer Mode, right-click the server and choose Copy Server ID).
6. **Run setup** from the repo. The first command asks for the server ID you copied in step 5:
   ```bash
   read -rp "Discord server ID: " GUILD_ID
   GUILD_ID="$GUILD_ID" bash ticket-desk/setup.sh
   ```
   It fills the config template with your home folder, your `KIT` folder and your server ID, does a dry run of the config and then applies it.
7. **Start the gateway:**
   ```bash
   set -a; . ~/.openclaw/.env; set +a; openclaw gateway
   ```
8. **Try it** in your Discord server:
   ```
   make a cozy mushroom lantern, pink cap, warm glow
   make the cap blue
   bigger glow
   go back to v1 but make it blue
   history        status        stats
   ```
   If you attach a reference image and ask for something in that style, the desk files a new ticket.

### Configuration

| Variable | Default | Used by |
|---|---|---|
| `DISCORD_BOT_TOKEN` | none (you add it to `~/.openclaw/.env`) | OpenClaw Discord channel |
| `GUILD_ID` | none (required by `setup.sh`) | Filled into the config template |
| `VLLM_API_KEY` | `vllm-local` (written by `setup.sh`) | vLLM provider |
| `TICKET_DB` | `~/.openclaw/tickets.db` | `ticket.py` |
| `TICKET_MAX_ATTEMPTS` | `3` | Reviewer retries per version |
| `TICKET_MAX_VERSIONS` | `0` (unlimited) | Requester changes per ticket |
| `KIT` | `~/kit` | Where `workflows/*.json` live |
| `COMFY_URL` | `http://127.0.0.1:8188` | `comfy_gen.py` |
| `DESK_OUT` | `~/desk-out` | Where generated images are saved |
| `DESK_MODEL` | `sdxl` (`sdxl` \| `zit`) | Which ComfyUI workflow to use |
| `DESK_NEGATIVE` | Blocks text, watermarks and cropping | Shared negative prompt |
| `DESK_DENOISE` | `0.65` | Denoise for `comfy_gen.py --from-image` |

Most tuning means editing plain-text instructions. The spec format is in `workspaces/desk/AGENTS.md`, the prompt recipe in `workspaces/artist/AGENTS.md` and the quality bar in `workspaces/reviewer/AGENTS.md`.

### Full checklist

The full install, tuning, WhatsApp and troubleshooting checklist is in [`ticket-desk/README.md`](ticket-desk/README.md) (see [Debugging](ticket-desk/README.md#debugging)).

## Known limits

- **Never tested on a live gateway.** The config keys come from the OpenClaw 2026.7.1 documentation, and the config has not been run against a live OpenClaw gateway. If a key is rejected, `openclaw config patch --dry-run` names it.
- **Workflows, prompt templates and the start-up helper are not included.** The ComfyUI workflows (`$KIT/workflows/sdxl_api.json`, optional `zit_api.json`), the prompt template file the artist reads (`~/kit/kb/prompt-templates.md`) and the `restore.sh` helper that started vLLM and ComfyUI came from the system the demo was built on and are not in this repo. You start vLLM and ComfyUI yourself with the settings in Setup, and supply your own workflows. Without the template file, the artist uses the fallback prompt recipe in its `AGENTS.md`.
- **Reference images are described.** By default, the desk describes a reference image and carries its traits into the prompt. Full img2img (LoadImage → VAEEncode → KSampler at denoise ≈ 0.6) is a stretch goal and is not part of the default agent flow.
- **Discord is required to try it as shipped.** WhatsApp is optional and needs a phone you can link.
- **No timing figures.** No run time or image turnaround has been measured, so none is stated.

## Credits

- **Created by the StudioClaw team: Paritosh Singh, Yi Zhu and Qijun Fang.** If this demo is useful to you, please star [the original repo](https://github.com/byronf-sama/dell-boston-hackathon-studio-claw).
- **Partner:** NVIDIA (silicon: NVIDIA GB10 Grace Blackwell superchip).
- **Software the demo runs on**, each installed by you and kept under its own licence (see `THIRD_PARTY_NOTICES.md`): [OpenClaw](https://openclaw.ai), [vLLM](https://github.com/vllm-project/vllm), [ComfyUI](https://github.com/comfyanonymous/ComfyUI), the Qwen3.6-35B NVFP4 model (`nvidia/Qwen3.6-35B-A3B-NVFP4`), and the SDXL and Z-Image image models you load into ComfyUI. No dependency repos are bundled, and setup doesn't clone anyone else's repo.
- **Upstream licence:** the original repo has no licence file. The authors keep their rights; Dell shares this copy with their permission under the terms in `LICENSE`.
- **AI-generated assets:** this repo ships none. The images the demo makes are generated on your own system by the models you install.
- **Changes by Dell Technologies:** Dell rewrote this README and the setup notes, removed a generated config file that held machine-specific values, stopped that file from being committed, and pointed setup at the vLLM and ComfyUI settings above instead of the helper that is not included.
- Dell, Dell Technologies and Dell Pro Precision are trademarks of Dell Inc. or its subsidiaries. NVIDIA, Grace, Blackwell and DGX are trademarks of NVIDIA Corporation. Discord, WhatsApp, OpenClaw, vLLM, ComfyUI, Qwen, SDXL and Z-Image, and other names, are trademarks of their respective owners. Naming them does not mean those owners endorse or sponsor this demo or Dell, and Dell does not endorse them.

## Support

Logan Lawler, Dell Technologies: logan.lawler@dell.com

## License

Not MIT. The original repo has no licence file, so the StudioClaw team keeps all rights to their code. Dell Inc. shares this copy with the authors' permission so that you can download, run and evaluate it as a demo; no other rights are granted, and for any other use, contact the authors. Dell's changes ("Changes made by Dell Technologies: Copyright (c) 2026 Dell Inc.") carry the same terms. Third-party components, models and data keep their own licences. See [`LICENSE`](LICENSE) and [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md).
