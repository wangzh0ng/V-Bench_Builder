# V-Bench Builder 2.0 开发设计文档

| 项目 | 内容 |
| --- | --- |
| 文档名称 | V-Bench Builder 2.0 开发设计文档 |
| 版本 | V1.0 |
| 日期 | 2026-07-29 |
| 依据 | 《V-Bench Builder 2.0 详细系统设计文档 V2.0 final》(需求设计) |
| 定位 | 实现级设计,供开发直接动手;补齐需求文档未展开的实现细节 |

> 本文档不重复需求文档已写清的"做什么",只写"怎么建"。章节编号独立。

---

## 1. 模块划分与代码组织

单体后端(Python),单进程,目录组织:

```
vbench/
├── api/                  # FastAPI 路由与 DTO
│   ├── cases.py          # POST /cases, GET /cases/{id}, GET /cases
│   ├── tasks.py          # GET /tasks/{id}
│   ├── benchmarks.py     # GET /benchmarks?tags, GET /benchmark/{id}/export
│   └── auth.py           # Bearer token 鉴权
├── core/
│   ├── reproduce.py      # 13.2 AI 解析与复现(调用 llm/ 模块)
│   ├── callgraph.py      # tree-sitter Call Graph 构建(§6)
│   ├── expand.py         # 13.3 向上调用链 + 13.4 相似度扩展
│   ├── workspace.py      # 13.5 最小集合并剪枝 + 目录保全
│   ├── difficulty.py     # 13.6 难度评分
│   └── qa.py             # 4 项 QA 检查(§10)
├── llm/
│   ├── client.py         # LLM HTTP 客户端(读 config)
│   ├── prompts/          # prompt 模板(可热加载)
│   └── context.py        # 大仓库上下文策略(§5.3)
├── pkg/
│   ├── decompile.py      # jadx 调用(APK/JAR,§7)
│   ├── detect_lang.py    # 语言检测
│   ├── extract.py        # zip 解压(安全)
│   └── scrub.py          # 敏感物脱敏(§7.4)
├── store/
│   ├── db.py             # SQLite 连接+WAL+迁移
│   ├── minio.py          # MinIO 客户端
│   ├── models.py         # ORM/数据类
│   └── paths.py          # MinIO 路径规约
├── worker/
│   ├── queue.py          # t_task 轮询 + claim
│   └── runner.py         # 任务分派(状态机驱动,§8)
├── ui/                   # 前端 SPA(React+Vite,§16)
├── config.py             # 配置加载(§3)
└── main.py               # FastAPI + worker 线程启动
```

---

## 2. 技术栈锁定

| 项 | 选型 |
| --- | --- |
| 语言 | Python 3.11+ |
| Web 框架 | FastAPI + uvicorn |
| DB | SQLite 3.45+(WAL, JSON1) |
| 对象存储 | MinIO (Python SDK) |
| Parser | tree-sitter + tree-sitter-languages(四语言 grammar) |
| LLM | 配置驱动(§3);HTTP 调用,structured output via JSON mode |
| 反编译 | jadx(APK 的 DEX + JAR 的 JVM bytecode 都吃) |
| 鉴权 | Bearer token(`SECOCTO_API_TOKEN`) |
| 前端 | React + Vite + TypeScript;Monaco Editor(代码浏览器);FastAPI 静态托管 |
| 部署 | K8s 单副本 `secflow-ns`/`secflow-api` |

**不引入**:Redis、Neo4j、PostgreSQL、Celery、Worker 池、沙箱。

---

## 3. 配置设计

配置文件 `config.yaml`(路径由 `VBENCH_CONFIG` 环境变量指定,默认 `./config.yaml`),启动加载,不热加载(改后重启)。

```yaml
server:
  host: 0.0.0.0
  port: 8080
  auth_token: ${SECOCTO_API_TOKEN}     # 鉴权 token,从 env 注入

db:
  path: /data/vbench.sqlite            # SQLite 文件路径
  wal: true

minio:
  endpoint: http://secflow-minio:9000
  access_key: ${MINIO_ACCESS_KEY}
  secret_key: ${MINIO_SECRET_KEY}
  bucket: vbench

llm:
  endpoint: ${LLM_ENDPOINT}            # 必填
  api_key: ${LLM_API_KEY}
  model: ${LLM_MODEL}                   # 如 chimera_llm/auto
  max_tokens: 8192
  timeout_sec: 120
  max_retries: 3
  max_concurrency: 4                   # §8 并发限流
  prompts_dir: ./llm/prompts

worker:
  concurrency: 2                       # worker 线程数
  poll_interval_sec: 2
  task_timeout_sec: 1800
  max_attempts: 3

decompile:
  jadx_path: /usr/bin/jadx           # APK(DEX)与 JAR(JVM bytecode)统一用 jadx
  timeout_sec: 600

qa:
  obfuscation_short_name_len: 3        # §10 混淆检测阈值
  obfuscation_min_density: 0.4
```

