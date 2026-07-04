---
type: Added
---
**Z.ai GLM Coding Plan is now a first-class runtime** — GSD Core's Discuss → Plan → Execute → Verify → Ship loop can now anchor to the Z.ai GLM Coding Plan (GLM-4.x/5 model family over Anthropic/OpenAI-compatible endpoints). Install with `npx @opengsd/gsd-core --zai --global` (config home `~/.zai`, env `ZAI_CONFIG_DIR`); aliases `glm`, `zhipu`, `z-ai`, and `glm-code` also resolve to the `zai` runtime. Tier-2 support, modelled on the Qwen Code runtime; concrete per-tier GLM model ids land once verified against a live Coding Plan subscription.
