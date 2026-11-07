# DeepSeek-V4.1-Flash-1017 W4A8 双机 1P1D（仅供参考）

这是 Eco-Tech W4A8 的两台 8 卡 Prefill/Decode，不是 [8 卡 chiro 页](deepseek-v4.1-flash.md)。chiro 那页的镜像标签、`/opt/dsv41` 入口、端口 8020 和单机 TP 8 不要套到这里。这里也不要用通用 `v0.23.0`。

2026-11-07 只读核对过 `me-east-1` 上正在 RUNNING 的实时服务。没有改服务，没有改 OBS。下面的旗标以该服务挂载的脚本为准。换账号只换自己的 SWR 命名空间、OBS 桶和 DEW，不要复用这份服务所在账号的桶、镜像仓库或池 id。池 id 从 `GET /pools` 取。不要 `POST /pools`。

平台命令两边相同，不要改成 chiro 入口，也不要设 `MULTISTREAM`：

```bash
bash /model-scripts/entrypoint.sh --nic-name eth0
```

`image` 必须是 `{"source":"SWR","swr_path":"..."}`。命令不传 `--role`。`get_ip.py` 读同一张 rank table：按 `group_id` 排序后，序号 0 是 Prefill，序号 1 是 Decode。

| Field | Value |
|---|---|
| Weights | ModelScope `Eco-Tech/DeepSeek-V4.1-Flash-1017-w4a8`。`config.json` 在前缀根上：`model_type` `deepseek_v41`，`quantization_config.quant_method` `ascend`，`text_config.max_position_embeddings` 1048576。81 个 `quant_model_weights-*-of-00081.safetensors`，4 个 engram 分片，`optional/quarot.safetensors`，`quant_model_weights.safetensors.index.json`。没有 `model.safetensors.index.json`。只读挂到 `/model-weight/`，`vllm serve` 直接读这个路径，不再拷贝。8 卡混合那份把同一 ModelScope 仓库先挂到 `/source/` 再拷进本地盘，那是另一套启动脚本 |
| Image | 8 卡混合记录用的上游是 `quay.io/ascend/vllm-ascend:v0.30.0`（`linux/arm64`，未再改镜像，不用 `v0.30.0rc2`）。正在跑的 1P1D 填的是 `swr.me-east-1.myhuaweicloud.com/inference-llm/vllm-ascend:DeepSeek-V4.1-Flash-1017-w4a8`。这次没有拆层，不把两个 digest 写成同一个。权重卡 README 另写了测试版本 `v0.30.0rc1` 及更高 |
| NPU | 两个单元，各 8 卡，共 16。每个 `unit_configs[].count` = **1**。卡数在 flavor 里 |
| Flavor | 单元 `modelarts.bm.arm.192u.1536g.npu.8d910b`（8× Snt9b2 / 910B3 / **A2**）。池节点规格 `modelarts.bm.npu.arm.8snt9b2.d` 是整机，不要填进 `flavor` |
| Pool | 专属池。已有专属池时，创建和更新都必须带该池 id。除非这次明确要求共享池，否则不要发共享池 |
| Port | 平台 `service_invoke` 与 `advanced_config.port` 都是 **8000**。引擎从 **7100** 起：Prefill 7100–7101，Decode 7100–7103。见下方注意 |
| Served name | `dsv41` |
| Context | `max-model-len` **1048576**。1048576 / 128 = 8192 |
| Speculative | 两边 `method` `dspark`，`num_speculative_tokens` **5**。Prefill 的 `enforce_eager` 是进程参数。Decode 的 `enforce_eager` 在 speculative-config 里，图模式 `FULL_DECODE_ONLY` |
| Health | 这次服务 JSON 没有存活 / 就绪 / 启动探针字段。部署超时 **60** 分钟。脚本在起代理前对 6 个引擎 `curl --max-time 3`，间隔 5 秒，总等待 36000 秒。请求超时 1200 秒，`rate_limit` 10000 / `SECONDS`，`request_size_limit` 50 |

挂载三条，都是只读：

| 容器路径 | 内容 | 类型 |
|---|---|---|
| `/model-weight/` | 权重前缀，`config.json` 在根上 | `OBSFS` / `MODEL` |
| `/model-scripts/` | 下面这组脚本，根上就是 `entrypoint.sh`，不要再套一层目录 | `OBSFS` / `FILE` |
| `/model-outputlog/` | 入口和每个 DP 的 `tee` 日志 | `OBSFS` / `FILE` |

启动命令会执行的文件：`entrypoint.sh`、`get_ip.py`、`launch_online_dp.py`、`prefill/start_prefill.sh`、`prefill/run_dp_template.sh`、`prefill/load_balance_proxy_server_example.py`、`decode/start_decode.sh`、`decode/run_dp_template.sh`。代理 SHA256 `ed1887e2841e4998b22f6d8f10186a15efae33d1ddf760da0a5cd28c67419acd`。`self_check.py` 只做静态核验，不是平台命令。同目录的 `Start_*.sh`、`entrypoint_bak_1.sh`、`decode/run_dp_template_bak1.sh` 不要被这条命令点名，也不要在准备脚本时覆盖已有的 `Start_DeepSeek-V41-Flash-1017-w4a8-v2.sh` 及其 `.bak`。

| 角色 | Rank | 并行 | 引擎 | KV |
|---|---:|---|---|---|
| Prefill | 0 | DP 2，本机 2，TP 4，专家并行。NPU 0–3 与 4–7 | 7100、7101 | `MooncakeHybridConnector`，`kv_producer`，`kv_port` 30000，`engine_id` 0 |
| Decode | 1 | DP 4，本机 4，TP 2，专家并行。每路 2 卡 | 7100–7103 | `MooncakeHybridConnector`，`kv_consumer`，`kv_port` 30100，`engine_id` 1 |

