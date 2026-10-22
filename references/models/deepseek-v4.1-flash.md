# DeepSeek-V4.1-Flash W4A8 专属池 8 卡（仅供参考）

2026-10-22 核对过两份正在运行的实时服务：镜像、平台命令、flavor、端口和下面这组环境变量一致，每份 `count` 为 1。第二份是另一个服务，不是把第一份的副本数改成 2。Qwen3.8 的 2 卡页和 4 卡页不要改。

平台 `cmd` 与镜像入口一致，仍然是：

```bash
bash /opt/dsv41/modelarts/modelarts_entrypoint.sh serve
```

这个入口会先写死一组起服开关，再 `exec bash /opt/dsv41/scripts/serve_v2.sh`。进程行以这两份脚本为准。控制台里的 `BAT_TOKENS`、`BLOCK`、`PREFIX`、`SPEC`、`SP_TOKENS`、`KV_DTYPE` 不会进进程：入口在调用 `serve_v2.sh` 之前把它们赋成下面的值。`GPU_UTIL`、`MAX_LEN`、`MAX_SEQS`、`PORT`、`SERVED_NAME`、`MODEL` 会读环境变量，缺省就是表里的值。`TP` 被入口写成 8。

`serve_v2.sh` 实际执行的命令（入口赋值之后，再加上它自己的默认项）：

```bash
vllm serve /data/models/DeepSeek-V4.1-Flash-w4a8-Ascend \
  --host 0.0.0.0 --port 8020 --served-model-name deepseek-v41 \
  --tensor-parallel-size 8 --enable-expert-parallel \
  --trust-remote-code --dtype bfloat16 --kv-cache-dtype bfloat16 \
  --max-model-len 1048576 --max-num-seqs 32 --max-num-batched-tokens 8192 \
  --block-size 128 --gpu-memory-utilization 0.92 \
  --additional-config '{"enable_engram":true,"enable_cpu_binding":true,"ascend_compilation_config":{"enable_npugraph_ex":true,"enable_static_kernel":true},"multistream_overlap_shared_expert":false,"multistream_dsv4_dsa_overlap":true,"enable_prefill_mc2":false,"enable_mc2_hierarchy_comm":false,"engram_storage":"int8"}' \
  --quantization ascend \
  --compilation-config '{"cudagraph_mode": "FULL_DECODE_ONLY", "cudagraph_capture_sizes": [1,2,3,4,6,8,12,16,20,24,32,40,48,96,192]}' \
  --enable-prefix-caching \
  --speculative-config '{"method":"dspark","num_speculative_tokens":5,"enforce_eager":true}' \
  --model-loader-extra-config '{"enable_multithread_load":true,"num_threads":128}' \
  --safetensors-load-strategy lazy \
  --limit-mm-per-prompt '{"image": 4}' \
  --tokenizer-mode=deepseek_v41 --reasoning-parser=deepseek_v41 \
  --tool-call-parser=deepseek_v41 --enable-auto-tool-choice \
  --default-chat-template-kwargs={"enable_thinking":false}
```

`--additional-config` 和 `--model-loader-extra-config` 在 `serve_v2.sh` 里，属于进程命令。一份较短的记录曾经把这两项省掉。

| Field | Value |
|---|---|
| Weights | ModelScope `chiro2001/DeepSeek-V4.1-Flash-w4a8-Ascend`（Ascend W4A8）。镜像清单约 490 GiB：72 个 `quant_model_weights-*-of-00072.safetensors`、`mtpq-*.safetensors`、`engram_int8/`、`optional/quarot.safetensors`，`config.json` 在前缀根上。**FILE**、只读，挂到 `/source/`。入口看到 `/source/config.json` 后用 `stage_weights.py` 校验并拷到本地 `MODEL`（默认 `/data/models/DeepSeek-V4.1-Flash-w4a8-Ascend`），`vllm serve` 读的是这块本地盘 |
| Image | `swr.af-south-1.myhuaweicloud.com/<ns>/vllm-ascend:deepseek-v4.1-flash-w4a8-modelarts-a2-cf57391d808d`。投机解码代码按 vLLM **0.27.1** 分支（`vllm_version_is("0.27.1")`）。换账号只换 `<ns>` |
| NPU | **8**，TP 8。`unit_configs[0].count` = **1**；卡数在 flavor 里 |
| Flavor | `modelarts.bm.arm.192u.1536g.npu.8d910b`（8× Snt9b2 / 910B）。池节点规格 `modelarts.bm.npu.arm.8snt9b2.d` 是整机，不要把它填进 `flavor` |
| Pool | 跑在专属池上。已有专属池时，创建和更新都必须带该池 id；除非这次明确要求共享池，否则不要发共享池。池 id 从 `GET /pools` 取，不要从本文抄 |
| Port | 容器和 `service_invoke` 都是 **8020**。不要照共用模板写成 8000 |
| Served name | `deepseek-v41` |
| Context | `max-model-len` **1048576** |
| Speculative | `method` `dspark`，`num_speculative_tokens` **5**，`enforce_eager` true。图模式 `FULL_DECODE_ONLY`。入口把 `SPEC_EAGER=1`、`DSPARK_GRAPH_CAPTURE_METADATA=0` 写死，draft 不入图 |
| Health | 见下表。部署超时 **180** 分钟。请求超时 **900** 秒。`rate_limit` 200 / `SECONDS` |

| Probe | Live setting |
|---|---|
| Startup | EXEC `/usr/bin/test -f /tmp/deepseek-startup-begun`，initial 5s，period 10s，timeout 10s，failure 30 |
| Readiness | HTTP `/health`，initial 5s，period 10s，timeout 10s，failure 3 |
| Liveness | HTTP `/health`，initial **3600**s，period 30s，timeout 10s，failure 5 |

启动探针那个标记文件由镜像里的日志包装创建，而且只有设置了 `DSV41_LOG_PUT_URLS` 才会走到这段。不要把现成的预签名上传地址抄进新账号。新部署若不做这路上报，把 startup 改成 HTTP `/health` 并给够初始延迟（本地铺权重加上编译），readiness 仍用 `/health`。

入口在起服前做硬件门禁：`/etc/hccn.conf` 要描述 8 个设备，`/dev/shm` 至少 512 GiB，主机驱动版本串含 `25.5.`，`npu-smi` 要看到 8 张健康的 910B3。权重铺完后还要有 `optional/quarot.safetensors`、`config.json`、量化 index，以及可写的 `engram_int8/*.safetensors`（Engram 运行时要做 host register）。

`MULTISTREAM` 不要设置。`serve_v2.sh` 默认是 0，所以上面的 `multistream_overlap_shared_expert` 为 false。`multistream_dsv4_dsa_overlap` 为 true 是 `DSA_OVERLAP` 的默认值 1，入口没有改它。只多开 `MULTISTREAM=1` 的试验服务不是这份配方。

`--block-size` 是 128。同一镜像在 2026-10-20 的 `/metrics` 里，混合 KV 初始化之后把 `cache_config.block_size` 报成 32，同时 `user_specified_block_size` 仍为 true。那是引擎改写后的上报，不是启动参数。这次核对没有重新抓 `/metrics`。
