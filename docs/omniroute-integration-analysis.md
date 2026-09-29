# OmniRoute ↔ Headroom: integration analysis

Analysis date: 2026-09-29. Sources: [`diegosouzapw/OmniRoute`](https://github.com/diegosouzapw/OmniRoute)
@ v3.8.52 (MIT), Headroom @ `main` (v0.39.1).

---

## 1. Bottom line

OmniRoute is a Node/Next.js **AI gateway**: one OpenAI/Anthropic-compatible endpoint
(`:20128`) in front of ~358 providers, with combo routing, quota-aware fallback, and its own
stack of in-process compression engines (RTK, Caveman, CCR, session-dedup, LLMLingua, …).
Headroom is the **context-compression layer**. They overlap on compression only; routing,
provider fan-out and quota management are OmniRoute's and have no Headroom counterpart.

The two already reference each other, but **they are not actually wired together**:

| What OmniRoute ships | What it really does |
|---|---|
| `src/lib/headroom/{detect,process}.ts`, `/api/headroom/{start,stop,status}`, Settings → "Headroom proxy" card, `HEADROOM_URL` | Finds the `headroom` CLI, spawns `headroom proxy --port 8787` detached, probes `GET /health`. |
| A compression engine with `id: "headroom"` (`open-sse/services/compression/engines/headroom/`) | OmniRoute's **own TypeScript reimplementation** of SmartCrusher-style tabular JSON compaction (GCF/TOON codec). It never calls Headroom. |

`headroomUrl` / `HEADROOM_URL` is read only by the start and status routes (and edited by the
Settings card); the stop route just kills the PID it recorded. **No request path ever sends a byte of chat traffic to the Headroom proxy.** A user
who clicks "Start" in OmniRoute's dashboard gets a Headroom process that sits idle on
`:8787`, and savings reported on OmniRoute's "Headroom" dashboard page are the in-house
engine's, not Headroom's.

The good news: Headroom already exposes exactly the seam a gateway needs —
`POST /v1/compress` with the **gateway turn contract** (`headroom/proxy/gateway_extension.py`,
documented in `docs/content/docs/proxy.mdx` → "Gateway mode"). It is written for "Kong,
Envoy, LiteLLM, Cloudflare …"; OmniRoute is the same shape.

Recommended order:

1. **Now, zero code:** document Topology A (Headroom in front of OmniRoute) — works today.
2. **Upstream PR to OmniRoute:** a `headroom-remote` async engine that calls Headroom's
   `/v1/compress` in gateway mode (Topology B). This is what makes OmniRoute's existing
   start/stop card do something.
3. **Optional, Headroom side:** a `docs/content/docs/omniroute.mdx` page and, later, a
   `headroom wrap`-style helper. No core changes needed for either.

---

## 2. OmniRoute at a glance (what matters for integration)

- **Runtime:** Node 22.22+ or 24–26 (`package.json` `engines`), Next.js app + `open-sse/` request engine, SQLite in `DATA_DIR`.
  Default port `20128` (`.env.example`).
- **Client-facing API:** `/v1/chat/completions`, `/v1/messages` (Anthropic), `/v1/responses`,
  embeddings, images, … (`src/app/api/v1/*`). Agents point `OPENAI_BASE_URL` /
  `ANTHROPIC_BASE_URL` at it.
- **Compression pipeline:** `open-sse/services/compression/`. Engines implement
  `CompressionEngine` (`engines/types.ts`) with sync `apply()` and an optional
  **`applyAsync()`** that the stacked pipeline awaits when present (`engines/types.ts:84-95`).
  Engines declare `stackable` / `stackPriority`; the default stack is `RTK → Caveman`.
  LLMLingua is the precedent for an async, **fail-open** engine (any error → original text).
- **Lifecycle guard:** the Headroom start/stop routes are `LOCAL_ONLY` and refuse
  non-loopback URLs ("External Headroom proxies must be started outside OmniRoute"); an
  external URL is only probed via `/health`.

Name collisions to be aware of (all unrelated to this product, but they will confuse users
and search):

| OmniRoute name | Meaning |
|---|---|
| `headroom` compression engine | In-house JSON-array tabular compaction ("inspired by" Headroom, per its README credits) |
| `headroom` combo routing strategy | "Pick the target with the most remaining **quota** headroom" |
| `ccr` engine / `omniroute_ccr_retrieve` tool / `[CCR retrieve hash=…]` markers | OmniRoute's own content-addressed retrieve, **not** wire-compatible with Headroom CCR (`<<ccr:…>>` + `headroom_retrieve`) |
| `rtk` engine | OmniRoute's TS port of RTK-style shell-output filters (Headroom's `headroom/rtk/installer.py` installs the real `rtk` binary) |

---

## 3. Integration topologies

### Topology A — Headroom in front of OmniRoute (works today)

```
agent ──► headroom proxy :8787 ──► OmniRoute :20128 ──► 358 providers
          (compress)               (route / fallback / quota)
```

```bash
# Headroom appends /v1/... itself, so pass the bare origin.
headroom proxy --port 8787 \
  --openai-api-url    http://127.0.0.1:20128 \
  --anthropic-api-url http://127.0.0.1:20128 \
  --provider-name OmniRoute

export OPENAI_BASE_URL=http://127.0.0.1:8787/v1
export ANTHROPIC_BASE_URL=http://127.0.0.1:8787
```

- **Pros:** no code on either side; full Headroom feature set (cache mode, CCR with
  `headroom_retrieve`, tool-schema compaction, memory, dashboards); OmniRoute keeps routing.
