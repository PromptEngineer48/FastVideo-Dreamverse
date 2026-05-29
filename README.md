# Running Dreamverse (FastVideo) on a RunPod B200

A practical, battle-tested guide to deploying [FastVideo's](https://github.com/hao-ai-lab/FastVideo) **Dreamverse** realtime video generation platform on a RunPod **B200** GPU pod over SSH.

This covers the exact steps — and the gotchas — to get from a fresh pod to generating video in the browser.

> **Heads up on the first boot:** Dreamverse compiles its inference paths with `torch.compile` on first startup. This cold warmup takes roughly **20–25 minutes** on a B200. It only happens once per cold pod. Plan around it.

---

## 1. Pod configuration (when deploying)

| Setting | Value | Why |
|---|---|---|
| Container image | `runpod/pytorch:...` | Standard PyTorch template |
| Container disk | 200 GB | Room for the build |
| **Volume disk** | **50 GB**, mounted at `/workspace` | Persists the model cache **and** the torch.compile cache, so a pod restart skips most of the warmup |
| **Expose TCP port** | **5299** | The frontend (Next.js) is served here |
| SSH port | 22 | For access |
| HTTP port (optional) | 8888 | If you want Jupyter |

**Use TCP for 5299, not HTTP.** The frontend is a Next.js dev server, and the RunPod HTTP proxy domain (`*.proxy.runpod.net`) tends to get rejected by the dev server's host check. The raw TCP `IP:port` mapping works reliably.

---

## 2. Get the API keys (free)

Dreamverse uses two providers for prompt rewrite/enhancement. The backend **will not start** without both.

- **Cerebras** — https://cloud.cerebras.ai
- **Groq** — https://console.groq.com

Grab a key from each.

---

## 3. One-time install (SSH into the pod)

```bash
# Install Node if `npm -v` doesn't already work
curl -fsSL https://deb.nodesource.com/setup_20.x | bash - && apt install -y nodejs

# Clone the repo and set up the Python environment
git clone https://github.com/hao-ai-lab/FastVideo.git
cd FastVideo
pip install --upgrade pip && pip install uv
uv venv .venv --python 3.12
source .venv/bin/activate
uv pip install -e ".[dreamverse]"

# Required: the fast HuggingFace downloader, or the backend errors on model download
uv pip install hf_transfer

# Install frontend dependencies (run once)
cd apps/dreamverse/web && npm ci && cd ../../..
```

> **Note on `uv pip` vs `pip`:** Because the environment is created with `uv venv`, it has no standalone `pip` inside it. Always use `uv pip install ...` so packages land in the active venv. Don't mix the two.

---

## 4. Start the backend

Run it inside a `tmux` session so it survives SSH disconnects.

```bash
tmux new -s dv
source .venv/bin/activate
export CEREBRAS_API_KEY=your-cerebras-key
export GROQ_API_KEY=your-groq-key
dreamverse-server --host 0.0.0.0 --port 8009
```

Detach (leave it running): `Ctrl-b` then `d`
Reattach later: `tmux attach -t dv`  ← note `-t`, not `-s`

> A direct `dreamverse-server` call does **not** auto-source `~/.env`, so export the keys in the shell (as above) or `source ~/.env` first.

---

## 5. Start the frontend

In a second `tmux` session:

```bash
tmux new -s dvweb
cd FastVideo/apps/dreamverse/web
BACKEND_HOST=localhost BACKEND_PORT=8009 npm run dev   # serves on port 5299
```

Detach: `Ctrl-b` then `d`

The frontend will load even while the backend is still warming up — it just can't generate until the backend reports ready.

---

## 6. Wait for warmup

The backend goes through model download → `torch.compile` warmup before it's ready. `/healthz` responds immediately, but `/readyz` stays `503` until warmup finishes.

In a third shell, poll until ready (auto-notifies with a terminal bell):

```bash
until curl -sf http://localhost:8009/readyz >/dev/null 2>&1; do
  echo "$(date +%T) still warming up..."
  sleep 15
done
echo -e '\a'; echo "===== BACKEND READY — GO ====="
```

**Expected progression:**

1. `curl: (7) Couldn't connect` — backend still compiling, hasn't opened the port yet
2. `503` — port is open, warmup finishing
3. ready (`200`) — done

---

## 7. Open it in the browser

1. RunPod console → your pod → **Connect**
2. Find the **TCP Port Mappings** for `5299` (e.g. `213.173.x.x : 40xxx → 5299`)
3. Open `http://<public-ip>:<external-port>` in your browser — **http**, not https

Type a prompt, hit Generate, and you should get a video.

---

## Troubleshooting

**Backend exits with `hf_transfer` error**
`RuntimeError: ... 'hf_transfer' package is not available`
→ `uv pip install hf_transfer`, then restart the backend. (Or set `HF_HUB_ENABLE_HF_TRANSFER=0` for a slower download with no extra package.)

**`Prompt-provider environment variable errors`**
→ Export both `CEREBRAS_API_KEY` and `GROQ_API_KEY` in the same shell that runs the backend.

**`tmux attach -s` fails with `unknown flag -s`**
→ Use `tmux attach -t dv`. The `-s` flag is only for `tmux new`.

**Autotune logs show `OutOfMemoryError: out of resource ... Ignoring this choice`**
→ Normal. Autotune tries many kernel configs; ones that exceed shared memory are skipped. Not a crash.

**Lots of `UserWarning` / `DeprecationWarning` during warmup**
(`struct.scalar.ptr`, `sourceTensor.detach().clone()`, cutlass warnings)
→ Harmless library noise. Ignore.

**`/readyz` stays at `503` for a long time**
→ Cold warmup is genuinely ~20–25 min on a B200. Confirm the `dv` tmux window is still printing new lines with advancing timestamps — that means it's progressing, not stuck.

**Frontend can't be reached from your laptop**
→ Don't use the `localhost:5299` or internal `172.x` URLs printed by the dev server — those are inside the pod. Use the external TCP mapping from the Connect panel.

**Next.js rejects the host / "Blocked cross-origin request"**
→ Prefer the TCP mapping (avoids it). If it still happens, restart with `npm run dev -- -H 0.0.0.0`.

---

## Notes

- Dreamverse owns its backend under `apps/dreamverse/dreamverse/`. Launch it with `dreamverse-server`, **not** `fastvideo serve`.
- A mock backend is available for UI-only work without a GPU: `dreamverse-mock-server --port 8009`.
- With the `/workspace` volume in place, the model cache and compile cache persist across pod restarts — so a warm restart is dramatically faster than the first cold boot.

---

*This guide reflects the official [FastVideo Dreamverse README](https://github.com/hao-ai-lab/FastVideo) plus fixes for issues encountered during a real RunPod B200 deployment.*
