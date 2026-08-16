# ASCIIty

<p align="center">
    <img src="./city.gif"
         alt="Qwen3.8-27B generated 3D ASCII city"
         width="960">
</p>

*Read [CITY.md](CITY.md)*

## Qwen 3.8 27B Capabilities Test

## Run summary

| Metric | Value |
|---|---:|
| Model | Qwen3.8-27B Q6_K, local llama.cpp |
| Harness | pi.dev |
| Thinking level | High |
| Assistant generations | 82: 81 produced output, 1 aborted |
| Output tokens | 231,999 |
| Uncached input tokens | 333,623 |
| Cache-read tokens | 11,132,403 |
| Cumulative tokens processed across calls | 11,698,025 |
| Average prompt context | 139,830 tokens/call |
| Peak prompt context | 249,351 / 262,144 tokens (95.1%) |
| Largest input + output turn | 249,521 tokens |
| Weighted generation speed | 16.04 tokens/s |
| Generation speed excluding one stalled call | 17.79 tokens/s |
| Median generation speed | 16.84 tokens/s |
| Peak generation speed | 29.94 tokens/s |
| Reported cost | $0, local |
| Session wall time | About 4h 19m |
| Recorded generation time | About 4h 1m |

The token counters are cumulative across requests. Uncached input is not necessarily unique input, and cache-read tokens include context processed repeatedly across turns. 
Output tokens include thinking, visible responses, and serialized tool calls.

The provider reported zero reasoning tokens despite emitting explicit thinking blocks. The session contains 489,720 characters of stored thinking, but an exact reasoning-only token count is unavailable. 
A character-proportional estimate would place thinking near 169,000 tokens; this is approximate because prose and code tokenize differently.

## Test environment

### Hardware

| Component | Configuration |
|---|---|
| System | ASRock Rack ROMED8-2T, Ubuntu 24.04.4 LTS, Linux 6.8.0-137-generic |
| CPU | AMD EPYC 7402P: 24 cores / 48 threads, up to 2.8 GHz, 128 MiB L3 |
| System memory | 256 GiB class |
| GPU | AMD Radeon AI PRO R9700, `gfx1201` |
| VRAM | 32,624 MB GDDR6, 256-bit |
| GPU interface | PCIe 4.0 x16 |
| GPU software | ROCm 7.2.4, AMD SMI 26.2.2, `amdgpu` 6.16.13 |

### llama.cpp build configuration

The HIP-enabled `llama-server` binary was compiled for the Radeon AI PRO R9700's `gfx1201` target.

```bash
HIPCXX="$(hipconfig -l)/clang" HIP_PATH="$(hipconfig -R)" \
    cmake -S . -B build \
    -DGGML_HIP=ON \
    -DGPU_TARGETS=gfx1201 \
    -DCMAKE_BUILD_TYPE=Release \
    && cmake --build build --config Release -- -j$(nproc)
```

### Runtime and inference configuration

