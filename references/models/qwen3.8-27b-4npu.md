# Qwen3.8-27B 专属池 4 卡（仅供参考）

这是**专属池 4 卡**记录。公共池 **2 卡**（TP 2，`max-model-len` 131072，served name `qwen3.8`）在 [qwen3.8-27b.md](qwen3.8-27b.md)。两份的 flavor、TP、上下文和 served name 各自独立，不要互相覆盖。

命令抄自 2026-10-21 仍在运行的服务配置。操作笔记里的投机解码 JSON 只有 `method` 和 `num_speculative_tokens`。线上 `--speculative-config` 还带 `"enforce_eager": true`。下面以线上命令为准。这套参数是打开 MTP 之后，和 Lite Server 上 8 卡节点里用 4 卡的 Qwen3.8 对齐的那一版。

| Field | Value |
|---|---|
| Weights | 与 2 卡页相同：`Eco-Tech/Qwen3.8-27B-w8a8`（W8A8，约 30GB，`quant_model_weights-*.safetensors`）平铺到 `obs://<bucket>/weight/`，**FILE** 挂到 `/weight/`。约 52GB 的未量化 checkpoint（`model-*-of-00018.safetensors`）对不上 `--quantization ascend`。分片名单见 [qwen3.8-27b.md](qwen3.8-27b.md) |
| Image | `swr.af-south-1.myhuaweicloud.com/<ns>/vllm-ascend:qwen3.8-a2`。标签名与 2 卡页相同。操作笔记称这张镜像为 vLLM-Ascend v0.23.0；服务 JSON 只返回标签，没有单独的版本字段。保持这个标签，不要换成通用标签 `vllm-ascend:v0.23.0` |
| NPU | **4**，TP 4。`unit_configs[0].count` = **1**；卡数在 flavor 里 |
| Flavor | `modelarts.bm.arm.96u.768g.npu.4d910b`（4× Snt9b2 / 910B） |
| Pool | 跑在专属池上。已有专属池时，创建和更新都必须带该池 id；除非这次明确要求共享池，否则不要发共享池。池 id 从 `GET /pools` 取，不要从本文抄 |
| Health | HTTP `/health`，initial delay 600s，period 30s，timeout 30s，failure threshold 40 |

```bash
vllm serve /weight --tensor-parallel-size 4 --quantization ascend \
  --served-model-name qwen3.8-reasoning --max-model-len 133120 --max-num-seqs 32 \
  --max-num-batched-tokens 16384 --gpu-memory-utilization 0.85 \
  --enable-prefix-caching \
  --speculative-config '{"method":"qwen3_5_mtp","num_speculative_tokens":3,"enforce_eager":true}' \
  --port 8000 --host 0.0.0.0
```

网关后来把客户端模型名别名成 `Qwen3.8-27B`。vLLM 的 `--served-model-name` 仍是 `qwen3.8-reasoning`。不要把 served name 改成网关别名。

`enforce_eager` 写在 `--speculative-config` 的 JSON 里。这不是旧版 `serve.sh --enforce-eager` 包装。

和 2 卡页相同的参数：`--quantization ascend`、`--max-num-seqs 32`、`--max-num-batched-tokens 16384`、`--gpu-memory-utilization 0.85`、FILE 挂载、镜像标签 `qwen3.8-a2`。

| | 公共池 2 卡 | 专属池 4 卡（本页） |
|---|---|---|
| NPU / TP | 2 / 2 | 4 / 4 |
| Flavor | `modelarts.bm.arm.48u.384g.npu.2d910b` | `modelarts.bm.arm.96u.768g.npu.4d910b` |
| `max-model-len` | 131072 | 133120 |
| `--served-model-name` | `qwen3.8` | `qwen3.8-reasoning` |
| 前缀缓存 | 命令里没有该参数 | `--enable-prefix-caching` |
| 投机解码 | 无 | `qwen3_5_mtp`，`num_speculative_tokens` 3，`enforce_eager` true |
| 池 | 公共池时省略 `pool_id` | 已有专属池时创建和更新都带该池 id |
