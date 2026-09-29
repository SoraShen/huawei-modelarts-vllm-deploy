# Model recipes（仅供参考）

每个模型单独一份，镜像和部署方案会更新。这里只做索引和跨模型的坑。部署前以当时的教程和本账号已跑通的服务为准。

Johannesburg NPU：Snt9b2 = Ascend 910B3 = **A2**。指南里的「8× 910B3」是公共池节点 `modelarts.bm.npu.arm.8snt9b2.d`，不是准备机规格。公共池单服务最多可调度 8 张卡。专属池和 SFS Turbo 不是下面这两个参考部署的前提。

| Service flavor | NPU |
|---|---|
| `modelarts.bm.arm.24u.192g.npu.1d910b` | 1 |
| `modelarts.bm.arm.48u.384g.npu.2d910b` | 2 |

| Model | Note |
|---|---|
| [Qwen3.8-27B](models/qwen3.8-27b.md) | `qwen3.8-a2`，2 卡，TP 2，FILE 挂载。权重必须是约 30GB 的 W8A8，不是约 52GB 的未量化 checkpoint |
| [Qwen3-VL-8B](models/qwen3-vl-8b.md) | `qwen3-vl-8b:v1`，1 卡，`bash /code/serve.sh` |
| [Whisper Sunbird](models/whisper-sunbird.md) | 自定义镜像，1 卡，`bash /code/serve.sh` |

换账号时只换 `<ns>` / `<bucket>` / DEW。不要复用别的账号的 SWR、OBS 或 DEW。

## v2 CreateService body (returned HTTP 200)

`POST https://modelarts.{ma_region}.myhuaweicloud.com/v2/{project_id}/services`。只保留形状，数值从对应模型的参考文件填。

创建请求里 `version` 是字符串。GET 返回的 `version` 对象不能原样 POST。组上的 `weight` 是流量权重整数（100），不是模型挂载。

```json
{
  "name": "<service>", "type": "REAL_TIME", "workspace_id": "0",
  "version": "1.0.0", "deploy_timeout_minutes": 60,
  "runtime_config": {
    "service_invoke": {"auth_type": "API_KEY", "protocol": "HTTP", "port": 8000},
    "service_limit": {"rate_limit": {"num": 200, "unit": "SECONDS"}, "request_timeout": 180, "request_size_limit": 50}
  },
  "group_configs": [{
    "name": "<group>", "count": 1, "weight": 100,
    "secret_type": "DEW", "secret_name": "<this-account DEW secret>",
    "unit_configs": [{
      "name": "role-0", "count": 1, "port": 8000,
      "flavor": "<service flavor>",
      "image": {"source": "SWR", "swr_path": "swr.<region>.myhuaweicloud.com/<ns>/<name>:<tag>"},
      "cmd": "<recipe cmd>",
      "files": [{"source": "OBS", "type": "FILE", "address": "obs://<bucket>/weight/", "mount_path": "/weight/", "read_only": true}],
      "startup_health": {"check_method": "HTTP", "protocol": "HTTP", "url": "/health", "initial_delay_seconds": 600, "period_seconds": 30, "timeout_seconds": 30, "failure_threshold": 40}
    }]
  }]
}
```

- 权重挂载用 `files` + `type: FILE`。`type: MODEL` 在 Qwen3.8 上会耗满 60 分钟部署超时；FILE 约 11 分钟进入运行。
- 创建前先列出挂载前缀。分片文件名必须和启动命令匹配。`OBS mount config settled` 只说明挂上了；紧接着的 `BackOffStart` 是容器崩溃（权重不对），不是挂载慢，也不是镜像 digest 不对。Qwen3.8 要 10 个 `quant_model_weights-*.safetensors`（约 30GB）。18 个 `model-*-of-00018.safetensors`（约 52GB）对不上 `--quantization ascend`。
- `image` 必须是对象 `{source: SWR, swr_path}`。字符串会被拒绝。
- `secret_type` 为 `DEW`。密钥里是 `accessKeyId` / `secretAccessKey`。
- `rate_limit` 在 `runtime_config.service_limit` 下，缺了会 `ModelArts.8037`。
- 字段名是 `flavor`。端口 8000。

## 跨模型的坑

- `obsutil` 拷目录会套两层（`prefix/dir/dir/`）。按文件上传。
- `obsutil ls obs://<不存在的桶>` 仍会打印桶名。要看对象或错误行，不要只看桶名。
- `docker login` 的仓库地址是 `swr.<region>.myhuaweicloud.com`。令牌来自 `swr-api.<region>`。不写地址会登到 docker.io。
- Euler 的 docker 网桥上不了 PyPI。用 `docker run --network host` 装包，再 `docker commit`。
- 新账号用自己的 OBS、SWR、DEW 和准备机。