| Setting | Value |
|---|---|
| Server | Custom `atomic-llama-cpp-turboquant` llama.cpp build |
| llama.cpp version | [`b10269-1.5.1`](https://github.com/AtomicBot-ai/atomic-llama-cpp-turboquant/tree/b10269-1.5.1), build 10697, commit `af8a7888d` |
| Model file | [Qwen3.8-27B-Q6_K.gguf](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF/blob/main/Qwen3.8-27B-Q6_K.gguf) |
| Context size | 262,144 tokens |
| Maximum generation | 32,768 tokens |
| Parallel slots | 1 |
| GPU offload | 999 model layers and 999 draft layers (full available offload) |
| CPU threads | 24 generation / 12 batch |
| Attention | Flash attention enabled |
| KV cache | K: `q8_0`; V: `turbo3` |
| Host cache | 32,768 MiB |
| Context checkpoints | 64, minimum step 256 |
| Idle slot caching | Disabled |
| Speculative decoding | Draft MTP, 1–3 draft tokens; draft K: `q8_0`, V: `turbo3` |
| Multimodal projector | Disabled |
| Reasoning mode | Auto, preserved, DeepSeek format |
| Template engine | Jinja |

### Sampling configuration
*Note: Qwen3.8-27B Recommended Defaults*

| Parameter | Value |
|---|---:|
| Temperature | 1.0 |
| Top-p | 0.95 |
| Top-k | 20 |
| Min-p | 0.0 |
| Presence penalty | 0.0 |
| Repeat penalty | 1.0 |

#### Server flags

```text
llama.cpp/build/bin/llama-server \
    --model Qwen3.8-27B-Q6_K.gguf \
    --host 127.0.0.1 \
    --port 8000 \
    --flash-attn on \
    --ctx-size 262144 \
    --jinja \
    --chat-template-file chat_template.jinja \
    --n-gpu-layers 999 \
    --n-gpu-layers-draft 999 \
    --threads 24 \
    --threads-batch 12 \
    --cache-type-k q8_0 \
    --cache-type-v turbo3 \
    --no-cache-idle-slots \
    --ctx-checkpoints 64 \
    --checkpoint-min-step 256 \
    --cache-ram 32768 \
    --reasoning auto \
    --reasoning-preserve \
    --reasoning-format deepseek \
    --parallel 1 \
    --temp 1.0 \
    --top-p 0.95 \
    --top-k 20 \
    --min-p 0.0 \
    --presence-penalty 0.0 \
    --repeat-penalty 1.0 \
    --no-mmproj \
    --spec-type draft-mtp \
    --spec-draft-n-max 3 \
    --spec-draft-n-min 1 \
    --spec-draft-type-k q8_0 \
    --spec-draft-type-v turbo3
```

## Refined chat template

The run used the heavily modified custom `qwen3.8-froggeric-v22` template preserved verbatim in [`chat_template.jinja`](chat_template.jinja). This is not the stock Qwen chat template; its prompt construction, reasoning handling, and tool-use behavior have been substantially reworked.
Its SHA-256 digest is `702d03993327b646467ba9e9794e0bd5e3457c63141d6ab53f61795c05e6cc31`.

This refined template was part of the tested configuration and materially improved the model's agentic performance. Its relevant behaviors include:

- Native Jinja rendering for system, developer, user, assistant, and tool-result messages.
- XML tool calls by default, with optional JSON tool-call formatting.
- Explicit thinking controls through `<|think_on|>` and `<|think_off|>`.
- Reasoning-effort mapping, with both `high` and `xhigh` rendered as “Think extensively and deeply.”
- Preservation of prior reasoning across tool-use turns.
- Correct grouping of consecutive tool responses into a single user turn.
- Optional tool-argument and tool-response truncation controls.
- Built-in focus guidance: “Stay focused. Stop once the answer is established. Answer in detail and state uncertainty clearly.”

Because the chat template materially affects instruction following, reasoning continuity, and tool-call behavior, it should be treated as part of the model configuration rather than as incidental harness formatting.

## Outcome

| Result | Value |
|---|---|
| Completion | Successful |
| Primary artifact | Self-contained `city.html` |
| External dependencies | None |
| Implemented systems | ASCII raycasting, first-person movement, collision, procedural city generation, cars, NPCs, and a day/night cycle |
| Validation | Rendering checks, movement and entity checks, pitch and occlusion checks, performance measurements, and an extended soak test |

## Prompt

```text
Create a small interactive engine that powers a walkable 3D city rendered entirely in ASCII characters
inside a web browser.

The final result should run as a self-contained HTML file, using vanilla JavaScript and CSS if needed,
with zero external dependencies, libraries, frameworks, assets, or network requests.

The user should explore the city from a first-person perspective using WASD movement, with an
appropriate method for looking around.

The scene should update in real time as the player moves through streets, buildings, and other urban
structures.

Everything visible in the world must be represented using text characters only. Do not use images,
canvas graphics, WebGL libraries, external fonts, or graphical assets.

Include a small engine responsible for rendering, movement, world simulation, and entity updates.

Include cars that spawn and drive around the city, plus NPCs that spawn and wander around
independently.

Add a day/night cycle that visibly changes the appearance of the city. It should run much faster than
real time so the full cycle can be observed quickly, and it should be possible to toggle or pause it.

Make the city feel convincingly three-dimensional, alive, navigable, and visually understandable while
remaining entirely ASCII-based.

Choose the rendering technique, engine architecture, controls beyond WASD, procedural generation
approach, traffic behavior, NPC behavior, collision handling, day/night representation, and other
implementation details yourself.
```
