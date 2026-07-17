# Multi-Turn 对话性能测试 - 本地使用说明

本仓库用于对 LLM 推理服务（当前主要面向 vLLM）进行**多轮对话**（Multi-Turn）在线性能压测，
核心脚本为 `benchmark_serving_multi_turn.py`。本文档说明如何在本地手动执行测试，
以及如何通过 Jenkins 流水线自动触发执行。

> 如需了解合成对话数据格式、随机分布定义、ShareGPT 数据转换等更通用的用法，
> 请参考上游 [`README.md`](README.md)。本文件聚焦于**本仓库当前的实际测试逻辑与运行方式**。

---

## 目录

- [1. 概述与产物](#1-概述与产物)
- [2. 环境准备](#2-环境准备)
  - [2.1 本地手动执行的环境准备](#21-本地手动执行的环境准备)
  - [2.2 Jenkins 流水线的环境准备](#22-jenkins-流水线的环境准备)
- [3. 手动执行测试](#3-手动执行测试)
- [4. 通过 Jenkins 流水线触发执行](#4-通过-jenkins-流水线触发执行)
- [5. 测试参数说明](#5-测试参数说明)
- [6. 常见问题](#6-常见问题)

---

## 1. 概述与产物

### 工作原理

`benchmark_serving_multi_turn.py` 通过多客户端并发向 OpenAI 兼容的 Chat Completions 接口
发送多轮对话请求，采集每轮请求的时延指标（TTFT、TPOT、端到端时延、输入/输出 token 数等），
最终汇总输出统计报告。

- 输入对话来源有两种：
  1. **合成对话**：通过 `generate_multi_turn.json` 这类配置文件 + 文本语料（`pg1184.txt`）随机生成；
  2. **ShareGPT 真实对话**：使用 `convert_sharegpt_to_openai.py` 转换后的数据集。
- 本仓库默认使用 `generate_multi_turn.json`（合成对话，300 轮，每轮 12~18 轮对话）。

### 产出物

| 产出 | 来源 | 说明 |
| --- | --- | --- |
| 控制台统计摘要 | `process_statistics()` 打印 | 含 `runtime_sec`、`requests_per_sec` 及 TTFT/TPOT/latency 等分位数表 |
| `output.json` | `--output-file` | 带 LLM 实际回答的对话记录 |
| `stats.json` | `--stats-json-output` | 每条请求的逐项指标（ttft_ms、tpot_ms 等） |
| `.xlsx` | `--excel-output` | 统计表格（可选） |
| `test_<BUILD>.log` | Jenkins 流水线 | 完整运行日志 |

---

## 2. 环境准备

### 2.1 本地手动执行的环境准备

#### (1) Python 依赖

建议使用 Python 3.10+。安装依赖：

```bash
pip install -r requirements.txt
```

`requirements.txt` 包含：`numpy`、`pandas`、`aiohttp`、`transformers`、`xlsxwriter`、`tqdm`。

#### (2) 模型文件

准备一份本地模型文件目录（含 tokenizer），用于：
- 加载 tokenizer（`AutoTokenizer.from_pretrained(args.model)`）做 token 计数；
- 被推理服务加载。

```bash
export MODEL_PATH=/dingofs/data2/userdata/llms/moonshotai/Kimi-K2.6
```

#### (3) 语料文件

合成对话依赖文本语料 `pg1184.txt`（已随仓库提供）。如需更换，请同步修改
`generate_multi_turn.json` 中的 `"text_files"` 字段。

#### (4) 确认已有推理服务地址

默认情况下**已有正在运行的模型服务**，本测试脚本只需连接该服务即可，无需自行启动。
请向服务提供方获取：

- `BASE_URL`：服务的 API 基地址，例如 `http://10.201.149.10:8080`
- `MODEL_NAME`：服务侧实际暴露的模型名（即 vLLM 启动时的 `--served-model-name`），例如 `kimi-k2.5`

```bash
export BASE_URL=http://10.201.149.10:8080
export MODEL_NAME=kimi-k2.5
```

> 注：`MODEL_PATH` 仅用于本地加载 tokenizer 做 token 计数，需与服务的模型一致；
> 被测服务本身由部署方负责维护，本仓库不包含服务启动流程。

#### (5) 连通性自检（可选但推荐）

测试前确认接口可用：

```bash
# 模型列表接口
curl -s -o /dev/null -w "%{http_code}\n" ${BASE_URL}/v1/models

# Chat Completions 接口
curl -s ${BASE_URL}/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model":"kimi-k2.5","messages":[{"role":"user","content":"hello"}],"max_tokens":10}'
```

两者均返回 200 即可继续。

### 2.2 Jenkins 流水线的环境准备

流水线（`Jenkinsfile`）不会在 Jenkins 节点本地执行 Python，而是通过 SSH 登录一台远端主机，
在该主机上启动 vLLM 官方容器并在容器内执行测试。因此需要准备：

| 准备项 | 说明 |
| --- | --- |
| **Jenkins 凭据** `HOST_SSH_KEY` | 远端主机的 SSH 私钥凭据（sshagent 方式），用于免密登录 |
| **远端测试主机** | 默认 `10.201.132.50`，用户 `root`（见 `environment`） |
| **Docker** | 远端主机需已安装 Docker，能拉取 `vllm/vllm-openai:v0.21.0-cu129` 镜像 |
| **代码仓库目录** | 远端主机上的仓库克隆路径，默认 `/dingofs/data2/userdata/liwt/maas-image/multi-turn-test`（参数 `WORK_DIR`），仓库会被挂载进容器 `/workspace/multi-turn-test` |
| **模型文件路径** | `MODEL_PATH` 指定的模型目录需在远端主机上存在（挂载进容器相同路径） |
| **报告目录** | 远端主机上的报告输出目录，默认 `$WORK_DIR/reports` |
| **网络代理** | 容器内 `pip install` 会临时使用 `http://100.64.1.68:1080` 代理（见 `启动容器并运行测试` 阶段） |
| **推理服务** | `BASE_URL` 指向的推理服务需对远端主机（容器 `--network host`）可达 |
| **Jenkins 节点** | 使用 `slave-2` 标签节点运行（见 `agent.label`） |

> 注意：流水线只负责跑压测脚本，**不负责启动被测的推理服务**，请确保 `BASE_URL` 对应的服务已提前启动。

---

## 3. 手动执行测试

在本地（或登录到测试主机后）执行以下命令。参数含义见 [第 5 节](#5-测试参数说明)。

```bash
export MODEL_PATH=/dingofs/data2/userdata/llms/moonshotai/Kimi-K2.6
export BASE_URL=http://10.201.149.10:8080
export MODEL_NAME=kimi-k2.5

python benchmark_serving_multi_turn.py \
    --model $MODEL_PATH \
    --served-model-name $MODEL_NAME \
    --url $BASE_URL \
    --input-file generate_multi_turn.json \
    --output-file ./output.json \
    --stats-json-output ./stats.json \
    --num-clients 10 \
    --max-active-conversations 30 \
    --warmup-step \
    --no-early-stop \
    --trust-remote-code
```

说明：
- `--warmup-step`：先用每个对话的第一轮做预热（不纳入正式统计），可让缓存预热更充分；
- `--no-early-stop`：要求所有客户端跑完，避免单个客户端退出即终止整体测试；
- `--no-stream`：如需关闭流式（sglang 建议关闭），追加该参数；
- 流式开关与参数 `STREAM_MODE` 对应：`STREAM_MODE=false` 时流水线会自动加 `--no-stream`。

成功时控制台会输出类似如下的统计摘要：

```
----------------------------------------------------------------------------------------------------
Statistics summary:
runtime_sec = 215.810
requests_per_sec = 0.769
----------------------------------------------------------------------------------------------------
                   count     mean     std      min      25%      50%      75%      90%      99%      max
ttft_ms           166.0    78.22   67.63    45.91    59.94    62.26    64.43    69.66   353.18   567.54
tpot_ms           166.0    25.37    0.57    24.40    25.07    25.31    25.50    25.84    27.50    28.05
latency_ms        166.0  2591.07  326.90  1998.53  2341.62  2573.01  2860.10  3003.50  3268.46  3862.94
----------------------------------------------------------------------------------------------------
```

如需在容器中手动复现（与 Jenkins 等价），可：

```bash
docker run -d --name multi-turn-test-manual \
    --network host --memory=32g --shm-size=1g --entrypoint bash \
    -v $WORK_DIR:/workspace/multi-turn-test \
    -v $MODEL_PATH:$MODEL_PATH \
    -w /workspace/multi-turn-test \
    vllm/vllm-openai:v0.21.0-cu129 \
    -c "sleep infinity"

docker exec multi-turn-test-manual bash -c \
    "cd /workspace/multi-turn-test && pip install -q -r requirements.txt"

docker exec multi-turn-test-manual bash -c \
    "cd /workspace/multi-turn-test && python benchmark_serving_multi_turn.py \
        --model $MODEL_PATH --served-model-name $MODEL_NAME \
        --url $BASE_URL --input-file generate_multi_turn.json \
        --num-clients 10 --max-active-conversations 30 \
        --warmup-step --no-early-stop --trust-remote-code"

docker rm -f multi-turn-test-manual
```

---

## 4. 通过 Jenkins 流水线触发执行

### 4.1 触发方式

在 Jenkins 上选中本仓库对应的流水线任务，点击 **Build with Parameters（构建并配置参数）**，
填入参数后构建即可。

### 4.2 构建参数

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `TESTER` | `liwt` | 测试人员名称（必填），用于报告目录与邮件 |
| `CHIP` | `nvidia-h100` | 芯片平台名称（必填） |
| `ENGINE` | `vllm` | 推理框架，当前仅支持 `vllm` |
| `PD` | `agg` | PD 分离模式：`agg`=非分离，`disagg`=PD 分离 |
| `MODEL` | `kimi-k2.5` | 模型服务名称（必填，用于 API 调用与 `--served-model-name`） |
| `MODEL_PATH` | `/dingofs/data2/userdata/llms/moonshotai/Kimi-K2.6` | 模型文件本地绝对路径，会挂载进容器相同路径 |
| `BASE_URL` | `http://10.201.149.10:8080` | API 地址（必填） |
| `NUM_CLIENTS` | `10` | 并发客户端数量 |
| `MAX_ACTIVE_CONVERSATIONS` | `30` | 每个客户端的活跃对话槽位数 |
| `INPUT_FILE` | `generate_multi_turn.json` | 输入配置文件名（仓库内相对路径） |
| `STREAM_MODE` | `true` | 是否流式，`false` 会自动追加 `--no-stream`（sglang 建议 `false`） |
| `DESCRIPTION` | （空） | 模型服务的描述信息，仅用于邮件报告 |
| `RECIPIENTS` | `liwt@zetyun.com` | 测试报告邮件接收者（逗号分隔） |
| `WORK_DIR` | `/dingofs/data2/userdata/liwt/maas-image/multi-turn-test` | 远端测试仓库目录（一般勿改） |

> `MODEL` 若包含路径分隔符（如 `/foo/bar/baz`），流水线会自动取最后一段 `baz`
> 作为容器名与报告目录的安全标识；调用 API 时仍使用原始完整名称。

### 4.3 流水线阶段

流水线共 7 个阶段，依次为：

1. **打印测试参数**：在控制台输出本次构建的全部参数，便于追溯。

2. **环境检查**：SSH 登录远端主机，确认 `WORK_DIR` 存在、Docker 可用、`INPUT_FILE` 内容正确。

3. **API 连通性预检**：
   - 检查 `${BASE_URL}/v1/models` 返回 200；
   - 检查 `${BASE_URL}/v1/chat/completions` 发送一条 `hello` 请求返回 200；
   - 任一失败则将构建置为 `UNSTABLE`，并跳过后续“启动容器”阶段（通过 `env.CONNECTIVITY_FAILED` 控制）。
   - 日志同时写入远端 `/tmp/test_${BUILD_NUMBER}.log`。

4. **启动容器并运行测试**（连通性通过时才执行）：
   - 容器名：`multi-turn-test-${CHIP}-${MODEL_NAME}-${BUILD_NUMBER}`；
   - 用 `vllm/vllm-openai:v0.21.0-cu129` 镜像启动 `--network host` 容器，挂载 `WORK_DIR`、`MODEL_PATH`、报告目录；
   - 容器内 `pip install -r requirements.txt`（临时走代理）；
   - 执行 `benchmark_serving_multi_turn.py`，关键参数映射自构建参数：
     ```
     --model ${MODEL_PATH}
     --served-model-name ${MODEL}
     --url ${BASE_URL}
     --input-file ${INPUT_FILE}
     --output-file ${reportsDir}/output.json
     --num-clients ${NUM_CLIENTS}
     --max-active-conversations ${MAX_ACTIVE_CONVERSATIONS}
     ${STREAM_MODE=='false' ? '--no-stream' : ''}
     --warmup-step --no-early-stop --trust-remote-code
     ```
   - 标准输出实时 `tee` 到远端 `/tmp/test_${BUILD_NUMBER}.log`，并尽力复制到报告目录归档。

5. **拉取测试结果**：通过 `scp` 将远端报告目录与本地日志备份拉取到 Jenkins workspace
   的 `builds/${BUILD_NUMBER}/` 下，供归档与邮件使用。

6. **发送邮件**：
   - 解析日志中的 `Parameters:` 与 `Statistics summary:` 区块（会先剥离 ANSI 转义码）；
   - 若识别到连通性检查失败，邮件中额外展示失败原因块；
   - 若日志解析不完整，尝试用 `output.json` 兜底补充“完成对话数”；
   - 生成 HTML 测试报告邮件，发送给 `RECIPIENTS`，附件为 `test_${BUILD_NUMBER}.log`。

7. **清理容器**：删除本次构建启动的容器，保证环境整洁。

构建完成后：
- `post.always` 中 `archiveArtifacts` 归档 `builds/${BUILD_NUMBER}/**`；
- `post.cleanup` 中 `cleanWs()` 清理 workspace。

### 4.4 构建结果判定

| 构建结果 | 触发场景 |
| --- | --- |
| `SUCCESS` | 全流程正常，日志中含完整 Statistics summary |
| `UNSTABLE` | API 连通性预检失败；或拉取结果/发邮件阶段失败（`catchError`） |
| `FAILURE` | “启动容器并运行测试”阶段内执行失败（`stageResult: 'FAILURE'`） |

---

## 5. 测试参数说明

### 5.1 常用 CLI 参数（`benchmark_serving_multi_turn.py`）

| 参数 | 说明 |
| --- | --- |
| `-i/--input-file` | 输入 JSON（合成对话配置或对话列表），**必填** |
| `-o/--output-file` | 输出含 LLM 回答的对话 JSON |
| `-m/--model` | 模型路径（用于加载 tokenizer），**必填** |
| `--served-model-name` | API 实际使用的模型名（不填则同 `--model`） |
| `-u/--url` | API 基地址，默认 `http://localhost:8000` |
| `-p/--num-clients` | 并发客户端数，默认 1 |
| `-k/--max-active-conversations` | 同时活跃对话数上限 |
| `-n/--max-num-requests` | 请求总数上限（所有客户端合计） |
| `--warmup-step` | 先跑预热轮（仅第一轮），不计入正式统计 |
| `--no-early-stop` | 关闭“单客户端退出即停止”行为 |
| `--no-stream` | 关闭流式 |
| `--send-conversation-id` | 注入 `conversation_id`（PD 分离代理用） |
| `--limit-min-tokens` / `--limit-max-tokens` | 覆盖数据集里的输出 token 数（负值禁用） |
| `--request-rate` | 单客户端请求速率（Poisson），0=无延迟 |
| `--max-retries` | 超时请求重试次数，默认 0；可用环境变量 `MULTITURN_BENCH_MAX_RETRIES` |
| `--request-timeout-sec` | 单请求超时秒数，默认 120 |
| `--api-key` / `--header` | API Key 与自定义请求头 |
| `--stats-json-output` | 导出每条请求指标 JSON |
| `-e/--excel-output` | 导出统计 Excel |
| `--max-turns` | 限制每个对话的最大轮数 |
| `--conversation-sampling` | 对话选取策略：`round_robin`(默认) / `random` |
| `--trust-remote-code` | 加载 tokenizer 时信任远程代码 |
| `--seed` | 随机种子，默认 0 |

### 5.2 合成对话配置（`generate_multi_turn.json`）

| 字段 | 说明 |
| --- | --- |
| `num_conversations` | 生成对话总数（默认 300） |
| `text_files` | 抽取文本的语料文件列表（默认 `pg1184.txt`） |
| `prompt_input.num_turns` | 每个对话的轮数分布（默认 uniform 12~18） |
| `prompt_input.common_prefix_num_tokens` | 所有对话共享前缀 token 数 |
| `prompt_input.prefix_num_tokens` | 每个对话独有前缀 token 数分布 |
| `prompt_input.num_tokens` | 每轮 **user** 消息 token 数分布 |
| `prompt_output.num_tokens` | 每轮 **assistant** 消息 token 数分布 |

每个数值字段都需指定一种分布：`constant`、`uniform`、`lognormal`、`zipf`、`poisson`，
完整字段说明见上游 [`README.md`](README.md)。

---

## 6. 常见问题

- **连通性预检失败**：先确认 `BASE_URL` 服务已启动且对测试主机可达，再检查 `MODEL`/`served-model-name` 是否与服务侧一致。
- **容器内 pip 安装失败**：检查远端主机到 `100.64.1.68:1080` 代理是否可达；必要时在容器外预装依赖或换源。
- **DingoFS 报告归档失败**：流水线会回退到 `/tmp/test_${BUILD_NUMBER}.log` 并继续发邮件，不影响主流程。
- **统计为空/邮件显示“无统计数据”**：通常是测试未真正执行（如连通性失败或脚本异常退出），查看 `test_${BUILD_NUMBER}.log` 定位。
- **sglang 框架**：当前 `ENGINE` 仅支持 `vllm`；若选 sglang，建议同时将 `STREAM_MODE` 设为 `false`。