**校验**:启动时 `config.py` 校验必填项(endpoint/api_key/bucket),缺失即启动失败并报错。

---

## 4. 数据库详细设计

### 4.1 建表 DDL(SQLite)

```sql
-- 1. 漏洞案例表
CREATE TABLE t_vulnerability_case (
  id            TEXT PRIMARY KEY,
  project_name  TEXT,
  cve_id        TEXT,
  raw_description TEXT,
  package_type  TEXT,                  -- SOURCE_ZIP / APK / JAR
  language      TEXT,                  -- Java / Python / Go / C++
  platform      TEXT,                  -- web / android / harmony-next / harmony-sa
  original_path TEXT,                  -- MinIO source/{id}/original.<ext>
  source_path   TEXT,                  -- MinIO source/{id}/extracted|decompiled/
  structured_data JSON,                -- §9.3 schema
  status        TEXT,                  -- §4.3 状态机
  created_at    TEXT
);
CREATE INDEX idx_case_status ON t_vulnerability_case(status);
CREATE INDEX idx_case_cve ON t_vulnerability_case(cve_id);

-- 2. Benchmark 表
CREATE TABLE t_benchmark (
  id          TEXT PRIMARY KEY,        -- VB001
  case_id     TEXT,
  workspace_path TEXT,
  metadata_path   TEXT,
  task_path       TEXT,
  ground_truth_path TEXT,
  package_path    TEXT,                -- benchmark/{id}/benchmark.zip (长期有效)
  difficulty   INTEGER,
  tags         JSON,                    -- §9.2
  qa_status    TEXT,                   -- PENDING / PASSED / FAILED
  created_at   TEXT
);
CREATE INDEX idx_bench_qa ON t_benchmark(qa_status);
-- tags facet 查询靠 JSON1,数据量小不额外建表达式索引

-- 3. 任务表(兼队列)
CREATE TABLE t_task (
  id          TEXT PRIMARY KEY,
  case_id     TEXT,
  task_type   TEXT,                    -- REPRODUCE / BUILD_WORKSPACE / QA / EXPORT
  status      TEXT,                    -- PENDING / RUNNING / SUCCESS / FAILED
  locked_by   TEXT,                    -- worker 线程 id(并发 claim)
  claimed_at  TEXT,
  attempts    INTEGER DEFAULT 0,
  result      JSON,
  error       TEXT,
  created_at  TEXT,
  updated_at  TEXT
);
CREATE INDEX idx_task_claim ON t_task(status, locked_by);

-- 4. QA 结果表
CREATE TABLE t_qa_result (
  id           TEXT PRIMARY KEY,
  benchmark_id TEXT,
  check_name   TEXT,                   -- SCHEMA / EXISTENCE / PARSABLE / LINE_INTEGRITY
  passed       INTEGER,
  detail       TEXT,
  created_at   TEXT
);
CREATE INDEX idx_qa_bench ON t_qa_result(benchmark_id);
```

### 4.2 迁移

启动时 `db.py` 执行 `migrations/` 下顺序 SQL 文件(如 `001_init.sql`),用 `PRAGMA user_version` 记录已应用版本。无第三方迁移框架,纯 SQL 文件。

### 4.3 Case 状态机

```
POST /cases:
  解压/反编译成功 ──> CREATED (source_path 已设)
  解压失败       ──> EXTRACT_FAILED (人工)
  反编译失败     ──> DECOMPILE_FAILED (人工)

CREATED ──reproduce()──> REPRODUCING
   REPRODUCING ──Sink 定位成功(存在性预检通过)──> LOCALIZED
   REPRODUCING ──存在性预检不过/无候选──> LOCALIZATION_FAILED (人工)
   LOCALIZED ──build_workspace()(zip)──> BUILDING
   BUILDING ──剪枝成功──> BUILT
   LOCALIZED (APK/JAR) ──跳过剪枝直接──> BUILT
   BUILT ──qa()──> QAING
   QAING ──QA PASS──> BUILT_PASSED ──export()──> EXPORTING ──打包完成──> EXPORTED
   QAING ──QA FAIL 不可自愈──> QA_FAILED (人工)
```

> **QA 在 EXPORT 之前**:QA 4 项查 workspace + ground_truth,通过后再打包;失败可回 BUILDING 重做 workspace,不必重打 zip。
>
> 状态枚举写入 `t_vulnerability_case.status`:`CREATED / REPRODUCING / LOCALIZED / BUILDING / BUILT / QAING / BUILT_PASSED / EXPORTING / EXPORTED / EXTRACT_FAILED / DECOMPILE_FAILED / LOCALIZATION_FAILED / QA_FAILED`。

