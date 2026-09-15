# Huawei ModelArts vLLM Deploy — Cursor Agent Skill

A [Cursor](https://cursor.com) Agent Skill for deploying LLM/VL/ASR models to **Huawei Cloud ModelArts** real-time inference on Ascend NPU (Snt9b2 / Atlas A2), using AK/SK signed REST calls — no CLI package required.

## What it does

- **Runtime gate**: checks the [vLLM-Ascend support matrix](https://docs.vllm.ai/projects/ascend/en/latest/user_guide/support_matrix/supported_models.html) to pick `vllm serve` vs custom FastAPI (Whisper ASR).
- **Signed REST**: signs all ModelArts / OBS / SWR / ECS / DEW calls with `scripts/huawei_signed.py` (Huawei Cloud SDK signer).
- **Full deploy pipeline**: intake → ARM ECS prep → agency → SWR image → OBS weights → DEW secret → CreateInferService → API key → health + task probe → cleanup.
- **LoRA merge**: on-prep-VM merge with swap + config-drift fix.
- **Custom ASR**: Whisper on `torch_npu` via FastAPI (vLLM does not support Whisper on Ascend).

## Files

```
SKILL.md                         # main skill instructions
scripts/huawei_signed.py         # AK/SK REST signer
templates/whisper/               # custom ASR runtime (serve.sh, server.py, language_tokens.py)
references/asr-custom.md          # custom ASR reference
```

## Usage in Cursor

Copy this directory to `~/.cursor/skills/huawei-modelarts-vllm-deploy/` and attach the skill in chat.

## Security

- Never stores AK/SK, passwords, or tokens in files — all via env vars / session only.
- No account-specific info (IPs, project IDs, bucket names) in the skill itself.

## License

MIT
