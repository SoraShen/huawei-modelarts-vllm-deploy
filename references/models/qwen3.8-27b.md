# Qwen3.8-27B（仅供参考）

镜像标签、卡数和启动参数会随 vLLM-Ascend 版本更新。下面是一份在南非公共池跑通过的记录，不是永久配置。部署前对照当时的教程，以及本账号里已跑通的服务。不要沿用别的账号的 SWR、OBS 或 DEW。

| Field | Value |
|---|---|
| Weights | `Eco-Tech/Qwen3.8-27B-w8a8`（`Qwen/Qwen3.8-27B` 的 W8A8）→ 平铺到 `obs://<bucket>/weight/`。挂载用 **FILE**，不要用 MODEL：`{"source":"OBS","type":"FILE","address":"obs://<bucket>/weight/","mount_path":"/weight/"}`。同一套权重，FILE 约 11 分钟进入运行；MODEL 会耗满 60 分钟部署超时。`--quantization ascend` 需要这份量化权重 |
| Upstream image | `quay.io/ascend/vllm-ascend:qwen3.8-a2`（不是 `v0.23.0`） |
| SWR | `swr.af-south-1.myhuaweicloud.com/<ns>/vllm-ascend:qwen3.8-a2`（`docker pull` → `tag` → `push`，不用重建） |
| NPU | **2**，TP 2。`unit_configs[0].count` = **1**；卡数在 flavor 里 |
| Flavor | `modelarts.bm.arm.48u.384g.npu.2d910b` |
| Health | HTTP `/health`，initial delay 600s，period 30s，timeout 30s，failure threshold 40 |

```bash
vllm serve /weight --tensor-parallel-size 2 --quantization ascend \
  --served-model-name qwen3.8 --max-model-len 131072 --max-num-seqs 32 \
  --max-num-batched-tokens 16384 --gpu-memory-utilization 0.85 \
  --port 8000 --host 0.0.0.0
```

不要用 1 卡、TP 1 或 `v0.23.0`。

权重必须是 W8A8：10 个 `quant_model_weights-*.safetensors`，合计约 30GB。18 个 `model-*.safetensors`、合计约 52GB 的未量化 checkpoint 对不上 `--quantization ascend`，容器会 BackOffStart。两边镜像 digest 相同也没用。