---

## 5. LLM 集成设计

### 5.1 函数接口定义

三个 LLM 函数签名与输入输出 schema:

```python
def llm_extract_intent(raw_desc: str) -> dict:
    """输入: 漏洞描述文本
    输出: {"cwe": "CWE-79", "vuln_type": "Stored XSS",
           "keywords": ["parseXml","upload","avatar"], 
           "trigger": "未对上传文件名/内容做转义"}"""

def llm_backward_trace_to_entry(repo_index: "RepoIndex", sink: dict, intent: dict) -> list | None:
    """输入: 仓库函数索引(§5.3)、候选 sink、intent
    输出: 调用链 [{file,function,line}, ...] 从 Entry 到 sink; None=推不出"""

def ai_verify_reproducibility(path: list, sink: dict, intent: dict) -> dict:
    """输入: 调用链、sink、intent
    输出: {"triggerable": bool, "reason": "..."}  # 仅审计,不作门"""
```

### 5.2 结构化输出

所有 LLM 调用用 **JSON mode**(若模型支持)或 prompt 末尾要求 `只输出 JSON, schema: {...}`,解析失败则重试(计入 `max_retries`)。解析用 `pydantic` 校验 schema,不合法重试。

### 5.3 大仓库上下文策略(核心)

源码 >> 上下文窗口,不能整包塞给 LLM。分阶段:

```
1. tree-sitter 解析全包 → 建 RepoIndex:
   - 全部 Function 节点 {file, name, start_line, end_line, package, class}
   - 全部 Call 边(§6,按名匹配)
2. llm_extract_intent(描述) → keywords + trigger
3. 关键词检索 RepoIndex → 召回候选 sink 函数(按名/路径模糊匹配 keywords)
4. 对每个候选 sink:
   a. 取 sink 函数体 + 调用链上下游 caller 函数体(由 RepoIndex Call Graph 定位,§6)
   b. 拼成 prompt:"给定 sink 函数 X 与其上游 caller,反推完整调用链 Entry→Sink"
   c. llm_backward_trace_to_entry 输出链
5. 存在性预检(§5.4):链与 sink 的文件/函数在源码真实存在 → 采信;否则换候选
```

**关键**:LLM 只读"相关函数片段",不读整包。RepoIndex 是 tree-sitter 产物的结构化索引,不是 LLM 上下文。

> **RepoIndex 存储与生命周期**:函数索引写 SQLite **case 级临时表**(`t_repo_func_{case_id}`,字段 `id/file/name/start_line/end_line/package/class/ast_blob`),Call 边写 `t_repo_call_{case_id}(caller_id,callee_id)`——避免大仓库(几万函数)全内存驻留 OOM。内存只留"当前要喂给 LLM 的函数片段"与查询结果集。临时表在 Case 进入 `LOCALIZED` 后(或失败终态后)随 worker 清理 DROP,不持久化、不入业务库。

### 5.4 存在性预检

```python
def existence_check(sink: dict, path: list, repo_index) -> bool:
    files = {sink["file"]} | {step["file"] for step in path}
    return all(f in repo_index.files for f in files) and \
           sink["function"] in repo_index.functions_by_file[sink["file"]]
```

### 5.5 编排策略

- 超时:`timeout_sec`;重试:`max_retries`(指数退避)
- 并发:`asyncio.Semaphore(max_concurrency)` 限流
- 降级:不设降级模型链;重试耗尽即 `mark_localization_failed()` 转人工

---

## 6. tree-sitter Call Graph 构建

### 6.1 边定义(近似,不做符号解算)

**规则**:函数 A 调用 B,当且仅当 A 函数体的 AST 含一个 call 表达式,其被调名与 B 的函数名**字符串相等**,且 B 与 A **同文件**或**同 package**(由 RepoIndex 的 package 字段判)。

- 不解析 import、不解析虚函数分发、不解析函数指针
- 重名:同 package 内取第一个匹配(近似可接受,ground truth 由 AI 兜底)
- 跨 package:Java/Go 按包名匹配,Python 按模块名,C++ 按命名空间

### 6.2 实现

```python
def build_call_graph(repo_index: "RepoIndex") -> "CallGraph":
    edges = {}  # caller_id -> [callee_id]
    for caller in repo_index.all_functions:
        calls = extract_call_names(caller.ast)  # tree-sitter query, 裸名
        for name in calls:
            callee = repo_index.find_in_scope(name, caller)  # 同文件/同 package
            if callee:
                edges.setdefault(caller.id, []).append(callee.id)
    return CallGraph(edges, repo_index)
```

### 6.3 tree-sitter binding

`tree-sitter-languages`(预打包 grammar,免编译)。四语言 call 表达式的 AST query 各一份,放 `core/callgraph.py`。

