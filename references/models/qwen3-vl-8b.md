# Qwen3-VL-8B-Instruct（仅供参考）

镜像标签和启动参数会变。下面抄自账号上已停止的服务 `service-qwen3-vl-8b`（南非公共池，1 卡）。部署前再对一下当时的服务，不要复用别的账号的 SWR、OBS 或 DEW。

| Field | Value |
|---|---|
| Weights | `Qwen/Qwen3-VL-8B-Instruct`，约 16GB，4 个 `model-*.safetensors`。挂载地址要指到**内层目录**，让 `config.json` 就在 `/weight/` 根上 |
| Image | `swr.af-south-1.myhuaweicloud.com/<ns>/qwen3-vl-8b:v1`（参考服务在 `rocky-image/qwen3-vl-8b:v1`，不是 `qwen3.8-a2` / `qwen3.8-a3`） |
| NPU | **1**。`unit_configs[0].count` = 1 |
| Flavor | `modelarts.bm.arm.24u.192g.npu.1d910b` |
| Cmd | `bash /code/serve.sh` |
| Health | HTTP `/health`，initial delay 480s，period 10s，timeout 10s，failure threshold 18 |

`serve.sh` 实际执行：

```bash
vllm serve /weight --host 0.0.0.0 --port 8000 \
  --served-model-name Qwen3-VL-8B-Instruct --trust-remote-code \
  --dtype bfloat16 --max-model-len 16384 --max-num-batched-tokens 16384 \
  --tensor-parallel-size 1 --gpu-memory-utilization 0.90 \
  --limit-mm-per-prompt.image 4
```

两个挂载都是 **FILE**，不要用 MODEL：

| Mount | Address |
|---|---|
| `/weight/` | `obs://<bucket>/qwen3-vl-8b/weight/qwen3-vl-8b/` |
| `/code/` | `obs://<bucket>/qwen3-vl-8b/code/`（里面要有 `serve.sh`） |
