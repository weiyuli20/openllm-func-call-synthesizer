# 🛠️ openllm-func-call-synthesizer

![PyPI version](https://img.shields.io/pypi/v/openllm-func-call-synthesizer.svg)
[![Documentation Status](https://readthedocs.org/projects/openllm-func-call-synthesizer/badge/?version=latest)](https://openllm-func-call-synthesizer.readthedocs.io/en/latest/?version=latest)

> Lightweight toolkit to synthesize function-call datasets and convert them to formats compatible with OpenAI-style function-call training and downstream tooling (including Llama Factory compatible exports).

---

## ✨ Features

- 📝 Generate synthetic function call datasets for LLM training and evaluation
- 💬 Generate synthetic multi-turn conversation datasets
- ⚙️ Flexible configuration via YAML and Hydra
- 💻 CLI interface powered by Typer & Rich
- 🔧 Utility functions for dataset manipulation
- 🔄 Extensible and easy to integrate into your own pipeline
- 🌐 Supports multiple LLM backends (OpenAI, Google, etc.)
- 📊 Export formats: JSONL, CSV, Parquet, LlamaFactory-compatible

---

## 🛠 Installation

### Prerequisites

- Python 3.12+ (match environment used by the project)
- API credentials for any LLM backend (set via environment variables or `.env` file)
  - Example: `OPENAI_API_KEY`
  - See `.env.example` for reference

- 🔌 MCP Server (Required)

	This project relies on an MCP server to provide tool/function metadata.

	Before running the synthesizer, you must start an MCP server.

	▶ Start the example MCP server

	An example MCP server is included in the repository:

	python examples/mcp_example_sserver/server.py

	This will start a local MCP server that the synthesizer can connect to.

	Make sure your configuration (e.g. mcp_servers.transport) matches the server address.

	⸻

	⚠ Important
	* The synthesizer will fail if no MCP server is available.
	* Ensure the server is running before executing:

python -m apps.main

	* If you see connection errors, verify:
	* The server is running
	* The transport URL in your config is correct
	* Network/firewall settings allow local connections

⸻
---

### Install from PyPI

```bash
pip install openllm-func-call-synthesizer
# or using uv
uv add openllm-func-call-synthesizer
```

Install from source
```bash
git clone https://github.com/diqiuzhuanzhuan/openllm-func-call-synthesizer.git
cd openllm-func-call-synthesizer
uv sync
```
Is there no tool named 'uv'? You can install it with just one command:
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

⸻

⚡ Quickstart

Run the synthesizer with default config:
```bash
python -m apps.main
```

Enable only query generation:
```bash
python -m apps.main synthesizer.query_generation.enable=True
```

Enable function-call generation with custom name:
```bash
python -m apps.main synthesizer.function_call_generation.enable=True synthesizer.function_call_generation.name=function_call_gpt_4o
```

Enable conversation dataset generation:
```bash
python -m apps.main synthesizer.conversation_generation.enable=True
```

Override languages dynamically:

```bash
python -m apps.main synthesizer.query_generation.languages=[English,Spanish]
```

⸻

📂 Outputs
* Generated datasets are written to data/<name>/
* Each run produces:
* train.jsonl
* output.csv
* output.parquet
* llama_factory step creates LlamaFactory-compatible train.jsonl

⸻

🧪 Testing

Run the test suite:
```bash
pytest -q
```


⸻

📝 Configuration Highlights

Configuration file: examples/conf/synthesizer/default.yaml
* mcp_servers — MCP server(s) to query for available tools
* choose_part_tools — filter toolset to a subset
* query_generation — generate seed queries from function docs
* conversation_generation — generate multi-turn user/assistant conversations from scenarios
* function_call_generation — generate function-call pairs from queries
* critic — optional scoring/critique step
* llama_factory — export to LlamaFactory-compatible dataset
* verl - export to verl-compatible dataset

See docs for full field descriptions.

### Default pipeline walk-through

The provided `examples/conf/synthesizer/default.yaml` wires every stage together:

- **MCP bootstrap**: points to a local `ugreen_mcp` server on `http://localhost:8000/mcp`; leave it running before launching the synth job or queries will fail.
- **Tool filtering**: `choose_part_tools: false` keeps the full toolset; set it to a list (e.g. `["search_photos"]`) to restrict generations to specific tools.
- **Query generation**: reads `examples/function_docs.json`, emits multilingual prompts (English/Chinese/Japanese/German) under `data/function_query` via parallel OpenAI + Google model pools, each with generous TPM throttles for high-throughput runs.
- **Function-call synthesis**: consumes the query dataset, calls `gpt-4o` through the OpenAI backend, and writes `data/function_call_gpt_4o/*.jsonl` (set `max_num` to limit volume or switch `output_format`).
- **Critic pass**: re-scores every call with the configured `critic.backend_type`. The default config now uses `deepeval` with a Hydra-managed judge model, expects `query`, `prompt`, `function_call`, `functions`, `answer` fields, and emits a scored dataset named `function_call_gpt_4o_critiqued_by_deepeval`.
- **Downstream exports**: both `llama_factory` and `verl` blocks draw from the critic output, keep only rows with `score >= 0.8`, and materialize ready-to-train JSONL files plus optional train/val splits.

Feel free to copy the default file, tweak model lists or directories, and pass it via `python -m apps.main synthesizer=@your_config.yaml` for customized runs. For custom configurations, please refer to `example/conf/synthesizer/default.yaml`.
⸻

🐚 Parallel Runner

Helper script: bin/run_pipeline.sh
* Launch multiple synthesizer runs in parallel
* Requires .venv virtual environment
* Example usage:

```bash
chmod +x bin/run_pipeline.sh
bin/run_pipeline.sh default other
```
* Logs are printed to console; returns non-zero if any run fails
* Can also run manually using:

```bash
python -m apps.main synthesizer=default &
python -m apps.main synthesizer=other &
wait
```

⸻

## Contributing

Welcome to contribute！Please refer to [CONTRIBUTING.md](CONTRIBUTING.md) for details.

## License

MIT License. See [LICENSE](LICENSE) for details.

## Links

- [Documentation](https://openllm-func-call-synthesizer.readthedocs.io)
- [PyPI](https://pypi.org/project/openllm-func-call-synthesizer/)
- [GitHub](https://github.com/diqiuzhuanzhuan/openllm-func-call-synthesizer)

⸻

🌟 Star History

[![Star History Chart](https://api.star-history.com/svg?repos=diqiuzhuanzhuan/openllm-func-call-synthesizer&type=Date)](https://www.star-history.com/#diqiuzhuanzhuan/openllm-func-call-synthesizer&Date)


'''
仅生成问题
python -m apps.main ^
  synthesizer.query_generation.enable=true ^
  synthesizer.conversation_generation.enable=false ^
  synthesizer.tool_calling_conversation_generation.enable=false ^
  synthesizer.function_call_generation.enable=false ^
  synthesizer.critic.enable=false ^
  synthesizer.llama_factory.enable=false ^
  synthesizer.verl.enable=false


python -m apps.main \
  synthesizer.query_generation.enable=true \
  synthesizer.conversation_generation.enable=false \
  synthesizer.tool_calling_conversation_generation.enable=false \
  synthesizer.function_call_generation.enable=false \
  synthesizer.critic.enable=false \
  synthesizer.llama_factory.enable=false \
  synthesizer.verl.enable=false

仅生成function_call
python -m apps.main ^
  synthesizer.query_generation.enable=fasle ^
  synthesizer.conversation_generation.enable=false ^
  synthesizer.tool_calling_conversation_generation.enable=false ^
  synthesizer.function_call_generation.enable=true ^
  synthesizer.critic.enable=false ^
  synthesizer.llama_factory.enable=false ^
  synthesizer.verl.enable=false


python -m apps.main \
  synthesizer.query_generation.enable=fasle \
  synthesizer.conversation_generation.enable=false \
  synthesizer.tool_calling_conversation_generation.enable=false \
  synthesizer.function_call_generation.enable=true \
  synthesizer.critic.enable=false \
  synthesizer.llama_factory.enable=false \
  synthesizer.verl.enable=false

进行critic
python -m apps.main ^
  synthesizer.query_generation.enable=fasle ^
  synthesizer.conversation_generation.enable=false ^
  synthesizer.tool_calling_conversation_generation.enable=false ^
  synthesizer.function_call_generation.enable=false ^
  synthesizer.critic.enable=true ^
  synthesizer.llama_factory.enable=false ^
  synthesizer.verl.enable=false


python -m apps.main \
  synthesizer.query_generation.enable=fasle \
  synthesizer.conversation_generation.enable=false \
  synthesizer.tool_calling_conversation_generation.enable=false \
  synthesizer.function_call_generation.enable=false \
  synthesizer.critic.enable=true \
  synthesizer.llama_factory.enable=false \
  synthesizer.verl.enable=false
'''


step1:

启动mcp服务


step2:

将mcp服务地址配置到default.yml


step3:运行代码合成query

```
python -m apps.main \
  synthesizer.query_generation.enable=true \
  synthesizer.conversation_generation.enable=false \
  synthesizer.tool_calling_conversation_generation.enable=false \
  synthesizer.function_call_generation.enable=false \
  synthesizer.critic.enable=false \
  synthesizer.llama_factory.enable=false \
  synthesizer.verl.enable=false

```

生成的每一个query:主要字段是query和function(当前工具说明)
```
{"query":"新手小白请教：怎么查‘市区’这个供电单位的10kV线损概览？比如达标率这些，今天的数据就行～","dimension":"User Persona","language":"Chinese","function":"{\n  \"type\": \"function\",\n  \"function\": {\n    \"name\": \"getXljc\",\n    \"description\": \"获取指定地区10KV分线线损监测的概览信息(达标率、高损数、经济运行率)\\nArgs:\\n    org_name: 组织机构名称，地市，区县以及具体供电所名称，如：咸阳市、高新区、礼泉县、兴平市、杨凌、市区、城东中心供电所、古渡供电所等，\\n              示例：【全省、陕西、陕西省】---> 【陕西省】；【陕西省咸阳市】---> 【咸阳市】；【陕西省咸阳市高新区】--->【咸阳市高新区】\\n    start_time: 开始时间(格式YYYY-MM-DD)，当用户问当天，或不指定日期时，默认值为当前日期，当用户按月查询信息时，该参数默认为开始月份的第一天，如‘11月’，则为 当前年份的‘11-01’\\n    end_time: 结束时间(格式YYYY-MM-DD)，当用户问当天，或不指定日期时，默认值为‘当前日期+1’的日期，当用户按月查询信息时，该参数默认为结束月份的第一天（当不指定结束月份时，默认为：开始月份+1），如‘11月’，则为：当前年份的‘12-01’。\\n    tjlx: str类型，统计类型（例如：2-按日，4-按月），默认值为\\\"2\\\"，表示按日统计，当用户按月查询时，统计类型应为月，如：2025年12月\\nReturns:\\n    调用api接口查询到的指定地区10KV分线线损监测的概览信息结果字典\",\n    \"parameters\": {\n      \"additionalProperties\": false,\n      \"properties\": {\n        \"org_name\": {\n          \"type\": \"string\"\n        },\n        \"start_time\": {\n          \"type\": \"string\"\n        },\n        \"end_time\": {\n          \"type\": \"string\"\n        },\n        \"tjlx\": {\n          \"default\": \"2\",\n          \"type\": \"string\"\n        }\n      },\n      \"required\": [\n        \"org_name\",\n        \"start_time\",\n        \"end_time\"\n      ],\n      \"type\": \"object\"\n    }\n  }\n}","function_hash":"128ba2d46ff5bbab","provider":"qwen","model":"qwen-plus"}

```

step4:基于强模型生成function_call调用（只能生成一次工具调用）

```
python -m apps.main \
  synthesizer.query_generation.enable=fasle \
  synthesizer.conversation_generation.enable=false \
  synthesizer.tool_calling_conversation_generation.enable=false \
  synthesizer.function_call_generation.enable=true \
  synthesizer.critic.enable=false \
  synthesizer.llama_factory.enable=false \
  synthesizer.verl.enable=false

```
输出结果：相比query数据增加了functions, prompt,raw_output, function_call, answer字段

```
{"query":"咦，这个‘线损治理达人’在哪看啊？能帮我捞一下咸阳市的数据吗？时间从今年元旦开始。","dimension":"Linguistic","language":"Chinese","function":"{\n  \"type\": \"function\",\n  \"function\": {\n    \"name\": \"get_xszldr_info\",\n    \"description\": \"获取线损治理达人信息详情\\nArgs:\\n    org_name: 组织机构名称，地市，区县以及具体供电所名称，如：咸阳市、高新区、礼泉县、兴平市、杨凌、市区、城东中心供电所、古渡供电所等，\\n              示例：【全省、陕西、陕西省】---> 【陕西省】；【陕西省咸阳市】---> 【咸阳市】；【陕西省咸阳市高新区】--->【咸阳市高新区】\\n    start_time: 开始时间(格式YYYY-MM-DD), 如果问题中有明确的年月日信息就取该年月日信息 比如2025年10月1日就取2025-10-01  如果问到当年信息时,该参数应取当年的1月1日 例如如果当年是2024年该参数就取2024年01月01日 如果问题中没有时间信息时 应取当年的日期\\nReturns:\\n    调用api接口查询到的线损治理达人结果字典\",\n    \"parameters\": {\n      \"additionalProperties\": false,\n      \"properties\": {\n        \"org_name\": {\n          \"type\": \"string\"\n        },\n        \"start_time\": {\n          \"type\": \"string\"\n        }\n      },\n      \"required\": [\n        \"org_name\",\n        \"start_time\"\n      ],\n      \"type\": \"object\"\n    }\n  }\n}","function_hash":"a210c546daa4a720","provider":"qwen","model":"qwen-plus","functions":"[{\"type\": \"function\", \"function\": {\"name\": \"get_xszldr_info\", \"description\": \"获取线损治理达人信息详情\\nArgs:\\n    org_name: 组织机构名称，地市，区县以及具体供电所名称，如：咸阳市、高新区、礼泉县、兴平市、杨凌、市区、城东中心供电所、古渡供电所等，\\n              示例：【全省、陕西、陕西省】---> 【陕西省】；【陕西省咸阳市】---> 【咸阳市】；【陕西省咸阳市高新区】--->【咸阳市高新区】\\n    start_time: 开始时间(格式YYYY-MM-DD), 如果问题中有明确的年月日信息就取该年月日信息 比如2025年10月1日就取2025-10-01  如果问到当年信息时,该参数应取当年的1月1日 例如如果当年是2024年该参数就取2024年01月01日 如果问题中没有时间信息时 应取当年的日期\\nReturns:\\n    调用api接口查询到的线损治理达人结果字典\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\"}, \"start_time\": {\"type\": \"string\"}}, \"required\": [\"org_name\", \"start_time\"], \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"get_sqbq\", \"description\": \"获取十强市 \\\\\/ 百强县 \\\\\/ 百强所 当月、累计、覆盖率信息\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\", \"description\": \"组织机构名称，地市，区县以及具体供电所名称，如：咸阳市、高新区、礼泉县、兴平市、杨凌、市区、城东中心供电所、古渡供电所等，\\n      示例：【全省、陕西、陕西省】---> 【陕西省】；【陕西省咸阳市】---> 【咸阳市】；【陕西省咸阳市高新区】--->【咸阳市高新区】\"}, \"start_time\": {\"type\": \"string\", \"description\": \"开始时间（YYYY-MM-DD）如果问题中有明确的年月日信息就取该年月日信息 比如2025年10月1日就取2025-10-01 当问到需要2024年信息时该参数应取2024年01月01日 以此类推 如果问到当年信息时,该参数应取当年的1月1日 例如如果当年是2024年该参数就取2024年01月01日 如果问题中没有时间信息时 应取当年的日期\"}}, \"required\": [\"org_name\", \"start_time\"], \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"getXljc\", \"description\": \"获取指定地区10KV分线线损监测的概览信息(达标率、高损数、经济运行率)\\nArgs:\\n    org_name: 组织机构名称，地市，区县以及具体供电所名称，如：咸阳市、高新区、礼泉县、兴平市、杨凌、市区、城东中心供电所、古渡供电所等，\\n              示例：【全省、陕西、陕西省】---> 【陕西省】；【陕西省咸阳市】---> 【咸阳市】；【陕西省咸阳市高新区】--->【咸阳市高新区】\\n    start_time: 开始时间(格式YYYY-MM-DD)，当用户问当天，或不指定日期时，默认值为当前日期，当用户按月查询信息时，该参数默认为开始月份的第一天，如‘11月’，则为 当前年份的‘11-01’\\n    end_time: 结束时间(格式YYYY-MM-DD)，当用户问当天，或不指定日期时，默认值为‘当前日期+1’的日期，当用户按月查询信息时，该参数默认为结束月份的第一天（当不指定结束月份时，默认为：开始月份+1），如‘11月’，则为：当前年份的‘12-01’。\\n    tjlx: str类型，统计类型（例如：2-按日，4-按月），默认值为\\\"2\\\"，表示按日统计，当用户按月查询时，统计类型应为月，如：2025年12月\\nReturns:\\n    调用api接口查询到的指定地区10KV分线线损监测的概览信息结果字典\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\"}, \"start_time\": {\"type\": \"string\"}, \"end_time\": {\"type\": \"string\"}, \"tjlx\": {\"default\": \"2\", \"type\": \"string\"}}, \"required\": [\"org_name\", \"start_time\", \"end_time\"], \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"getXljcPenetrate\", \"description\": \"获取指定区域的子区域的10KV分线线损监测详细信息(子区域的达标率、高损数、经济运行率等)\\nArgs:\\n    org_name: 组织机构名称，地市，区县以及具体供电所名称，如：咸阳市、高新区、礼泉县、兴平市、杨凌、市区、城东中心供电所、古渡供电所等，\\n              示例：【全省、陕西、陕西省】---> 【陕西省】；【陕西省咸阳市】---> 【咸阳市】；【陕西省咸阳市高新区】--->【咸阳市高新区】\\n    start_time: 开始时间(格式YYYY-MM-DD)，当用户问当天，或不指定日期时，默认值为当前日期，当用户按月查询信息时，该参数默认为开始月份的第一天，如‘11月’，则为 当前年份的‘11-01’\\n    end_time: 结束时间(格式YYYY-MM-DD)，当用户问当天，或不指定日期时，默认值为‘当前日期+1’的日期，当用户按月查询信息时，该参数默认为结束月份的第一天（当不指定结束月份时，默认为：开始月份+1），如‘11月’，则为：当前年份的‘12-01’。\\n    tjlx: str类型，统计类型（例如：2-按日，4-按月），默认值为\\\"2\\\"，表示按日统计，当用户按月查询时，统计类型应为月，如：2025年12月\\nReturns:\\n    调用api接口查询到的指定区域的所有子区域的10KV分线线损监测详细信息结果字典\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\"}, \"start_time\": {\"type\": \"string\"}, \"end_time\": {\"type\": \"string\"}, \"tjlx\": {\"default\": \"2\", \"type\": \"string\"}}, \"required\": [\"org_name\", \"start_time\", \"end_time\"], \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"getTqjc\", \"description\": \"获取指定区域的台区线损监测详细信息(台区的达标率、高损数、经济运行率等)\\nArgs:\\n    org_name: 组织机构名称，地市，区县以及具体供电所名称，如：咸阳市、高新区、礼泉县、兴平市、杨凌、市区、城东中心供电所、古渡供电所等，\\n              示例：【全省、陕西、陕西省】---> 【陕西省】；【陕西省咸阳市】---> 【咸阳市】；【陕西省咸阳市高新区】--->【咸阳市高新区】\\n    start_time: 开始时间(格式YYYY-MM-DD)，当用户问当天，或不指定日期时，默认值为当前日期，当用户按月查询信息时，该参数默认为开始月份的第一天，如‘11月’，则为 当前年份的‘11-01’\\n    end_time: 结束时间(格式YYYY-MM-DD)，当用户问当天，或不指定日期时，默认值为‘当前日期+1’的日期，当用户按月查询信息时，该参数默认为结束月份的第一天（当不指定结束月份时，默认为：开始月份+1），如‘11月’，则为：当前年份的‘12-01’。\\n    tjlx: str类型，统计类型（例如：2-按日，4-按月），默认值为\\\"2\\\"，表示按日统计，当用户按月查询时，统计类型应为月，如：2025年12月\\nReturns:\\n    调用api接口查询到的指定区域的台区线损监测概览信息结果字典\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\"}, \"start_time\": {\"type\": \"string\"}, \"end_time\": {\"type\": \"string\"}, \"tjlx\": {\"default\": \"2\", \"type\": \"string\"}}, \"required\": [\"org_name\", \"start_time\", \"end_time\"], \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"getTqjcPenetrate\", \"description\": \"获取指定区域的子区域的台区线损监测详细信息(子区域的达标率、高损数、经济运行率等)\\nArgs:\\n    org_name: 组织机构名称，地市，区县以及具体供电所名称，如：咸阳市、高新区、礼泉县、兴平市、杨凌、市区、城东中心供电所、古渡供电所等，\\n              示例：【全省、陕西、陕西省】---> 【陕西省】；【陕西省咸阳市】---> 【咸阳市】；【陕西省咸阳市高新区】--->【咸阳市高新区】\\n    start_time: 开始时间(格式YYYY-MM-DD)，当用户问当天，或不指定日期时，默认值为当前日期，当用户按月查询信息时，该参数默认为开始月份的第一天，如‘11月’，则为 当前年份的‘11-01’\\n    end_time: 结束时间(格式YYYY-MM-DD)，当用户问当天，或不指定日期时，默认值为‘当前日期+1’的日期，当用户按月查询信息时，该参数默认为结束月份的第一天（当不指定结束月份时，默认为：开始月份+1），如‘11月’，则为：当前年份的‘12-01’。\\n    tjlx: str类型，统计类型（例如：2-按日，4-按月），默认值为\\\"2\\\"，表示按日统计，当用户按月查询时，统计类型应为月，如：2025年12月\\nReturns:\\n    调用api接口查询到的指定区域的子区域的台区线损监测详细信息结果字典\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\"}, \"start_time\": {\"type\": \"string\"}, \"end_time\": {\"type\": \"string\"}, \"tjlx\": {\"default\": \"2\", \"type\": \"string\"}}, \"required\": [\"org_name\", \"start_time\", \"end_time\"], \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"getJczlbh\", \"description\": \"监测治理闭环查询(包含线路如期核实率,线路如期治理率,台区如期核实率和台区如期治理率)\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\", \"description\": \"组织机构名称，地市，区县以及具体供电所名称，如：咸阳市、高新区、礼泉县、兴平市、杨凌、市区、城东中心供电所、古渡供电所等，\\n      示例：【全省、陕西、陕西省】---> 【陕西省】；【陕西省咸阳市】---> 【咸阳市】；【陕西省咸阳市高新区】--->【咸阳市高新区】\"}, \"start_time\": {\"type\": \"string\", \"description\": \"开始时间（YYYY-MM-DD）\"}, \"end_time\": {\"type\": \"string\", \"description\": \"结束时间（YYYY-MM-DD），需注意：如果查询的是具体的某一天，结束时间应晚于开始时间一天，例如：start_time若为2025-01-01，end_time应为2025-01-02\"}, \"tjlx\": {\"default\": \"2\", \"type\": \"string\", \"description\": \"统计类型（2代表按日，4代表按月）如果查询的是具体的某一天,或者时间段,则该参数取2,如果查询的是某个整月的信息,比如11月,12月,11月1日到12月1日之间,则该参数取4\"}}, \"required\": [\"org_name\", \"start_time\", \"end_time\"], \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"getJczlbhPenetrate\", \"description\": \"监测治理闭环穿透查询(包含线路如期核实率,线路如期治理率,台区如期核实率和台区如期治理率)\\n\\n说明（穿透层级）：\\n- 使用“省号\/省公司”查询：通常返回省公司 + 各市公司等信息\\n- 使用“市号\/市公司”查询：通常返回区县公司信息\\n- 使用“县号\/区县公司”查询：通常返回供电所信息\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\", \"description\": \"组织机构名称，地市，区县以及具体供电所名称，如：咸阳市、高新区、礼泉县、兴平市、杨凌、市区、城东中心供电所、古渡供电所等，\\n      示例：【全省、陕西、陕西省】---> 【陕西省】；【陕西省咸阳市】---> 【咸阳市】；【陕西省咸阳市高新区】--->【咸阳市高新区】\"}, \"start_time\": {\"type\": \"string\", \"description\": \"开始时间（YYYY-MM-DD）\"}, \"end_time\": {\"type\": \"string\", \"description\": \"结束时间（YYYY-MM-DD）\"}, \"tjlx\": {\"default\": \"2\", \"type\": \"string\", \"description\": \"统计类型（2代表按日，4代表按月）如果查询的是具体的某一天,或者时间段,则该参数取2,如果查询的是某个整月的信息,比如11月,12月,11月1日到12月1日之间,则该参数取4\"}}, \"required\": [\"org_name\", \"start_time\", \"end_time\"], \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"getFqxs\", \"description\": \"分区线损查询（返回供电量\/售电量\/线损电量\/线损率等）\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\", \"description\": \"组织机构名称，地市，区县以及具体供电所名称，如：咸阳市、高新区、礼泉县、兴平市、杨凌、市区、城东中心供电所、古渡供电所等，\\n      示例：【全省、陕西、陕西省】---> 【陕西省】；【陕西省咸阳市】---> 【咸阳市】；【陕西省咸阳市高新区】--->【咸阳市高新区】\"}, \"start_time\": {\"type\": \"string\", \"description\": \"开始时间（YYYY-MM-DD）\"}, \"end_time\": {\"type\": \"string\", \"description\": \"结束时间（YYYY-MM-DD）\"}}, \"required\": [\"org_name\", \"start_time\", \"end_time\"], \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"getFyxs\", \"description\": \"获取分压线损查询结果\\n\\n使用说明：\\n- 当需要查询“分压线损 \/ 不同电压等级线损率”相关数据时，使用本函数\\n- 返回结果按单位 + 电压等级维度给出线损率指标\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\", \"description\": \"组织机构名称，地市，区县以及具体供电所名称，如：咸阳市、高新区、礼泉县、兴平市、杨凌、市区、城东中心供电所、古渡供电所等，\\n      示例：【全省、陕西、陕西省】---> 【陕西省】；【陕西省咸阳市】---> 【咸阳市】；【陕西省咸阳市高新区】--->【咸阳市高新区】\"}, \"start_time\": {\"type\": \"string\", \"description\": \"开始时间（YYYY-MM-DD）\"}, \"end_time\": {\"type\": \"string\", \"description\": \"结束时间（YYYY-MM-DD），\"}}, \"required\": [\"org_name\", \"start_time\", \"end_time\"], \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"getLlxs\", \"description\": \"查询指定单位每月理论线损合理率（主网合理率、线路合理率、台区合理率），一次查询一个单位。\\ntjlx 固定为 2。合理率一般在当月 20 日左右统计，start_time\/end_time 须覆盖该日（通常为月初至次月初）。\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\", \"description\": \"组织机构名称，如：咸阳市、高新区、礼泉县、兴平市、杨凌、市区等\"}, \"start_time\": {\"type\": \"string\", \"description\": \"查询月第一天（YYYY-MM-DD），如 2025-11-01\"}, \"end_time\": {\"type\": \"string\", \"description\": \"次月第一天（YYYY-MM-DD），如 2025-12-01\"}}, \"required\": [\"org_name\", \"start_time\", \"end_time\"], \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"getLlxsPenetrate\", \"description\": \"查询咸阳各区县理论线损合理率穿透结果（主网\/线路\/台区合理率），一次查询多个区县。\\ntjlx 固定为 2。省\/市层通常含三项合理率，县层多为线路+台区，供电所层多为台区合理率。\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\", \"description\": \"组织机构名称（省级查询常用 61403 对应咸阳公司）\"}, \"start_time\": {\"type\": \"string\", \"description\": \"查询月第一天（YYYY-MM-DD）\"}, \"end_time\": {\"type\": \"string\", \"description\": \"次月第一天（YYYY-MM-DD）\"}}, \"required\": [\"org_name\", \"start_time\", \"end_time\"], \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"getSbzb\", \"description\": \"查询指定单位每月设备指标：线路赋值率、台区赋值率。一次查询一个单位，tjlx 固定为 2。\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\", \"description\": \"组织机构名称，如：咸阳市、高新区、礼泉县、兴平市、杨凌、市区等\"}, \"start_time\": {\"type\": \"string\", \"description\": \"开始时间（YYYY-MM-DD），查询月第一天\"}, \"end_time\": {\"type\": \"string\", \"description\": \"结束时间（YYYY-MM-DD），次月第一天\"}}, \"required\": [\"org_name\", \"start_time\", \"end_time\"], \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"getSbzbPenetrate\", \"description\": \"查询设备指标穿透结果（线路赋值率、台区赋值率）。一次查询多个下级单位，tjlx 固定为 2。\\n\\n穿透层级：省公司查询返回省+市；市公司返回区县；区县返回供电所。\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\", \"description\": \"组织机构名称，如：咸阳市、高新区、礼泉县、兴平市、杨凌、市区等\"}, \"start_time\": {\"type\": \"string\", \"description\": \"开始时间（YYYY-MM-DD）\"}, \"end_time\": {\"type\": \"string\", \"description\": \"结束时间（YYYY-MM-DD）\"}}, \"required\": [\"org_name\", \"start_time\", \"end_time\"], \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"getFxxssjCountDetail\", \"description\": \"获取指定具体区域的10kV分线线损的详细信息, 包括母线异常，采集异常以及线变关系异动三种情况，并且得到对应设备的详细的信息。比如查看线路具体是三种异常的哪一种，还可以返回线路关口采集成功率，公变采集成功率，专变采集成功率等。\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\"}, \"start_time\": {\"type\": \"string\"}, \"end_time\": {\"type\": \"string\"}, \"abnormal_type\": {\"default\": \"\", \"type\": \"string\", \"description\": \"异常种类。支持中文（采集异常\/母线异常\/线变关系异动）、\\n代码（cjyc\/mpyc\/xbgxyd），或留空\/「全部」查询三类合并结果。\"}, \"tjlx\": {\"default\": \"2\", \"type\": \"string\"}}, \"required\": [\"org_name\", \"start_time\", \"end_time\"], \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"getTqxssjCountDetail\", \"description\": \"获取指定具体区域的台区线损的详细信息, 包括三相不平衡异常，功率因数异常，失压失流，电流越限以及采集异常五种情况，并且得到对应设备的详细的信息\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\"}, \"start_time\": {\"type\": \"string\"}, \"end_time\": {\"type\": \"string\"}, \"abnormal_type\": {\"default\": \"\", \"type\": \"string\", \"description\": \"异常种类。支持中文（采集异常\/电流越限\/功率因数异常\/三相不平衡异常\/失压失流）、\\n代码（cjyc\/dlyx\/glysyc\/sxbphyc\/sysl），或留空\/「全部」查询五类合并结果。\"}, \"tjlx\": {\"default\": \"2\", \"type\": \"string\"}}, \"required\": [\"org_name\", \"start_time\", \"end_time\"], \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"get_Fxxssj_List\", \"description\": \"获取指定具体区域的10kV分线高损统计数据(包括线路名称，线损类型，线损率，线损电量，输入电量，输出电量等),重点是可以列出线路名称\\nArgs:\\n    org_name: 组织机构名称，省，地市，区县以及具体供电所名称，如：咸阳市、高新区、礼泉县、兴平市、杨凌、市区等\\n    start_time: 开始时间(格式2025-01-01), 当用户问某一天时,该参数为当天，当用户按月查询时，start_time为该月的第一天，如：2025-11-01\\n    end_time: 结束时间(格式2025-01-01)，当用户问某一天时，end_time为start_time+1天，当用户按月查询时，end_time为start_time的下一个月的第一天，如：2025-12-01\\n    tjlx: 统计的时间单位类型，2-按天统计，4-按月统计\\n\\nReturns:\\n    调用api接口查询到的指定区域的设备信息，并且包含其所属公司与四种电力相关指标。\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\"}, \"start_time\": {\"type\": \"string\"}, \"end_time\": {\"type\": \"string\"}, \"tjlx\": {\"default\": \"2\", \"type\": \"string\"}}, \"required\": [\"org_name\", \"start_time\", \"end_time\"], \"type\": \"object\"}}},  {\"type\": \"function\", \"function\": {\"name\": \"queryMeteoFcstData\", \"description\": \"查询起始时间-结束时间范围内的天气预报\/气温。\", \"parameters\": {\"additionalProperties\": false, \"properties\": {\"org_name\": {\"type\": \"string\", \"description\": \"组织机构或行政区名称，如咸阳市、礼泉县、泾阳县、高新区等。\\n      市级使用 areaType=2（如咸阳 610400），区县使用 areaType=3。\"}, \"start_time\": {\"type\": \"string\", \"description\": \"开始日期 YYYY-MM-DD\"}, \"end_time\": {\"anyOf\": [{\"type\": \"string\"}, {\"type\": \"null\"}], \"default\": null, \"description\": \"结束日期 YYYY-MM-DD\"}, \"time_type\": {\"default\": \"DAY\", \"type\": \"string\", \"description\": \"时间粒度，DAY 按日（默认）；也可传 HOUR 等接口支持的类型\"}, \"data_type\": {\"default\": \"1202\", \"type\": \"string\", \"description\": \"数据类型，1202 表示气温\/天气预报（默认）\"}}, \"required\": [\"org_name\", \"start_time\"], \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"get_current_time\", \"description\": \"Return the current local time as an ISO-like string.\", \"parameters\": {\"additionalProperties\": false, \"properties\": {}, \"type\": \"object\"}}}, {\"type\": \"function\", \"function\": {\"name\": \"get_current_date\", \"description\": \"Return the current local date.\", \"parameters\": {\"additionalProperties\": false, \"properties\": {}, \"type\": \"object\"}}}]","prompt":[{"content":"You are a helpful assistant.","role":"system"},{"content":"咦，这个‘线损治理达人’在哪看啊？能帮我捞一下咸阳市的数据吗？时间从今年元旦开始。","role":"user"}],"raw_output":{"content":"我来帮您查询咸阳市的线损治理达人信息。首先，我需要获取当前日期来确定今年的具体年份。\n\n","role":"assistant","tool_calls":[{"function":{"arguments":"{}","name":"get_current_date"},"id":"call_b01ab97f69c74e5ea2f10e","index":0,"type":"function"}]},"function_call":"[{\"name\": \"get_current_date\", \"arguments\": {}}]","answer":"{\n  \"role\": \"assistant\",\n  \"content\": \"我来帮您查询咸阳市的线损治理达人信息。首先，我需要获取当前日期来确定今年的具体年份。\\n\\n\",\n  \"tool_calls\": [\n    {\n      \"id\": \"call_b01ab97f69c74e5ea2f10e\",\n      \"type\": \"function\",\n      \"function\": {\n        \"name\": \"get_current_date\",\n        \"arguments\": \"{}\"\n      },\n      \"index\": 0\n    }\n  ]\n}"}


```

step5:基于强模型对生成的function数据做llm judge,质量过滤
```
python -m apps.main \
  synthesizer.query_generation.enable=fasle \
  synthesizer.conversation_generation.enable=false \
  synthesizer.tool_calling_conversation_generation.enable=false \
  synthesizer.function_call_generation.enable=false \
  synthesizer.critic.enable=true \
  synthesizer.llama_factory.enable=false \
  synthesizer.verl.enable=false

```
测试工具是否选择正确，参数是否整理，工具是否是json格式


step5.1: 基于tool_call_conversation_generation生成多轮工具调用数据
```
python -m apps.main \
  synthesizer.query_generation.enable=fasle \
  synthesizer.conversation_generation.enable=false \
  synthesizer.tool_calling_conversation_generation.enable=true \
  synthesizer.function_call_generation.enable=false \
  synthesizer.critic.enable=false \
  synthesizer.llama_factory.enable=false \
  synthesizer.verl.enable=false

```



step6:编写测试脚本，测试base模型的效果

测评维度：json合法性、工具正确性，functioncall格式正确性，答案正确性（侧面反映了具体参数的正确性）

卡点：

1.需要模型在大批样本上快速推理，因此需要推理优化，加速
2.需要tool_calling结果（需要数据沙盒模拟接口数据）