---

## 7. 源码包处理

### 7.1 语言检测

按文件扩展名统计:`.java`/`.py`/`.go`/`.cpp+.h`,取最多者。多语言项目取主语言。

### 7.2 解压安全

zip 解压前:
- 校验总解压大小 < 1GB(防 zip bomb)
- 校验每个 entry 路径无 `..` 与绝对路径(防穿越)
- 用 `zipfile` 内置,不 shell 调 `unzip`

### 7.3 反编译

| 包类型 | 工具 | 调用 |
| --- | --- | --- |
| APK | jadx | `jadx -d {out} {apk}`,超时 `decompile.timeout_sec` |
| JAR | jadx | `jadx -d {out} {jar}` |

> jadx 同时支持 DEX(APK)与 JVM bytecode(JAR)反编译,统一一个工具,不引入 CFR。失败(非零退出/超时)→ 标记 `DECOMPILE_FAILED` → 人工。

### 7.4 敏感物脱敏 denylist

打包进 `workspace/` 前剔除(7.1 规则的具体清单):

```
**/.env, **/.env.*, **/*.key, **/*.pem, **/*.p12,
**/.git/**, **/.svn/**,
**/opencode.jsonc, **/.claude/**, **/.secocto/**,
**/docker-compose*.yml, **/Dockerfile,
**/application*.yml, **/application*.properties  # Spring 配置(含凭据)
```

实现:`scrub.py` 打包前用 denylist glob 过滤。

---

## 8. 任务编排与并发

### 8.1 Worker 模型

单进程内 `worker/concurrency` 个线程,共享一个 SQLite `t_task` 表队列。claim 用事务:

```sql
-- 原子 claim(单实例多线程也要防重复领取)
BEGIN IMMEDIATE;
UPDATE t_task SET locked_by=:wid, claimed_at=now(), status='RUNNING', attempts=attempts+1
  WHERE id=(SELECT id FROM t_task WHERE status='PENDING' ORDER BY created_at LIMIT 1);
SELECT id FROM t_task WHERE locked_by=:wid AND status='RUNNING';
COMMIT;
```

完成后改 `SUCCESS`/`FAILED`;超时由监控线程扫 `claimed_at` 超时行重置 `PENDING`(达 `max_attempts` 则改 `FAILED`,不再重置)。

### 8.2 子任务串联(状态机驱动)

每个 Case 状态变化触发下一子任务:
- `CREATED → REPRODUCING`:写入 `REPRODUCE` 任务
- `LOCALIZED → BUILDING`(zip):写 `BUILD_WORKSPACE` 任务;`LOCALIZED → BUILT`(APK/JAR):跳过 BUILD_WORKSPACE,直接进 QA
- `BUILT → QAING`:写 `QA` 任务;`BUILT_PASSED → EXPORTING`:写 `EXPORT`(打包)任务
- QA FAIL → 回 `BUILDING` 重做 workspace(可自愈);不可自愈 → `QA_FAILED` 人工
- 子任务完成回调更新 Case 状态,触发下一任务

> `task_type` 取值:`EXTRACT` / `DECOMPILE` / `REPRODUCE` / `BUILD_WORKSPACE` / `QA` / `EXPORT`。

### 8.3 幂等

- 同 Case 重复 `reproduce` 请求:若已有 `RUNNING`/`SUCCESS` 的 REPRODUCE 任务,拒绝重复(返回 409)
- 失败重试:同一任务 `attempts` 达 `max_attempts` 不再重置,Case 标 FAILED

### 8.4 资源限制与并发模型

- worker 并发:`worker.concurrency`(默认 2)
- LLM 并发:`llm.max_concurrency`(默认 4)
- 大仓库解析内存:tree-sitter 逐文件解析,不整包 AST 缓存;函数索引写 SQLite 临时表(§5.3),内存只留索引

> **线程-asyncio 混用模型**:FastAPI 主进程跑 uvicorn event loop 接 API;`worker/concurrency` 个**线程**跑同步重活(SQLite 读写、tree-sitter 解析、jadx subprocess)——这些是同步阻塞调用,放线程避免卡 event loop。LLM 调用是 HTTP I/O,在线程内用 `asyncio.run()` 起临时 event loop + `asyncio.Semaphore(max_concurrency)` 限流。API 端点里访问 SQLite 统一用 `starlette.concurrency.run_in_threadpool` 包,避免阻塞主 event loop。

### 8.5 `t_task.result` 内容(按 task_type)