两边 KV JSON 的拓扑都是 prefill `dp_size` 2 / `tp_size` 4、decode `dp_size` 4 / `tp_size` 2。代理只在 Prefill 上启动，登记这 6 个后端。DP RPC 都是 12321。

环境变量写在两个 `run_dp_template.sh` 里，只传给 `vllm` 子进程。共有：清掉 `http_proxy` / `https_proxy`，`no_proxy` 带两个节点 IP，`HCCL_IF_IP` 与 `VLLM_HOST_IP` 为本机 IP，三个 `*_SOCKET_IFNAME` 用 `--nic-name`，`VLLM_ENGINE_READY_TIMEOUT_S=360000`，`HCCL_CONNECT_TIMEOUT=1200`，`HCCL_INTRA_ROCE_ENABLE=1`，`PYTORCH_NPU_ALLOC_CONF=expandable_segments:True`。Prefill 另加 `HCCL_BUFFSIZE=1024`、`HCCL_OP_EXPANSION_MODE=AIV`、`VLLM_USE_V2_MODEL_RUNNER=0`。Decode 是 `HCCL_BUFFSIZE=1800`、`VLLM_USE_V2_MODEL_RUNNER=1`，没有 `HCCL_OP_EXPANSION_MODE`。平台命令不传 `--jemalloc-lib`，因此不设置 `LD_PRELOAD`。

Prefill `vllm serve /model-weight/` 的固定旗标：`--max-num-batched-tokens 8192`，`--max-num-seqs 16`，`--gpu-memory-utilization 0.9`，`--enforce-eager`，`--enable-prefix-caching`，`--quantization ascend`，`--block-size 128`，`--safetensors-load-strategy lazy`，tokenizer / reasoning / tool parser 都是 `deepseek_v41`，`--enable-auto-tool-choice`，`--seed 1024`。Engram `'{"cpu_offload":true,"dp_shared_memory":true}'`。`additional-config` 是 `enable_cpu_binding`、`enable_dsa_cp`、`enable_flashcomm1`、`enable_shared_expert_dp`。后三项是 A3 开关，A2 的 8 卡混合和 16 卡混合都没有放进命令。这里照实记录当前脚本，不要在 A2 上再补 `enable_fused_mc2`。

Decode 固定旗标：`--max-num-batched-tokens 400`，`--max-num-seqs 32`，`--gpu-memory-utilization 0.95`，`--async-scheduling`，`--no-enable-prefix-caching`，`--compilation-config '{"cudagraph_mode":"FULL_DECODE_ONLY"}'`。`additional-config` 是 `enable_npugraph_ex` true、`enable_static_kernel` true、`enable_cpu_binding`、`multistream_overlap_shared_expert`、`recompute_scheduler_enable`。`decode/run_dp_template_bak1.sh` 仍是 `VLLM_USE_V2_MODEL_RUNNER=0` 和 `enable_static_kernel` false。启动器引用的是不带 bak 的文件。

启动顺序：rank table `completed` 且恰好两个 IP 之后，Decode 拉起四个引擎，Prefill 拉起两个引擎，六个 `/health` 都成功后 Prefill 才执行代理。README 写先观察 Decode 再观察 Prefill。入口没有跨 Pod 的先后锁。表被拆开时自动模式会报错，那时才使用 `--role`、`--prefill-ip`、`--decode-ip`。当前平台命令没有这些参数。

官方 v0.30.0 的 A2 1P1D 是 4 台 32 卡。本包 README 写明改成了 2 台 16 卡。`VALIDATION_REPORT.md` 写报告时还没有真实机通过记录。服务状态后来是 RUNNING，那份报告文件仍在脚本前缀里。

注意，记录现状，不在这里改脚本：

1. 平台端口 8000，引擎 7100–7103。当前 `entrypoint.sh` 的代理默认端口是 **8000**。`entrypoint_bak_1.sh` 默认 **2999**，启动命令不引用它。前缀里没有 `entrypoint_bak.sh`。包内 README 仍写入口要映射到 2999。早先一次读取把 2999 记在 `entrypoint.sh`、把 8000 记在 `entrypoint_bak.sh`。服务 JSON 没有进程监听表，不能据此判断已经在跑的代理绑在哪个端口。`custom_metrics_path` 是 `http,8000,metrics`。
2. Prefill 的 `enable_dsa_cp`、`enable_flashcomm1`、`enable_shared_expert_dp` 出现在 A2 规格上。Decode 没有这三项。
3. 日志挂载 `read_only` 与 `readOnly` 都是 true，而 `entrypoint.sh` 在 `/model-outputlog/` 不可写时退出。服务仍是 RUNNING，不能单凭状态判断写入是否失败。
4. 平台 `mooncake_enable` 是 false，命令仍使用 `MooncakeHybridConnector`。JSON 不能证明平台开关会关掉 connector。
5. Decode `--max-num-batched-tokens` 400 不能被 block size 128 整除。上下文 1048576 可以。服务是 RUNNING，不能据此判断 Decode 有没有因此拒绝启动。

`image.category` 在那份 JSON 里是 `CPU`。规格是 8 卡 NPU 且状态为 RUNNING。这个字段有没有改调度，证据不足。实例组有 `secret_type` 和 `secret_name`；`secret_enable` 为 false，`secret_volumes` 为空。不要把密钥写进脚本或本文。
