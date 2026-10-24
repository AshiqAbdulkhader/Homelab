# Local LLM

A CPU-only llama.cpp server behind an OpenAI-compatible API, a chat UI, and a
private search engine. All private (tailnet only), all declared in
`homelab-gitops` under `apps/llm`, `apps/open-webui` and `apps/searxng`.

| Piece | Details |
|---|---|
| Model | Qwen3-1.7B, Q4_K_M, downloaded once by an init container (pinned revision, SHA-256 checked, resumable) |
| Server | llama.cpp `server-b11804`, 4 generation threads, 5 prompt threads, 8K context, one slot, thinking off |
| UI | Open WebUI (slim image) with web search through SearXNG |
| n8n | Credential "Local LLM (llama.cpp)" (OpenAI type, base URL `http://llm.llm.svc:8080/v1`) |

## Measured on this host (i5-9400, memory-bandwidth bound)

| Model | Generation | Prompt processing |
|---|---|---|
| Qwen3-1.7B Q4_K_M | 17 to 24 tok/s (10 tok/s once the context is about 2K tokens) | about 120 tok/s |
| Qwen3.5-4B Q4_K_M | about 6 tok/s | about 55 tok/s |

Generation stops improving past 4 threads: memory bandwidth, not compute, is
the limit (about 16 to 19 GB/s effective). Repeated prompt prefixes are cached,
so follow-up turns in a chat are much faster than the first.

## Keeping it fast and warm

- The model is pinned in RAM (`--load-mode mmap+mlock`, root with only
  `IPC_LOCK`). Without it the kernel evicted the model's pages and re-read
  them from disk, which looked like a model reload on the next prompt.
- Memory limit 5 GiB (request 3 GiB); llama.cpp's prompt cache is capped at
  512 MiB (the default 8 GiB filled the old 3 GiB limit).
- llama.cpp never unloads the model on idle (`--sleep-idle-seconds` is off).
- Web search passes search snippets, not whole pages. Whole pages put about
  6,000 tokens in the prompt, which is about a minute of prompt processing.
- Open WebUI's background title, tag and follow-up generation is off: each is
  another request on a slow model.

## Gotchas

- This llama.cpp build rejects `--flag=value`; pass flag and value as
  separate arguments. `--mlock` is now `--load-mode mmap+mlock`.
- The first start downloads about 1.1 GB over Wi-Fi (10 to 20 minutes). Flux's
  `apps` Kustomization reports not-Ready until then.
- Backups skip the model volume; it is re-downloaded and verified.