| task_type | result JSON | 说明 |
| --- | --- | --- |
| `EXTRACT`/`DECOMPILE` | `{"source_path":"source/{id}/extracted|decompiled/","language":"Java"}` | 写入 `t_vulnerability_case.source_path/language` |
| `REPRODUCE` | `{"structured_data_ref":"case/{id}/structured_data"}` | structured_data 已写回案例行,result 仅指明完成 |
| `BUILD_WORKSPACE` | `{"workspace_path":"workspace/{id}/","file_count":N}` | zip 临时 workspace 路径 |
| `QA` | `{"checks":[{name,passed,detail},...]}` | 4 项明细,同步写 `t_qa_result` |
| `EXPORT` | `{"package_path":"benchmark/{id}/benchmark.zip","benchmark_id":"VB001"}` | 写入 `t_benchmark` 行 + tags(此时 difficulty 已算出) |

---

## 9. 数据交付物 schema(完整)

### 9.1 metadata.json

```json
{
  "benchmark_id": "VB001",
  "case_id": "case-xxx",
  "cve_id": "CVE-2024-xxxx",
  "vulnerability_type": "CWE-79",
  "vuln_type": "Stored XSS",
  "language": "Java",
  "platform": "web",
  "package_type": "SOURCE_ZIP",
  "difficulty": 72,
  "vuln_related_files": [".../FileUploadUtils.java", ".../SysProfileController.java"],
  "extra_evidence_files": [],
  "source_package_paths": [],
  "trace_files": [],
  "created_at": "2026-07-29T00:00:00Z"
}
```

### 9.2 t_benchmark.tags(JSON)

```json
{"cwe":"CWE-79","vuln_type":"Stored XSS","language":"Java","platform":"web","package_type":"SOURCE_ZIP"}
```

> tags 在 EXPORT 任务打包时(§9.7 第 3 步)随 `t_benchmark` 行写入——此时 `difficulty` 已算出、`cwe/vuln_type` 已从 structured_data 镜像确定。

**MinIO 路径规约**(`store/paths.py` 实现):

| 对象 | 路径 |
| --- | --- |
| 原始上传包 | `source/{case_id}/original.<ext>` |
| zip 解压源码 | `source/{case_id}/extracted/` |
| APK/JAR 反编译整包 | `source/{case_id}/decompiled/` |
| zip 最小集 workspace | `workspace/{case_id}/` |
| Benchmark 产物 | `benchmark/{benchmark_id}/`(含 `metadata.json`/`task.json`/`ground_truth.json`/`workspace/`) |
| 最终交付 | `benchmark/{benchmark_id}/benchmark.zip`(唯一长期保留) |

### 9.3 structured_data(ground truth 来源,存 t_vulnerability_case.structured_data)

```json
{
  "cwe": "CWE-79",
  "vuln_type": "Stored XSS",
  "vulnerable_file": "FileUploadUtils.java",
  "vulnerable_function": "upload",
  "sink_point": "FileUploadUtils.java:88",
  "call_chain": ["SysProfileController.avatar", "FileUploadUtils.upload"],
  "entry_point": "SysProfileController.avatar",
  "method": "AI_TRACE",
  "audit": {"triggerable": true, "reason": "..."}
}
```

> 补齐需求文档 4.6 缺的 `cwe` / `vuln_type`——由 `llm_extract_intent` 产出,写入 structured_data,再镜像到 metadata.json 与 tags。
>
> **表示转换约定**:LLM 内部输出对象数组 `[{function,file,line}, ...]`(供 §5.4 存在性预检逐项校验);落库到 `structured_data.call_chain` 与交付 `ground_truth.call_chain` 时转为字符串数组 `["Class.function", ...]`。sink 同理:LLM 输出 `{file,function,line}` 对象;`structured_data.sink_point` 存 `"file:line"` 字符串(需求文档 4.6 约定),`ground_truth.sink_location` 存对象。转换在 `reproduce.py` 写回时统一完成。

### 9.4 ground_truth.json

```json
{
  "vulnerability_type": "CWE-79",
  "sink_location": {"file":"...","function":"upload","line":88},
  "entry_location": {"file":"...","function":"avatar"},
  "call_chain": ["SysProfileController.avatar", "FileUploadUtils.upload"]
}
```

### 9.5 task.json

```json
{
  "task": "在 workspace 中定位漏洞的 Sink 点、入口与完整调用链",
  "workspace": "workspace/",
  "platform": "web",
  "package_type": "SOURCE_ZIP",
  "expected_output": {
    "sink_location": {"file":"...","function":"...","line":0},
    "entry_location": {"file":"...","function":"..."},
    "call_chain": ["..."]
  }
}
```

> 评分由参考验证 agent(`vuln-hunt-validate`)做,V-Bench 不出 grader。task.json 只给任务与期望输出 schema。

### 9.6 难度各量口径(对应需求文档 13.6 公式)

