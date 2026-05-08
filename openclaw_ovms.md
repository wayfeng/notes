# 使用 OVMS 本地运行 OpenClaw

## 一、环境配置

1. 操作系统推荐使用 Ubuntu 24.04

2. 安装 Intel GPU 驱动（参考 https://dgpu-docs.intel.com/driver/client/overview.html）

3. 安装 Docker Engine (参考 https://docs.docker.com/engine/install/ubuntu/)

4. 安装模型下载和转换工具，建议使用 Python 虚拟环境：

```bash
python3 -m venv venv
source venv/bin/activate
pip install -U pip
pip install optimum[openvino,nncf]
pip install huggingface_hub
```

5. 如果不能访问 Hugging Face，请先设置 HF_ENDPOINT 环境变量指向可用的镜像，例如：

```bash
export HF_ENDPOINT="https://hf-mirror.com"
```

## 二、安装 OVMS

官方推荐通过 Docker 运行 OVMS（OpenVINO Model Server）。

```bash
docker pull openvino/model_server:latest-gpu
```

镜像下载完成后，可以查看本地镜像列表确认下载成功：

```bash
docker images

IMAGE                              ID             DISK USAGE   CONTENT SIZE   EXTRA
openvino/model_server:latest-gpu   08ad73dfd651       1.05GB             0B
```

注意：上面输出只是示例，镜像 ID、大小等会根据实际版本不同而变化。

## 三、支持 Qwen3 MoE 模型

有两种方式将 Qwen3 模型转换为或取得 OpenVINO IR 格式：

1. 直接从 Hugging Face 下载已经转换好的 OpenVINO IR 文件；
2. 从 Hugging Face 的原始模型使用 Optimum 工具导出为 OpenVINO IR。

下面分别说明两种方法及验证步骤。

### 方式 A：直接从 Hugging Face 下载 OpenVINO 模型

以下示例演示下载 Qwen3-30B-A3B 的 OpenVINO IR（权重已压缩为 int4）：

```bash
mkdir -p ./models
hf download OpenVINO/Qwen3-30B-A3B-Instruct-2507-int4-ov --local-dir ./models/qwen3-30b-a3b-ov-int4
```

下载完成后，检查模型目录中文件是否齐全（示例输出）：

```bash
l ./models/qwen3-30b-a3b-instruct-2507-int4-ov
.rw-rw-r--   707 added_tokens.json
.rw-rw-r--  2.6k chat_template.jinja
.rw-rw-r--   963 config.json
.rw-rw-r--   213 generation_config.json
.rw-rw-r--  1.7M merges.txt
.rw-rw-r--   471 openvino_config.json
.rw-rw-r--  2.2M openvino_detokenizer.bin
.rw-rw-r-- 10.0k openvino_detokenizer.xml
.rw-rw-r--   16G openvino_model.bin
.rw-rw-r--  6.3M openvino_model.xml
.rw-rw-r--  5.6M openvino_tokenizer.bin
.rw-rw-r--   27k openvino_tokenizer.xml
.rw-rw-r--  5.7k README.md
.rw-rw-r--   613 special_tokens_map.json
.rw-rw-r--   11M tokenizer.json
.rw-rw-r--  5.4k tokenizer_config.json
.rw-rw-r--  2.8M vocab.json
```

如果某些大文件（例如 openvino_model.bin）下载缓慢或缺失，请确认网络、镜像源及磁盘空间。

### 方式 B：从 Hugging Face 的原始模型用 Optimum 导出 OpenVINO IR

使用 optimum-cli 导出 OpenVINO IR（示例：导出为 int4 权重格式）：

```bash
optimum-cli export openvino --model Qwen/Qwen3-30B-A3B-Instruct-2507 --weight-format int4 --trust-remote-code ./models/qwen3-30b-a3b-ov-int4
```

有关 Optimum Intel 的详细说明，请参阅官方文档：
https://docs.openvino.ai/2026/openvino-workflow-generative/inference-with-optimum-intel.html

### 验证 Qwen3 MoE 模型

下面的命令用于以 Docker 运行 OVMS 并将本地 models 目录挂载为模型仓库。请注意，--source_model 的值应与你在 /models 目录下模型文件夹的名称一致（仅文件夹名，不含路径）。

```bash
docker run --name ovms --user $(id -u):$(id -g) -d --device /dev/dri --group-add=$(stat -c "%g" /dev/dri/render* | head -n 1) --rm -p 8000:8000 -v $(pwd)/models:/models:rw openvino/model_server:latest-gpu --source_model qwen3-30b-a3b-ov-int4 --model_repository_path /models --task text_generation --rest_port 8000 --target_device GPU --cache_dir /models/.ov_cache --enable_tool_guided_generation true --enable_prefix_caching true --tool_parser hermes3
```

说明与验证：
- --device /dev/dri 与 --group-add 用于暴露主机上的 GPU/drm 设备到容器，确保宿主机已安装合适的驱动并允许容器访问。
- -p 8000:8000 将容器 REST 接口映射到宿主机的 8000 端口。

启动后可以通过以下命令检查模型是否已被 OVMS 加载：

```bash
curl http://localhost:8000/v3/models
{"data":[{"id":"qwen3-30b-a3b-ov-int4","object":"model","created":1778052060,"owned_by":"OVMS"}],"object":"list"}
```

也可发送一次简单的 chat/completions 请求以确认推理流程正常：

```bash
curl -s http://localhost:8000/v3/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-30b-a3b-ov-int4",
    "max_tokens": 30,
    "temperature": 0,
    "stream": false,
    "messages": [
      { "role": "system", "content": "You are a helpful assistant." },
      { "role": "user", "content": "What are the 3 main tourist attractions in Paris?" }
    ]
  }' | jq .
```

预期结果是一个包含模型返回的文本内容的 JSON 响应。若收到错误，请查看容器日志（docker logs ovms）以获取详细信息。

### 模型推理性能测试

项目地址与benchmark示例：

```bash
git clone https://github.com/openvinotoolkit/openvino.genai.git
cd openvino.genai/tools
pip install -r requirements.txt
python benchmark.py -m ./models/qwen3-30b-a3b-ov-int4 -n 2 -t text_gen -d GPU -f ov -p "what is openvino?"
```

更多用法查看帮助：

```bash
python benchmark.py -h
```

## 四、支持 Qwen3 VL 模型

### 模型转换

OpenVINO 官方目前没有提供 Qwen3 VL 模型下载，所以这里使用 optimum-cli 导出 OpenVINO IR（示例：导出为 int4 权重格式）：

```bash
optimum-cli export openvino --model "Qwen/Qwen3-VL-8B-Instruct" --trust-remote-code --weight-format int4 --task image-text-to-text ./qwen3-vl-8b-ov-int4
```

检查导出文件：

```bash
 707 added_tokens.json
5.3k chat_template.jinja
1.5k config.json
 213 generation_config.json
1.2k graph.pbtxt
1.7M merges.txt
1.2k openvino_config.json
2.2M openvino_detokenizer.bin
 13k openvino_detokenizer.xml
4.2G openvino_language_model.bin
3.9M openvino_language_model.xml
623M openvino_text_embeddings_model.bin
5.7k openvino_text_embeddings_model.xml
5.6M openvino_tokenizer.bin
 30k openvino_tokenizer.xml
574M openvino_vision_embeddings_merger_model.bin
1.3M openvino_vision_embeddings_merger_model.xml
1.8M openvino_vision_embeddings_model.bin
8.6k openvino_vision_embeddings_model.xml
2.7M openvino_vision_embeddings_pos_model.bin
5.7k openvino_vision_embeddings_pos_model.xml
 782 preprocessor_config.json
 613 special_tokens_map.json
 11M tokenizer.json
5.4k tokenizer_config.json
 817 video_preprocessor_config.json
2.8M vocab.json
```

### 验证模型

要使用本地图片测试OVMS，需要增加 *allowed_local_media_path* 设置。

```bash
docker run -d --name ovms-mm --user $(id -u):$(id -g) --device /dev/dri --group-add=$(stat -c "%g" /dev/dri/render* | head -n 1) --rm -p 8000:8000 -v $(pwd)/models:/models:rw openvino/model_server:latest-gpu --source_model qwen3-vl-8b-ov-int4 --model_repository_path /models --task text_generation --rest_port 8000 --target_device GPU --cache_dir /models/.ov_cache --enable_tool_guided_generation true --enable_prefix_caching true --tool_parser hermes3 --allowed_local_media_path "/home/ubuntu/assets"

docker cp assets ovms-mm:/home/ubuntu/
```

检查模型是否加载：

```bash
curl http://localhost:8000/v3/models
{"data":[{"id":"qwen3-vl-8b-ov-int4","object":"model","created":1778121530,"owned_by":"OVMS"}],"object":"list"}
```

测试推理：

```bash
curl http://localhost:8000/v3/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-vl-8b-ov-int4",
    "temperature": 0.8,
    "max_completion_tokens": 128,
    "messages": [
        { "role": "system", "content": "You are a helpful assistant." },
        {
          "role": "user",
          "content": [
              { "type": "text", "text": "What is on the picture?" },
              { "type": "image_url", "image_url": { "url": "/home/ubuntu/assets/demo.jpeg" } }
          ]
        }
    ]
}'
```

推理正确会输出类似如下 JSON 格式字符串：

```json
{"choices":[{"finish_reason":"length","index":0,"logprobs":null,"message":{"content":"Based on the image provided, here is what is depicted:\n\n*   **Main Subjects:** A woman and her dog are the central focus.\n*   **The Woman:** She is sitting on the sand at the beach, smiling warmly. She has long, dark hair and is wearing a plaid (checkered) shirt and dark pants. She is interacting with the dog.\n*   **The Dog:** It is a large, light-brown dog, likely a Labrador Retriever, sitting on its hindquarters. It is wearing a colorful harness and has its front paw raised, appearing to high-five or interact with the woman's hand","role":"assistant","tool_calls":[]}}],"created":1778129978,"model":"qwen3-vl-8b-ov-int4","object":"chat.completion","usage":{"prompt_tokens":33,"completion_tokens":128,"total_tokens":161}}
```

## 五、将 OVMS 作为 OpenClaw 的后端启动

### 使用 OVMS 支持多个模型

OVMS 支持通过配置文件使用多个模型。创建 `./models/config.json`

```json
{
  "model_config_list":[
    {
      "config":{
        "name":"qwen3-30b-a3b-ov-int4",
        "base_path":"/models/qwen3-30b-a3b-ov-int4",
        "target_device": "GPU",
      }
    },
    {
      "config":{
        "name":"qwen3-vl-8b-ov-int4",
        "base_path":"/models/qwen3-vl-8b-ov-int4",
        "target_device": "GPU",
      }
    }
  ]
}
```

然后启动OVMS。

```bash
docker run -d --name ovms --user $(id -u):$(id -g) --device /dev/dri --group-add=$(stat -c "%g" /dev/dri/render* | head -n 1) --rm -p 8000:8000 -v $(pwd)/models:/models:rw openvino/model_server:latest-gpu --config_path /models/config.json --rest_port 8000
```

启动服务后，可以看到配置的多个模型。

```bash
curl http://localhost:8000/v3/models | jq .
```

命令输出类似如下。

```json
{
  "data": [
    {
      "id": "qwen3-30b-a3b-ov-int4",
      "object": "model",
      "created": 1778139971,
      "owned_by": "OVMS"
    },
    {
      "id": "qwen3-vl-8b-ov-int4",
      "object": "model",
      "created": 1778139971,
      "owned_by": "OVMS"
    }
  ],
  "object": "list"
}
```

需要注意的是每个模型目录下都需要有 `graph.pbtxt` 文件。这个文件会在 OVMS 启动单独模型的时候生成。

```bash
cd models
fd pbtxt
qwen3-30b-a3b-ov-int4/graph.pbtxt
qwen3-vl-8b-ov-int4/graph.pbtxt
```

### 配置 OpenClaw

通过 OpenClaw TUI 配置 OVMS 服务，确认 baseUrl 为 "http://localhost:8000/v3"；或者直接修改openclaw.json，将 OVMS 加入 providers。

```json
"ovms": {
  "api": "openai-completions",
  "apiKey": "ovms",
  "baseUrl": "http://localhost:8000/v3",
  "models": [
    {
      "contextWindow": 128000,
      "cost": {
        "cacheRead": 0,
        "cacheWrite": 0,
        "input": 0,
        "output": 0
      },
      "id": "qwen3-30b-a3b-ov-int4",
      "input": ["text"],
      "maxTokens": 8192,
      "name": "qwen3-30b-a3b-ov-int4",
      "reasoning": true
    }
    {
      ...,
    }
  ]
}

```