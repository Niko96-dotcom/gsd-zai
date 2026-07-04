# Implementation Plan — `gsd-zai`: Z.ai / GLM Coding Plan runtime for GSD Core

**Fork:** `Niko96-dotcom/gsd-zai` ← `open-gsd/gsd-core` (synced to `8f2ebbe9` on `next`, 2026-07-04)
**Goal:** Make GSD Core a first-class agent-discipline layer for **Z.ai's GLM Coding Plan**, so the
five-step Discuss → Plan → Execute → Verify → Ship loop runs on top of GLM models the same way it
already runs on Claude Code, Codex, Kimi, Qwen, etc.

## Status

| Phase | Status | Notes |
|---|---|---|
| 0 — Foundation branch | ✅ Done | `feat/zai-runtime` off `next`; baseline green |
| 1 — Runtime descriptor | ✅ Done | `capabilities/zai/capability.json`; registry regenerates clean |
| 2 — Name-policy + aliases | ✅ Done | `zai`/`glm`/`zhipu`/`z-ai`/`glm-code` → `zai`; label `Z.ai`; `.zai` dir |
| 3 — Model catalog | ✅ Partial | `zai` added to `runtimeTierDefaults` (Group B, all-null tiers) |
| 4 — Installer surface | ✅ Done | `--zai`, `ZAI_CONFIG_DIR`, menu option 16, skill-writing + branding wired |
| 5 — Live verification | ⏳ Deferred | Requires a real GLM Coding Plan subscription + `ZHIPU_API_KEY`; pin concrete GLM model ids here |
| 6 — Tests, docs, release | ✅ Done | All golden masters updated; full suite at parity with clean upstream; changeset added |

**Phase 5 is the only outstanding item** — it needs live access to a GLM Coding Plan to confirm the
exact GLM model ids per tier (e.g. `glm-4.7`, `glm-5`, `glm-5.2`) and verify hook events fire. Until
then `zai` ships as a Group B runtime (host-configured model wins), which is correct and safe.

**Verification:** the full test suite shows **zero failures introduced** by this branch (diffed
against clean `next`: identical 16 pre-existing failures, all unrelated graphify/security/render-hooks).
ESLint clean on all changed files.

---



## 1. Background & key finding

GSD Core (`@opengsd/gsd-core`, v1.7.0-rc.2) is a runtime-agnostic meta-prompting / spec-driven
development framework. It supports 15 host runtimes via a **descriptor-driven capability system**
(ADR-857): each runtime is declared in `capabilities/<id>/capability.json`, and a central registry
generator (`scripts/gen-capability-registry.cjs`) validates and wires them in.

**The integration gap:** Z.ai (GitHub org `zai-org`, 51 public repos) publishes the GLM model
family (GLM-4.5/4.6/4.7/5/5.2) and the **GLM Coding Plan**, but ships **no first-party "Z.ai CLI"**.
GLM Coding Plan is consumed *through* existing CLIs/agents via OpenAI-/Anthropic-compatible protocols:

| Surface | Mechanism |
|---|---|
| Claude Code, OpenCode, Kilo, Droid | `ANTHROPIC_BASE_URL` → Z.ai endpoint + `ANTHROPIC_AUTH_TOKEN`/`ZHIPU_API_KEY` |
| Cline, Roo, Kilo Code (VS Code) | OpenAI-compatible provider config |
| Cursor, Windsurf | provider/model selection |

So "supporting z.ai" in GSD means **two complementary things**, and this plan does both:

1. **Register `zai` as a first-class runtime capability** (for users who run the GLM Coding Plan as
   a standalone provider and want GSD's loop, skills, and project-instruction file conventions
   anchored to a Z.ai identity — mirroring how `kimi` and `qwen` are registered).
2. **Register the GLM model family + Z.ai provider preset** in the model catalog, so any host
   runtime configured against the Z.ai endpoint gets the right model ids per tier (opus/sonnet/haiku).

This mirrors exactly how `qwen` (Alibaba) is integrated today: a runtime capability descriptor +
a `providerPresets.qwen` entry + tier→model mappings.

---

## 2. Design principles (from `docs/how-to/develop-a-capability.md`)