$$\text{Difficulty} = \min(100,\ \max(20,\ 5\log_{10}(\text{Tokens}+1) + 4\cdot\text{ChainDepth} + 3\cdot\text{SimDistractions} + \text{Base}))$$

| 量 | 取值口径 |
| --- | --- |
| `Tokens` | tiktoken `cl100k_base` 统计 workspace 内全部源码文件 token 数之和 |
| `ChainDepth` | `call_chain` 元素数 − 1(Entry 到 Sink 的边数) |
| `SimDistractions` | 13.4 相似度扩展纳入的**文件**数(非函数数),即 `final_minimal_set` 中属于 `S_sim` 的文件数 |
| `Base` | 按项目主语言(§7.1 语言检测取最多者):C/C++ 10、Java/Go 5、Python 3 |
| 上下界 | 钳制到 [20, 100] |

APK/JAR 场景:`SimDistractions=0`(不裁剪无干扰)、`ChainDepth` 取复现所得调用链层数、`Tokens` 为反编译整包源码 token 数。

### 9.7 打包流程(EXPORT 任务内)

```
1. build/确认 workspace 目录(workspace/{id}/ 或 source/{id}/decompiled/)
2. scrub 脱敏(§7.4 denylist 过滤)
3. 写 metadata.json / task.json / ground_truth.json 到 benchmark/{id}/
   (metadata 的 difficulty 由 §9.6 此刻算出,tags 此刻写 t_benchmark)
4. zip 打包为 benchmark/{id}/benchmark.zip
5. 写 t_benchmark 行(package_path 长期有效,其余路径为中间过程件)
6. 按保留期策略,中间过程件(source/{id}/*、workspace/{id}/、未打包散文件)可由外部调度清理
```

---

## 10. QA 实现细节

| 检查 | 实现 |
| --- | --- |
| **Schema** | pydantic 校验 metadata/ground_truth/task JSON;并扫 workspace 不含 denylist 文件(§7.4) |
| **Existence** | 建 workspace 文件集索引;ground_truth 的 file/function/line 逐一查:文件存在、函数在 tree-sitter AST 中、line 在函数范围内 |
| **Parsable** | tree-sitter 逐文件 parse,统计 error 节点;全文件无 error 即过(悬空引用不影响) |
| **Line Integrity** | 混淆检测:统计函数体 call 表达式被调名长度;短名(≤`obfuscation_short_name_len`)占比 > `obfuscation_min_density` → 判定混淆,`sink_location.line` 置 null |

```python
def is_obfuscated(func_ast) -> bool:
    calls = extract_call_names(func_ast)
    if not calls: return False
    short = sum(1 for n in calls if len(n) <= cfg.obfuscation_short_name_len)
    return short / len(calls) > cfg.obfuscation_min_density
```

---

## 11. API 接口定义

### 11.1 鉴权

除健康检查外,所有端点 `Authorization: Bearer {SECOCTO_API_TOKEN}`,不符 401。

### 11.2 端点

| 方法 | 路径 | 请求 | 响应 |
| --- | --- | --- | --- |
| POST | `/api/v1/cases` | multipart: `raw_description`(form)、`source_package`(file)、`package_type`、`platform`(form) | `{case_id, status:"CREATED"}` (解压/反编译同步,失败 422) |
| GET | `/api/v1/cases?status=&page=&size=` | query | `{items:[{case_id,status,cve_id,language,platform}], total}` |
| GET | `/api/v1/cases/{id}` | — | `{case_id, status, structured_data?, error?}` |
| POST | `/api/v1/cases/{id}/reproduce` | — | `{task_id}` (异步) |
| POST | `/api/v1/cases/{id}/build-minimal-workspace` | `{max_similar_files?, max_tokens?}` | `{task_id}` |
| GET | `/api/v1/tasks/{id}` | — | `{task_id, status, result?, error?}` |
| GET | `/api/v1/benchmarks?language=&cwe=&platform=&page=&size=` | query | `{items:[{id,tags,difficulty,qa_status}], total}` |
| GET | `/api/v1/benchmark/{id}/export` | — | `benchmark.zip` (stream) |
| GET | `/api/v1/benchmarks/export?language=&cwe=&platform=&ids=` | query | `bundle.zip` (内含多个 `benchmark.zip` + `manifest.json` 列表,按条件批量导出) |
| PATCH | `/api/v1/cases/{id}/structured_data` | `{sink_location, entry_location, call_chain}` | `{case_id, status:"LOCALIZED"}` (人工覆写 ground truth,后端做存在性校验,通过则续跑流水线) |
| GET | `/api/v1/cases/{id}/files` | — | `{items:[{path,size}]}` (source_path 文件树,供代码浏览器) |
| GET | `/api/v1/cases/{id}/files/{path}` | — | `text/plain` 源码内容 |
| GET | `/api/v1/cases/{id}/functions?file=` | — | `{items:[{function,start_line,end_line}]}` (RepoIndex 函数定位,供点选) |
| GET | `/api/v1/metrics/automation` | — | `{auto_rate, pending_human, failed, by_status:{...}}` (自动化总览) |

