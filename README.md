# 6gb-agent

A private, uncensored, local coding agent for a laptop with a 6 GB GPU. It runs a 35B Mixture-of-Experts model with llama.cpp and drives it from OpenCode.

Reference machine: RTX 3060 Laptop (6 GB), 30 GB RAM, 16 threads, CachyOS, llama.cpp `5676dfb`, OpenCode 2.1.4.

## Why this works on 6 GB

Qwen3.6-35B-A3B has 35B parameters, but only about 3B are active per token. The attention layers and KV cache stay on the GPU (about 3 GB). The expert weights (about 15 GB) stay in system RAM and run on the CPU (`--n-cpu-moe 40`). This model is also a hybrid with mostly linear-attention layers, so a 64K context costs little VRAM.

## Measured on the reference machine

| | Old: qwen3.5 4B in Ollama | This setup: 35B-A3B IQ4_XS |
|---|---|---|
| Generation | 16 tok/s (27% on CPU) | 20–27 tok/s |
| Prompt processing | 45 tok/s | 460–565 tok/s |
| VRAM | spilled | 4.65 GB of 6 GB at 64K context |
| Tool calls | unreliable | valid in thinking and non-thinking modes |
| OpenCode cold first turn | – | ~15 s (6K-token system prompt) |

Tuning results: `--n-cpu-moe` below 40 runs out of VRAM once the desktop holds 1.7 GB. `-ub 2048` gives 2.7× faster prompt processing than 512, and `-ub 4096` does not fit. MTP speculative decoding was skipped: its draft buffers need VRAM this card does not have.

## Layout

| Path | Installed as | What it does |
|---|---|---|
| `bin/llm-serve` | `~/.local/bin/llm-serve` | llama-server with the tuned flags. Every value can be overridden with env vars. |
| `systemd/llm.service` | `~/.config/systemd/user/llm.service` | Started on demand by `oc`. Weights pinned in RAM, restarts after crashes, killed first under memory pressure. |
| `bin/llm-gameguard` + `systemd/llm-gameguard.service` | `~/.local/bin`, enabled at login | Stops the model within 5 s of a Steam game starting. |
| `bin/oc` | `~/.local/bin/oc` | Starts the model if needed (refuses while a game runs), then runs OpenCode without `~/.claude` skills, which cuts the system prompt from 14K to 6K tokens. |
| `opencode/opencode.json` | merged into `~/.config/opencode/opencode.json` | The `local` provider, with `qwen-think` and `qwen-fast`. |
| `opencode/AGENTS.md` | `~/.config/opencode/AGENTS.md` | Working rules tuned for a slow local model. |
| `searxng/` | `docker compose up -d` | Private web search on `127.0.0.1:8888`, for the `searxng` MCP. Ask "use the searxng tool to find …" in the TUI. |
| `herdr/local-big.md` | `~/.config/herdr-core/templates/` | herdr helper template. |

## Daily use

```sh
cd ~/some/repo && oc                         # TUI; default model is local/qwen-think
oc run -m local/qwen-fast "explain src/x.ts"  # one-shot, no thinking
```

In the TUI, switch models with `/models`. Use `qwen-fast` for reading, explaining and simple edits. Use `qwen-think` for bugs and multi-file changes.

### Writing prompts for a local model

A 35B model needs tighter tasks than Claude. Good prompts include:

1. **Scope:** name the files. Example: "In `src/auth/session.ts` and its test…"
2. **Verification:** give the exact command. Example: "Run `pnpm test session` until it passes."
3. **Done condition:** say what finished looks like. Example: "Then show `git diff --stat`. Do not commit."

Split big jobs into steps, one prompt per step, in the same session. For architecture questions or bugs it can't crack after two tries, switch to Claude.

## Failure modes and recovery

| Situation | What happens | What to do |
|---|---|---|
| Idle for 5 min | The server sleeps: weights and KV cache are freed from RAM and VRAM. | Nothing. The next request reloads it (~30 s for the first reply). |
| Server crashes | systemd restarts it after 5 s. After 5 crashes in 2 min it gives up. | `journalctl --user -u llm -n 50`, then `systemctl --user reset-failed llm && systemctl --user start llm`. |
| RAM fills up (browser, game) | Weights are mmapped, so the kernel evicts them first and generation slows down. If memory is still short, the OOM killer picks the model first (`OOMScoreAdjust=500`) and systemd restarts it. | Close apps, or wait. OpenCode sessions survive. Retry with `oc -c`. |
| You start a Steam game | `llm-gameguard` stops the model within 5 s, which frees ~3 GB of VRAM and ~18 GB of RAM. `oc` refuses to restart it while the game runs. | Nothing. Run `oc` after the game. For non-Steam games, set `GAME_PATTERN` in the gameguard unit. |
| Model thinks forever | Thinking is capped at 4096 tokens, then it is told to act. | Use `qwen-fast` for simple tasks. |
| Agent loops on a failing command | AGENTS.md tells it to stop after two identical failures. | Interrupt (Esc), restate the task more narrowly. |

Tuning knobs (with `systemctl --user edit llm`):

```ini
[Service]
Environment=CTX=32768      # less VRAM, more headroom for other apps
Environment=IDLE=120       # free memory even sooner
Environment=MLOCK=0        # don't pin weights (no root change needed, slower under RAM pressure)
Environment=NCMOE=38       # only on GPUs with more free VRAM; faster
```

## Pinning the weights in RAM

Without pinning, a busy desktop makes the kernel evict model weights, and prompt reading drops from ~500 to 7–80 tok/s. `llm-serve` uses `--load-mode mmap+mlock` to keep them resident. That needs a one-time root change to the memlock limit:

```sh
sudo mkdir -p /etc/systemd/system/user@.service.d
printf '[Service]\nLimitMEMLOCK=infinity\n' | sudo tee /etc/systemd/system/user@.service.d/memlock.conf
sudo prlimit --pid "$(pgrep -u "$USER" -f 'systemd --user' | head -1)" --memlock=unlimited:unlimited  # applies now; the file covers later logins
```

The trade-off: while the model is awake, other apps get ~18 GB less RAM and are pushed into (compressed) swap. Idle sleep and the game guard give that RAM back.

## Install on another machine

1. Build llama.cpp with CUDA: `cmake -B build -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=<sm> && cmake --build build -j --target llama-server llama-bench`.
2. Download the model: `hf download mradermacher/Huihui-Qwen3.6-35B-A3B-abliterated-i1-GGUF Huihui-Qwen3.6-35B-A3B-abliterated.i1-IQ4_XS.gguf --local-dir ~/models`. It needs 18.7 GB of disk and at least 24 GB of RAM. With 16 GB of RAM, use `i1-IQ3_XXS` (13.6 GB).
3. Link `bin/*`, `systemd/llm.service`, `opencode/AGENTS.md`, and merge `opencode/opencode.json`.
4. Run `systemctl --user enable --now llm`, then `curl localhost:8080/health`.
5. Tune with `llama-bench -ncmoe <N> -ub 512,2048`: use the lowest `N` that doesn't run out of memory.

## Security

The server binds to `127.0.0.1` only and has no API key, so do not expose port 8080. The model is uncensored and will run any command you approve. Review permission prompts, and avoid `--auto` outside throwaway directories.