- **Descriptor-driven, not code-sprawl.** Add a capability manifest; let the registry generator
  validate it. Do **not** fork the loop, skills, or core infra.
- **Tier-2 support to start.** Match `kimi`/`qwen` — no hook surface claim until we verify Z.ai's
  host-integration behaviour empirically.
- **Reuse converters, don't invent them.** GLM Coding Plan's primary host is Claude Code (via the
  Z.ai base URL), so the Claude→skill converters are the correct ones.
- **Keep upstream mergeable.** All changes are additive files or append-only registry entries, so
  future `open-gsd/gsd-core` syncs stay conflict-free.

---

## 3. Touch-point inventory (verified by reading source)

| # | File | What's there | What we add |
|---|---|---|---|
| 1 | `capabilities/zai/capability.json` | — (new) | Runtime descriptor, modelled on `qwen` |
| 2 | `src/runtime-name-policy.cts` | `FALLBACK_ALIASES` map (line ~17) | `zai` aliases |
| 3 | `gsd-core/bin/shared/runtime-aliases.manifest.json` | alias manifest | `zai` entry |
| 4 | `gsd-core/bin/shared/model-catalog.json` | `providerPresets`, `runtimeTierDefaults` | `zai` preset + tier map (GLM models) |
| 5 | `bin/install.js` | `selectRuntimesFromArgs()` (line 437), help text (line 639), env var note (line 1412) | `--zai` flag, `ZAI_CONFIG_DIR`, menu entry |
| 6 | `scripts/gen-capability-registry.cjs` | `VALID_CONVERTER_NAMES`, consistency gates | (validate only — likely no edits) |
| 7 | `tests/` | per-runtime parity tests | `tests/zai-*.test.cjs` parity tests |
| 8 | `README.md` (+ localized READMEs) | supported-runtimes list | add Z.ai / GLM Coding Plan |

---

## 4. Phased implementation

### Phase 0 — Foundation branch (no behaviour change)
- [ ] Create branch `feat/zai-runtime` off `next`.
- [ ] Confirm `npm ci && npm test` is green on a clean checkout (baseline).
- [ ] Capture baseline: `node scripts/gen-capability-registry.cjs --check` passes.

**Exit criteria:** clean tree, green baseline tests, branch pushed.

### Phase 1 — Runtime descriptor (`capabilities/zai/`)
Modelled directly on `capabilities/qwen/capability.json` (the closest analog: a CN model vendor
delivered through OpenAI/Anthropic-compatible protocols on top of an existing CLI).

```jsonc
// capabilities/zai/capability.json
{
  "id": "zai",
  "role": "runtime",
  "version": "1.7.0-rc.2",
  "title": "Z.ai GLM Coding Plan",
  "description": "Z.ai GLM Coding Plan (Zhipu) — GLM-4.x/5 model family over Anthropic/OpenAI-compatible endpoints; nested-skill artifact layout; settings-json hook surface when run under a host that supports it; tier-2 support.",
  "tier": "core",
  "requires": [],
  "engines": { "gsd": ">=1.6.0" },
  "runtime": {
    "configHome": {
      "kind": "dot-home",
      "name": ".zai",
      "env": ["ZAI_CONFIG_DIR"]
    },
    "localConfigDir": ".zai",
    "configFormat": "settings-json",
    "artifactLayout": {
      "global": [
        {
          "kind": "skills",
          "destSubpath": "skills",
          "prefix": "gsd-",
          "nesting": "nested",
          "recursive": false,
          "converter": "convertClaudeCommandToClaudeSkill"
        }
      ],
      "local": [
        {
          "kind": "skills",
          "destSubpath": "skills",
          "prefix": "gsd-",
          "nesting": "nested",
          "recursive": false,
          "converter": "convertClaudeCommandToClaudeSkill"
        }
      ]
    },
    "commandStyle": "slash-hyphen",
    "hooksSurface": "settings-json",
    "hookEvents": "claude",
    "sandboxTier": "none",
    "supportTier": 2,
    "installSurface": "settings-json",
    "writesSharedSettings": true,
    "permissionWriter": null,
    "extendedHookEvents": ["SubagentStop", "Stop", "PreCompact"],
    "hostIntegration": {
      "embeddingMode": "imperative",
      "commandSurface": "slash-file",
      "dispatch": {
        "namedDispatch": true, "nested": false, "maxDepth": 1,
        "background": true, "subagentToolkit": "full", "backgroundDispatch": false
      },
      "modelMode": "passive",
      "hookBus": "host",
      "stateIO": "filesystem",
      "transport": "mcp",
      "runtime": "node"
    }
  }
}
```