### 11.3 错误码

| HTTP | 含义 |
| --- | --- |
| 400 | 请求参数非法/包类型不支持 |
| 401 | token 无效 |
| 404 | case/benchmark 不存在 |
| 409 | 重复触发(同案 RUNNING 中) |
| 422 | source_package 解压/反编译失败 |
| 500 | 内部错误(写 t_task.error) |

---

## 12. 可观测性实现

- **日志**:structlog,JSON 行,字段 `{ts, level, trace_id, case_id, task_id, msg, ...}`;trace_id 由 API 入口生成贯穿子任务
- **Metrics**:Prometheus 文本暴露 `/metrics`;指标 `vbench_task_total{type,status}` / `vbench_task_failed{type}` / `vbench_benchmark_generated` / `vbench_llm_latency_seconds`
- **告警**:Prometheus alertmanager;`Worker 失败率>10%`、`DB 异常`、`任务积压>50`、`QA 失败率升高`

---

## 13. 测试与验收

### 13.1 黄金用例(fixture)

| 用例 | 语言 | 平台 | 包类型 | 用途 | 源码来源 |
| --- | --- | --- | --- | --- | --- |
| RuoYi 头像上传 XSS | Java | web | SOURCE_ZIP | 端到端基线 | 参考 zip 已提供(6 文件) |
| 公开 SQL 注入 CVE(候选) | Java | web | SOURCE_ZIP | 双维扩展验证 | 公开 CVE 仓库,立项时定具体编号 |
| 公开 XXE CVE(候选) | Python | web | SOURCE_ZIP | 调用链深度 | 公开 CVE 仓库,立项时定 |
| 公开 Android 组件 CVE(候选) | Java | android | APK | 反编译整包场景 | 公开 APK,立项时定 |

> 每个用例需**人工标注 ground truth**(Sink/Entry/Chain),作为流水线产出的对照答案。RuoYi 用例标注从参考 zip 的 `vuln_related_files` + 人工确认;其余 3 个立项时按"有公开源码 + Sink/Chain 可人工定位"标准选定,先跑通 web 的 RuoYi + SQL 注入两条,APK 与 Python XXE 视进度。

断言:ground_truth 的 sink/entry/chain 与人工标注一致;workspace 通过 QA 4 项;`benchmark.zip` 可解压且不含 denylist 文件。

### 13.2 测试计划

- 单元:callgraph/expand/workspace/difficulty/qa 各模块函数级
- 集成:黄金用例端到端
- 回归:每次发版跑全套黄金用例

---

## 14. 开发任务拆解(WBS)

| # | 任务 | 产出 | 依赖 |
| --- | --- | --- | --- |
| 1 | 脚手架 + config + db migration | 启动可连 SQLite/MinIO | — |
| 2 | API 骨架 + 鉴权 + DTO | 端点可调 | 1 |
| 3 | 源码包处理(解压/反编译/语言检测/脱敏) | pkg/ 模块 | 1 |
| 4 | tree-sitter RepoIndex + Call Graph | core/callgraph.py | 3 |
| 5 | LLM client + 大仓库上下文策略 | llm/ 模块 | 4 |
| 6 | 13.2 reproduce + 存在性预检 | core/reproduce.py | 5 |
| 7 | 13.3/13.4 expand | core/expand.py | 4 |
| 8 | 13.5 workspace 剪枝 | core/workspace.py | 7 |
| 9 | 13.6 difficulty | core/difficulty.py | 8 |
| 10 | QA 4 项 | core/qa.py | 8 |
| 11 | Worker 队列 + 状态机 + 幂等 | worker/ 模块 | 6,8 |
| 12 | Benchmark 打包 + 导出 | api/benchmarks.py | 11 |
| 13 | 前端 UI(总览/详情/介入/导出 + 代码浏览器) | ui/ SPA | 2,12 |
| 14 | 人工介入后端(PATCH structured_data + 文件/函数读取 + 存在性校验续跑) | api/cases.py 扩展 | 4,11 |
| 15 | 自动化总览指标 + 批量导出 bundle | api/metrics.py + benchmarks/export | 12 |
| 16 | 黄金用例 + 测试 | tests/ | 全部 |

---

## 15. 待确认决策(沿用需求文档,列出供追溯)