- **Cons / caveats:**
  - **Double compression.** Leave OmniRoute's stacked compression **off** (or limited to
    engines Headroom does not cover) on this path. Two independent rewriters on the same
    prefix is the fastest way to bust provider prompt cache — Headroom's `cache` mode
    freezes the prefix, but OmniRoute's engines do not know that.
  - OmniRoute's combo/virtual model IDs (e.g. `auto/smart`) are opaque to Headroom's
    tokenizer registry and pricing; savings are still measured, but `$` figures and
    context-limit-driven aggressiveness fall back to defaults. Pass a real model ID where
    possible.
  - OmniRoute's own `omniroute_ccr_retrieve` tool injection is irrelevant here and should
    stay off, to avoid two retrieve protocols in one context.
- **Status:** not yet exercised end-to-end in this analysis. Before publishing a user-facing
  page, verify chat, messages and responses streaming through both hops.

### Topology B — OmniRoute calls Headroom as a compression sidecar (recommended upstream PR)

```
agent ──► OmniRoute :20128 ──(POST /v1/compress, gateway mode)──► headroom :8787
               │ ◄──────────── compressed provider body ──────────────┘
               └──► provider ──► response ──(POST /v1/compress/response, usage relay)──► headroom
```

A new OmniRoute engine, e.g. `engines/headroomRemote/`, modelled on the LLMLingua engine:

- `applyAsync(body)` → `POST {headroomUrl}/v1/compress` with `model`, `messages` and a
  `gateway` block (so `system` and `tools` are compressed too — outside gateway mode they are
  silently ignored, see `proxy.mdx` "system and tools are ignored outside gateway mode").
- `apply(body)` returns the body unchanged (sync fallback must be safe).
- **Fail open** on timeout, non-2xx, `404` (remote caller without
  `HEADROOM_COMPRESS_ALLOW_REMOTE=1`), or `HEADROOM_GATEWAY_CONTRACT=0` legacy-shape answers.
- Leave `config.mode` **unset** (marker-free) unless OmniRoute also injects
  `headroom_retrieve` and can reach `/v1/retrieve`; otherwise CCR markers are dangling
  pointers for the model.
- Send `frozen_message_count` (or use `session_id` session mode) so already-cached prefix
  turns come back byte-identical.
- Relay provider usage after the response so Headroom's savings ledger is accurate.
- `stackable: true`, and document that it **replaces** OmniRoute's `rtk` / `caveman` /
  local `headroom` engines rather than stacking with them.
- Reuse the existing `settings.headroomUrl` and the Settings card; for a non-loopback URL
  (Docker sidecar) require `HEADROOM_PROXY_TOKEN` and send `X-Headroom-Proxy-Token`.

This is the only topology where OmniRoute's current "Headroom proxy" UI does real work, and
it keeps OmniRoute as the single endpoint agents talk to. It is a change to OmniRoute, so it
belongs in an issue/PR against `diegosouzapw/OmniRoute`, not in this repo.

### Topology C — Headroom behind OmniRoute as a "provider" (not recommended)

Registering `http://127.0.0.1:8787` as an OpenAI-compatible provider inside OmniRoute would
compress only the traffic routed to that one pseudo-provider, and Headroom would then need
its own upstream credentials — it defeats OmniRoute's routing. Listed only to rule it out.

---

## 4. What Headroom could do on its side

None of this needs core changes.

| Item | Effort | Notes |
|---|---|---|
| `docs/content/docs/omniroute.mdx` for Topology A | S | Mirror `litellm.mdx` / `opencode-deepseek.mdx`. Include the double-compression warning. Verify first. |
| Gateway-contract reference client in TS | S–M | The TypeScript SDK (`sdk/typescript/src/client.ts`) already wraps `/v1/compress`; a small `gateway` helper would make Topology B a thin adapter for any Node gateway, not just OmniRoute. |
| Test fixture: OmniRoute-shaped request bodies | S | Its combo model IDs and `/v1/responses` bodies through `/v1/compress` gateway mode, to catch tokenizer/model-resolution gaps early. |
| `headroom wrap omniroute` | M | Would write `--openai-api-url/--anthropic-api-url` and point OmniRoute's clients at Headroom. Only worth it once Topology A is verified and there is demand. |

---

## 5. Risks

- **Cache busting from stacked rewriters** (Topology A with OmniRoute compression on).
  Highest-impact risk; mitigated by documentation and, in Topology B, by having one
  compressor in the path.
- **Protocol confusion:** two CCR dialects and two "RTK"s. Only one retrieve tool should be
  exposed to the model per request.
- **Attribution / naming:** OmniRoute's dashboard labels its in-house engine "Headroom
  SmartCrusher". Users will attribute its behaviour and savings numbers to this project.
  Worth raising politely with the OmniRoute maintainers alongside the Topology B proposal
  (e.g. renaming the engine to `tabular` / "SmartCrusher-style").
- **License:** OmniRoute is MIT, Headroom is Apache-2.0. Calling each other over HTTP has no
  licensing implications; copying code in either direction would need the usual notices.

---

## 6. Suggested next steps

1. Stand up both locally, verify Topology A for `/v1/chat/completions`, `/v1/messages` and
   `/v1/responses` (streaming and non-streaming), then publish the docs page.
2. Open an issue on OmniRoute describing the gap in §1 and proposing Topology B, linking
   Headroom's gateway-mode docs.
3. If they accept, contribute the `headroomRemote` engine there (fail-open, async, gateway
   mode, usage relay).
