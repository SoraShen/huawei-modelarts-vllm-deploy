# Whisper Sunbird（仅供参考）

镜像标签、卡数和启动参数会随版本更新。下面是一份在南非公共池跑通过的记录，不是永久配置。部署前对照 [asr-custom.md](../asr-custom.md) 和本账号已跑通的服务。不要沿用别的账号的 SWR、OBS 或 DEW。

模型：`Sunbird/asr-whisper-51-african-languages`。

| Field | Value |
|---|---|
| Image | `swr.af-south-1.myhuaweicloud.com/<ns>/whisper-custom:v0.23`（FROM `quay.io/ascend/vllm-ascend:v0.23.0`，只要 CANN/`torch_npu`） |
| Runtime | `transformers` + `torch_npu` FastAPI（[templates/whisper/](../../templates/whisper/)） |
| NPU | **1** |
| Flavor | `modelarts.bm.arm.24u.192g.npu.1d910b` |
| Cmd | `bash /code/serve.sh`。不要 `vllm serve`，不要 `faster-whisper` |
| Mounts | `obs://<bucket>/weight/` → `/weight/`，`obs://<bucket>/code/` → `/code/` |
| Health | HTTP `/health`，initial delay 480s，period 10s，timeout 10s，failure threshold 18 |
| Languages | `language_tokens.py` 里的 ISO-639-3（`swa`、`eng`、`afr`、`zul` 等）。请求必须带 `language`，不要自动检测 |

挂载根上必须有这些文件（缺 `model.safetensors` 或缺 `language_tokens.py` 都导致过失败）：

| Mount | Must contain |
|---|---|
| `/weight/` | `config.json`、`model.safetensors`、tokenizer、`preprocessor_config.json` / `processor_config.json` |
| `/code/` | `serve.sh`、`server.py`、`language_tokens.py` |