| # | 决策 | 取值 |
| --- | --- | --- |
| 1 | 难度评分 | 公式出预估分;攒 AI 作答数据后按通过率回归校准 |
| 2 | platform 范围 | 枚举占位 web/android/harmony-next/harmony-sa;先跑通 web |
| 3 | API 鉴权 | Bearer `SECOCTO_API_TOKEN` |
| 4 | 评分方 | 参考验证 agent,V-Bench 不出 grader |
| 5 | task.json | 极简任务 + workspace 路径 + platform + 期望输出 schema |
| 6 | CVSS | 不进 metadata;由验证 agent 打 |
| 7 | 黄金用例 | RuoYi XSS + 3 个公开 CVE |
| 8 | Call Graph | 按函数名匹配建边,不做符号解算 |
| 9 | Case 状态机 | 见 §4.3 |
| 10 | 反编译器 | jadx(APK 的 DEX + JAR 的 JVM bytecode 统一一个工具) |

---

## 16. 前端 UI 与人工介入

### 16.1 技术栈

- React + Vite + TypeScript SPA;FastAPI 静态托管 build 产物(`GET /ui/*` 返回 `index.html` + 静态资源)
- 代码浏览器:Monaco Editor(read-only),行点击高亮;函数定位用 RepoIndex 的 `{function,start_line,end_line}` 在 Monaco 上叠加徽标
- 鉴权:UI 登录存 Bearer token(localStorage);同一 `SECOCTO_API_TOKEN`
- 后端复用 §11 API,前端不直接访问 SQLite/MinIO

### 16.2 页面

| 页面 | 路由 | 功能 |
| --- | --- | --- |
| **自动化总览** | `/` | 统计卡片(自动产出率 / 待人工数 / 失败数)+ Case 列表(状态/语言/平台/CVE/进度),按状态筛选,点行进详情 |
| **Case 详情** | `/cases/:id` | 原始描述、structured_data、任务进度时间线(各 task_type 状态与时间)、当前 Case 状态、错误信息 |
| **人工介入** | `/cases/:id/intervene` | 代码浏览器(左 source 文件树、右 Monaco 代码 + 函数徽标);点选函数标 Sink/Entry/chain,提交 PATCH |
| **Benchmark 导出** | `/benchmarks` | 按 tags(`language`/`cwe`/`platform`)筛选,勾选多个,批量导出 `bundle.zip` |

### 16.3 人工介入流程

1. Case 进入 `LOCALIZATION_FAILED` / `QA_FAILED` → 总览页"待人工"队列显示
2. 人工打开介入页,浏览器加载 `source_path`(解压 zip 源码 或 反编译整包源码)
3. 左侧文件树(`GET /files`)→ 右侧代码 + 函数徽标(`GET /functions?file=`)
4. 点选 Sink 函数 → 标 `sink_location{file,function,line}`;点选 Entry 函数 → 标 `entry_location`;必要时点选 chain 中间函数 → 组 `call_chain`
5. 提交 `PATCH /cases/{id}/structured_data`
6. 后端存在性校验通过 → `status` 回 `LOCALIZED` → 自动触发 `BUILD_WORKSPACE` 续跑;校验不过 → 返回 422,UI 标红提示重选

> 人工只覆写"代码级定位",不改 `cve_id`/`raw_description`/漏洞事实;`method` 字段标 `HUMAN_OVERRIDE` 供审计。

### 16.4 自动化总览指标(`/api/v1/metrics/automation`)

```json
{
  "auto_rate": 0.78,           // EXPORTED / (EXPORTED + FAILED + pending_human)
  "pending_human": 3,           // LOCALIZATION_FAILED + QA_FAILED
  "failed": 1,
  "by_status": {"CREATED":2,"REPRODUCING":1,"EXPORTED":35,"LOCALIZATION_FAILED":3,"QA_FAILED":0},
  "avg_case_duration_sec": 142
}
```

### 16.5 批量导出(`GET /benchmarks/export`)

按 `language` / `cwe` / `platform` / `ids` 任一组合过滤,把选中的多个 `benchmark.zip` 打包进一个 `bundle.zip`,附 `manifest.json` 列出每个 benchmark 的 `id`/`tags`/`difficulty`:

```
bundle.zip
├── manifest.json          # [{benchmark_id, tags, difficulty, file:"VB001/benchmark.zip"}, ...]
├── VB001/benchmark.zip
├── VB002/benchmark.zip
└── ...
```

流式打包,不在内存全量缓冲;选中数过多时限制单次上限(默认 50)。

### 16.6 目录新增

```
ui/                         # 前端 SPA(Vite)
├── src/
│   ├── pages/              # Dashboard / CaseDetail / Intervene / Export
│   ├── components/CodeBrowser/
│   └── api/                # 调后端 REST
└── vite.config.ts
api/static.py               # FastAPI 静态托管 ui/dist
```



---

> 本文档与需求文档的关系:需求文档定义"做什么、为什么";本文档定义"怎么建"。实现以本文档为准,口径变更同步回需求文档。