**Decisions to confirm at Phase 1 review:**
- `configHome.name`: `.zai` vs `.glm` vs `.zai-code`. Recommendation: **`.zai`** (short, matches
  the `zai-org` identity and the `ZAI_CONFIG_DIR` env var we'll register).
- Whether to also declare a hook surface. Since GLM Coding Plan's dominant host is Claude Code
  (which reads `settings.json` hooks), inherit `settings-json` + Claude hook events like `qwen` does.

**Exit criteria:** `node scripts/gen-capability-registry.cjs --check` passes with the new descriptor;
`tests/capability-registry.test.cjs` green.

### Phase 2 — Name-policy & alias registry
- [ ] `src/runtime-name-policy.cts` → add to `FALLBACK_ALIASES`:
  ```ts
  zai: ['zai', 'zai-cli', 'z-ai', 'glm', 'glm-code', 'zhipu'],
  ```
- [ ] `gsd-core/bin/shared/runtime-aliases.manifest.json` → add the same `zai` alias list (the
  runtime-name policy loader prefers this manifest; the `.cts` map is the fallback).
- [ ] `src/runtime-name-policy.cts` → `getProjectInstructionFile()` already defaults
  unknown/AGENTS-native runtimes to `AGENTS.md`, so `zai` resolves correctly with no extra case.
  Confirm via test rather than adding a branch.

**Exit criteria:** `tests/runtime-name-policy.test.cjs` + `tests/non-claude-runtimes-registry-derivation.test.cjs` green;
`canonicalizeRuntimeName('glm')` → `'zai'`.

### Phase 3 — Model catalog (GLM models + Z.ai provider preset)
Edit `gsd-core/bin/shared/model-catalog.json`:

- [ ] Add `runtimeTierDefaults.zai` (GLM model ids per tier). **Pending real-world verification**
  of the exact model strings the Coding Plan exposes, but proposed from Z.ai's published model line:
  ```jsonc
  "zai": {
    "opus":   null,
    "sonnet": null,
    "haiku":  null
  }
  ```
  Start with `null` per tier (same as `kimi`) to let the host's configured model win; refine to
  concrete ids (`glm-4.7`, `glm-4.7-flash`, `glm-5`, `glm-5.2`, `glm-5-turbo`) in Phase 5 once we
  can query the live endpoint.
- [ ] Add `providerPresets.zai` with the Z.ai endpoint + auth template:
  ```jsonc
  "zai": {
    "baseURL": "https://api.z.ai/api/anthropic",
    "authEnv": ["ZHIPU_API_KEY", "ANTHROPIC_AUTH_TOKEN"],
    "protocol": "anthropic-compatible",
    "note": "GLM Coding Plan exposes an Anthropic-compatible surface at /api/anthropic and an OpenAI-compatible surface at /api/paas/v4. Set ANTHROPIC_BASE_URL=https://api.z.ai/api/anthropic for Claude-Code-style hosts."
  }
  ```
  *(Endpoint paths to be confirmed against `docs.z.ai/devpack/tool/others` during Phase 5 live check.)*

**Exit criteria:** `tests/model-catalog-runtime-defaults.test.cjs` green; JSON validates.

### Phase 4 — Installer surface (`bin/install.js`)
- [ ] `selectRuntimesFromArgs()` (line 437): add `'zai'` to the default runtime list and a
  `--zai` flag branch (lines 444–458).
- [ ] Help text (line 639): add `--zai  Install for Z.ai GLM Coding Plan only` and a usage example.
- [ ] Env-var note (line ~1412): append `ZAI_CONFIG_DIR` to the precedence list.
- [ ] If `bin/install.js` has a `qwen`-specific branch (e.g. line 1903
  `if (runtime === 'qwen') { ... }`), mirror it for `zai` only if behaviour is actually needed —
  otherwise rely on the generic `settings-json` install path shared with qwen.

**Exit criteria:** `npx @opengsd/gsd-core --zai --global` installs into `~/.zai`; `tests/install.test.cjs`
and `tests/multi-runtime-select.test.cjs` green.

### Phase 5 — Live verification & tuning (manual, against a real GLM Coding Plan)
- [ ] Provision a GLM Coding Plan subscription + `ZHIPU_API_KEY`.
- [ ] Install GSD into a Claude Code pointed at Z.ai's Anthropic-compatible endpoint
  (`ANTHROPIC_BASE_URL=https://api.z.ai/api/anthropic`).
- [ ] Run `/gsd-new-project`, then one full Discuss→Ship phase on a sample task.
- [ ] Capture: which GLM model id each tier (opus/sonnet/haiku) actually resolves to, compaction
  behaviour, and whether `SubagentStop`/`Stop`/`PreCompact` hooks fire. Feed findings back into
  Phase 3 model ids and the `extendedHookEvents` list.
- [ ] Confirm the `provider-aware compaction tuning` mentioned in upstream GSD issue #4642 is
  exercised correctly for GLM.

**Exit criteria:** one green end-to-end GSD phase on a GLM model; model ids in the catalog match reality.

### Phase 6 — Tests, docs, release
- [ ] Parity tests modelled on the `kimi`/`qwen` ones:
  - `tests/zai-agent-converter.test.cjs`
  - `tests/zai-skill-converter.test.cjs`
  - extend `tests/multi-runtime-select.test.cjs`, `tests/runtime-name-policy.test.cjs`,
    `tests/model-catalog-runtime-defaults.test.cjs`.
- [ ] Add a changeset (`.changeset/<something>.md`) following the repo's changeset convention.
- [ ] README + localized READMEs (en, zh-CN, ja-JP, ko-KR, pt-BR): add Z.ai to the supported-runtimes line.
- [ ] Add `docs/how-to/install-on-your-runtime.md` section for Z.ai / GLM Coding Plan.
- [ ] Run the full gate: `npm test` (vitest→node:test suite, ~526 test files), ESLint,
  `gen-capability-registry --check`.
- [ ] Open PR `feat: add zai runtime (Z.ai GLM Coding Plan)` against `open-gsd/gsd-core` upstream
  (the repo is MIT and accepts runtime contributions — see `CONTRIBUTING.md`).

---

## 5. Risk register

| Risk | Likelihood | Mitigation |
|---|---|---|
| Z.ai has no native CLI, so "zai runtime" is a provider identity more than a host process | High | Document clearly: the `zai` runtime is the GSD-side anchor for the GLM Coding Plan regardless of which host CLI carries the calls. This is the same shape as `qwen`. |
| Model ids / endpoint paths guessed in Phase 3 are wrong | Medium | Start tier maps at `null`; pin concrete ids only after Phase 5 live check. |
| Upstream merges a conflicting runtime addition | Low | All edits are additive/append-only; expect clean merge. |
| Hook events (`PreCompact` etc.) don't fire the same way under GLM | Medium | Declared tier-2; Phase 5 verifies and trims `extendedHookEvents` if needed. |
| Registry generator rejects the descriptor shape | Low | Phase 1 runs `--check` immediately; qwen manifest is a proven template. |

---

## 6. Out of scope (explicitly)

- Writing a new native Z.ai CLI. (Doesn't exist upstream; out of scope for a GSD *runtime* add.)
- Re-routing GSD's own model calls (GSD is model-passive — `modelMode: "passive"` — it never calls
  models directly; the host does.)
- Forking core loop, skills, or agents. All reused unchanged.
- Non-GLM Zhipu products (CogVLM, CogVideoX, etc.) — not part of the Coding Plan.

---

## 7. Immediate next step

Start Phase 0 → Phase 1: cut `feat/zai-runtime`, add `capabilities/zai/capability.json`, and run
`node scripts/gen-capability-registry.cjs --check` to validate the descriptor before touching any
other file. Everything downstream keys off that manifest passing the gate.
