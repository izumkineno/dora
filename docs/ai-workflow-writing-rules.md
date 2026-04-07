# AI 专用的 Dora 工作流撰写规则

这份文档不是在解释 Dora 是什么，而是在约束 AI 应该怎样生成、审核和修订 Dora 工作流。

如果你还没有看过基础教程，先读 [`create-workflow-tutorial.md`](./create-workflow-tutorial.md)。这份规则默认你已经理解了三个基本事实：

- 工作流是一张蓝图，核心文件通常是 `dataflow.yml`
- 连接关系写在 YAML 里，不写在节点代码里
- 一个节点只负责一个明确职责

## 1. 目标

AI 输出的结果必须同时满足两个要求：

- 对人可读：人类读者能快速看懂节点职责、连接关系和运行方式
- 对机器可改：后续 AI 可以在不重写整份文档的前提下继续扩展、审查和修订

AI 不应只输出一段 YAML，也不应只输出泛泛的文字说明。一次合格的输出，至少要包含：

1. 工作流目标
2. 节点设计
3. `dataflow.yml` 草稿
4. 运行步骤
5. 假设与待确认项

## 2. 核心规则

### 2.1 必须遵守

- 必须先拆节点职责，再写连接关系，再补 `build` / `path` / `env`
- 必须显式列出每个节点的输入和输出
- 必须使用稳定、可读、可搜索的节点名和输出名
- 必须区分“工作流连接”与“节点内部逻辑”
- 必须给出运行命令，而不是只给 YAML
- 必须说明当前采用了哪些默认假设

### 2.2 明确禁止

- 不要臆造仓库里不存在的 Dora 字段
- 不要把上下游连接规则偷写进节点代码描述里
- 不要省略 `build` 或运行前提，除非用户明确说已经存在
- 不要发明并不存在的包名、二进制名或示例目录
- 不要把一个节点写成“采集 + 推理 + 展示”三合一的黑盒

## 3. AI 的工作流程

AI 在撰写 Dora 工作流时，按下面的顺序执行。

### 3.1 第一步：明确目标

先把用户目标压缩成一句话：

`这个工作流要从哪里拿数据，经过哪些处理，最后输出到哪里。`

如果这句话都说不清，就不要急着写 YAML。

### 3.2 第二步：拆节点

节点拆分时，优先按职责边界而不是按代码语言边界来拆：

- source 节点：采集、读取、触发
- processor 节点：推理、过滤、转换
- sink 节点：展示、保存、发布

默认拆分原则：

- 一个节点一个主职责
- 一个输出一个稳定名字
- 下游节点只消费它需要的数据

### 3.3 第三步：定义连接

连接时先写数据来源，再写字段名：

- `camera/image`
- `detector/bbox`
- `source/data`

AI 必须逐项检查：

1. 下游输入名是否清晰
2. 上游输出名是否存在
3. 一个输入是否只接一个来源
4. 是否真的需要 timer 输入

### 3.4 第四步：补构建和运行信息

每个节点都应该至少回答这几个问题：

- 怎么安装
- 怎么启动
- 是否依赖 Python / Rust / C / C++
- 是否需要 `env`

如果是 Python 工作流，优先给出仓库现有示例风格的命令路径，例如：

```bash
dora build dataflow.yml --uv
dora run dataflow.yml --uv
```

如果不是 Python，也要明确说明不需要 `--uv` 的原因。

### 3.5 第五步：自检

输出前，AI 必须做一轮自检：

1. 有没有只写概念、没写 YAML
2. 有没有只写 YAML、没写运行命令
3. 有没有输入名和输出名对不上
4. 有没有把多个职责压进一个节点
5. 有没有遗漏依赖或环境变量
6. 有没有使用仓库中不存在的名称

## 4. 标准输出格式

AI 生成 Dora 工作流时，建议固定为下面的顺序：

### 4.1 目标

用 2 到 4 句话说明：

- 工作流解决什么问题
- 数据从哪里来
- 最终产物是什么

### 4.2 节点设计

使用表格或列表，至少写清：

- 节点名
- 节点职责
- 输入
- 输出
- 运行方式

### 4.3 YAML 草稿

直接给出完整 `dataflow.yml`，不要只给片段。

### 4.4 运行步骤

至少包含：

- 依赖安装
- `dora build`
- `dora run` 或 `dora start`

### 4.5 假设和待确认项

如果有信息缺失，必须用单独一节列出来，而不是隐藏在自然语言里。

## 5. 数据流细分结构示例与可选字段

AI 在写 Dora 工作流时，不只要会写最普通的 `nodes`，还要会判断当前场景应该使用哪种节点结构。

### 5.1 普通节点示例

这是最基础的节点写法，适合大多数 source / processor / sink 场景。参考 `examples/python-dataflow/dataflow.yml`：

```yaml
nodes:
  - id: sender
    path: sender.py
    outputs:
      - message

  - id: transformer
    path: transformer.py
    inputs:
      message: sender/message
    outputs:
      - transformed

  - id: receiver
    path: receiver.py
    inputs:
      message: sender/message
      transformed: transformer/transformed
```

这种结构下，AI 应优先关注：

- `id` 是否稳定
- `path` 是否能直接启动节点
- `inputs` 是否和上游输出严格对齐
- `outputs` 是否只暴露必要数据

### 5.2 单个算子节点示例

如果节点本身以 operator 形式组织，可以使用 `operator:`。参考 `examples/python-operator-dataflow/dataflow.yml`：

```yaml
nodes:
  - id: object_detection
    operator:
      build: pip install -r requirements.txt
      python: object_detection.py
      send_stdout_as: stdout
      inputs:
        image: webcam/image
      outputs:
        - bbox
        - stdout
```

AI 在看到这类结构时，应理解为：

- 这是一个节点，但节点主体由 `operator:` 描述
- `python:` 是 operator 的入口
- `send_stdout_as` 会把标准输出映射成一个可连接的输出
- `outputs` 不只是业务数据，也可能包含日志或文本通道

### 5.3 多算子运行时节点示例

如果一个运行时节点下挂多个 operator，应使用 `operators:` 列表。参考 `examples/multiple-daemons/dataflow.yml`：

```yaml
nodes:
  - id: runtime-node
    _unstable_deploy:
      machine: A
    operators:
      - id: rust-operator
        build: cargo build -p multiple-daemons-example-operator
        shared-library: ../../target/debug/multiple_daemons_example_operator
        inputs:
          tick: dora/timer/millis/100
          random: rust-node/random
        outputs:
          - status
```

AI 应把这类写法识别为：

- 顶层仍然是一个 node
- 真正的数据处理单元在 `operators:` 里
- `shared-library` 指向可加载的 operator 实现
- 这种结构通常出现在 Rust runtime / shared library 场景

### 5.4 动态节点示例

当某个节点不是由当前 `dora run` 自动启动，而是需要外部单独启动时，可能会看到 `path: dynamic`。参考 `examples/python-dataflow/dataflow_dynamic.yml`：

```yaml
nodes:
  - id: opencv-plot
    build: pip install "git+https://github.com/dora-rs/dora-hub.git#egg=opencv-plot&subdirectory=node-hub/opencv-plot"
    path: dynamic
    inputs:
      image: camera/image
```

AI 写到这类结构时，必须补一句说明：

- 这是动态节点
- 它可能需要单独启动
- `dora start` 往往比 `dora run` 更适合这类场景

### 5.5 常见可选字段

下面这些字段不是每个工作流都要有，但 AI 需要知道它们什么时候该出现：

| 字段 | 位置 | 作用 | 什么时候用 |
| --- | --- | --- | --- |
| `env` | node | 给节点注入环境变量 | 摄像头设备号、宽高、模型路径等运行参数 |
| `build` | node / operator | 安装依赖或构建产物 | Python 包安装、Cargo build、shared library 构建 |
| `path` | node | 节点入口 | 普通进程型节点 |
| `operator` | node | 单个 operator 描述 | 节点主体就是一个 operator |
| `operators` | node | 多个 operator 列表 | runtime 节点下挂多个 operator |
| `python` | operator | Python operator 入口 | `operator:` 结构下的 Python 实现 |
| `shared-library` | operator | 动态库路径 | Rust/C/C++ operator 以共享库方式加载 |
| `send_stdout_as` | operator | 将 stdout 变成一个输出通道 | 需要把文本输出串进下游节点 |
| `_unstable_deploy` | node | 指定部署机器 | 多 daemon / 分布式实验场景 |

### 5.6 AI 的结构选择规则

AI 在生成工作流时，优先按下面的顺序判断该选哪种结构：

1. 如果只是普通独立节点，优先用 `path`
2. 如果一个节点内部只有一个 operator，考虑用 `operator`
3. 如果一个 runtime 节点内部挂多个处理单元，使用 `operators`
4. 如果节点需要人工或外部系统单独启动，考虑 `path: dynamic`
5. 如果涉及跨机器部署，再补 `_unstable_deploy`

如果 AI 没法确定该选哪种结构，默认先产出最简单的 `path` 节点版本，并在待确认项中说明是否要改成 `operator` 或 `operators`

### 5.7 顶层字段字典

下面这些字段都来自 `examples/` 目录中的真实 `dataflow*.yml`。如果 AI 需要解释字段，优先引用这些例子，而不是凭空造说明。

| 字段 | 位置 | 作用 | 最小示例 | 对应 example |
| --- | --- | --- | --- | --- |
| `nodes` | 顶层 | 定义整个工作流中的节点列表 | `nodes:` | `examples/python-dataflow/dataflow.yml` |
| `communication` | 顶层 | 定义工作流通信层配置 | `communication: { _unstable_local: UnixDomain }` | `examples/rust-dataflow/dataflow_socket.yml` |
| `_unstable_local` | `communication` 下 | 指定本地通信方式 | `_unstable_local: UnixDomain` | `examples/rust-dataflow/dataflow_socket.yml` |

AI 使用规则：

- 绝大多数示例只需要 `nodes`
- 只有在通信层有明确要求时才补 `communication`
- `_unstable_local` 这类字段带明显实验性质，默认不要主动生成，除非用户明确要指定通信方式

### 5.8 节点字段字典

这一组字段直接决定节点怎么被命名、构建、启动和连接。

| 字段 | 位置 | 作用 | 最小示例 | 对应 example |
| --- | --- | --- | --- | --- |
| `id` | node | 节点唯一标识，供连接引用 | `id: rust-node` | `examples/rust-dataflow/dataflow.yml` |
| `build` | node | 构建或安装节点依赖 | `build: cargo build -p rust-dataflow-example-node` | `examples/rust-dataflow/dataflow.yml` |
| `path` | node | 节点入口路径或命令目标 | `path: ../../target/debug/rust-dataflow-example-node` | `examples/rust-dataflow/dataflow.yml` |
| `inputs` | node | 定义节点输入映射 | `inputs: { random: rust-node/random }` | `examples/rust-dataflow/dataflow.yml` |
| `outputs` | node | 定义节点输出列表 | `outputs: [status]` | `examples/rust-dataflow/dataflow.yml` |
| `env` | node | 注入环境变量 | `env: { CAPTURE_PATH: 0 }` | `examples/python-dataflow/dataflow_dynamic.yml` |
| `restart_policy` | node | 控制节点重启策略 | `restart_policy: on-failure` | `examples/rust-dataflow/dataflow-restart.yml` |
| `_unstable_deploy` | node | 指定节点部署位置 | `_unstable_deploy: { machine: A }` | `examples/multiple-daemons/dataflow.yml` |
| `machine` | `_unstable_deploy` 下 | 指定机器标识 | `machine: B` | `examples/multiple-daemons/dataflow.yml` |
| `operator` | node | 用单个 operator 描述节点主体 | `operator:` | `examples/python-operator-dataflow/dataflow.yml` |
| `operators` | node | 在一个 runtime 节点下挂多个 operator | `operators:` | `examples/multiple-daemons/dataflow.yml` |
| `custom` | node | 用自定义脚本描述节点主体 | `custom: { source: keyboard_op.py }` | `examples/python-operator-dataflow/dataflow_llm.yml` |

AI 使用规则：

- `id` 必须稳定，不要用临时名字
- `build` 和 `path` 经常一起出现，但并不强制一一绑定
- `restart_policy` 只在明确需要故障恢复或常驻重启时才出现
- `_unstable_deploy` 只用于多机场景，单机工作流默认不要写
- `operator`、`operators`、`custom` 三者是互斥结构，默认选最简单的一个

### 5.9 operator / custom 细分字段字典

当 node 不是简单 `path` 进程，而是 operator 或 custom 结构时，AI 需要进一步理解下面这些字段。

| 字段 | 位置 | 作用 | 最小示例 | 对应 example |
| --- | --- | --- | --- | --- |
| `python` | `operator` 下 | 指定 Python operator 入口 | `python: object_detection.py` | `examples/python-operator-dataflow/dataflow.yml` |
| `python.source` | `operator.python` 下 | 显式指定源码文件 | `python: { source: plot.py, conda_env: base }` | `examples/python-operator-dataflow/dataflow_conda.yml` |
| `conda_env` | `operator.python` 下 | 指定 Conda 环境 | `conda_env: base` | `examples/python-operator-dataflow/dataflow_conda.yml` |
| `shared-library` | `operators` 下的 operator | 指向共享库实现 | `shared-library: ../../target/debug/multiple_daemons_example_operator` | `examples/multiple-daemons/dataflow.yml` |
| `send_stdout_as` | `operator` 下 | 把标准输出转成工作流输出 | `send_stdout_as: stdout` | `examples/python-operator-dataflow/dataflow.yml` |
| `source` | `custom` 下 | 指定自定义节点源码入口 | `source: keyboard_op.py` | `examples/python-operator-dataflow/dataflow_llm.yml` |
| `inputs` | `operator` / `custom` 下 | 定义 operator 或 custom 的输入 | `inputs: { image: webcam/image }` | `examples/python-operator-dataflow/dataflow.yml` |
| `outputs` | `operator` / `custom` 下 | 定义 operator 或 custom 的输出 | `outputs: [bbox, stdout]` | `examples/python-operator-dataflow/dataflow.yml` |
| `build` | `operator` 下 | 定义 operator 依赖构建 | `build: pip install -r requirements.txt` | `examples/python-operator-dataflow/dataflow.yml` |

AI 使用规则：

- `python` 可以是简单字符串，也可以是对象；只有需要额外配置时才扩成对象
- `conda_env` 只在 Conda 隔离环境确实存在时使用，不要默认生成
- `shared-library` 通常和 `operators` 一起出现，而不是单独挂在普通 node 上
- `send_stdout_as` 适合把文本、日志或助手消息送到下游节点
- `custom.source` 更像“自定义节点入口”，不要和 `operator.python` 混用

### 5.10 字段示例速查

如果 AI 需要快速知道“某个字段应该去看哪个 example”，可以先查这张表：

| 想解释的字段 | 先看哪个 example |
| --- | --- |
| `nodes` / `id` / `path` / `inputs` / `outputs` | `examples/rust-dataflow/dataflow.yml` |
| `env` | `examples/python-dataflow/dataflow_dynamic.yml` |
| `operator` / `python` / `send_stdout_as` | `examples/python-operator-dataflow/dataflow.yml` |
| `python.source` / `conda_env` | `examples/python-operator-dataflow/dataflow_conda.yml` |
| `custom` / `source` | `examples/python-operator-dataflow/dataflow_llm.yml` |
| `operators` / `shared-library` / `_unstable_deploy` / `machine` | `examples/multiple-daemons/dataflow.yml` |
| `communication` / `_unstable_local` | `examples/rust-dataflow/dataflow_socket.yml` |
| `restart_policy` | `examples/rust-dataflow/dataflow-restart.yml` |

## 6. 默认决策规则

当用户没有给够信息时，AI 可以使用下面这些默认值，但必须写出来：

- 默认用 3 节点结构：`source -> processor -> sink`
- 默认优先单机工作流，而不是分布式部署
- 默认优先选仓库里已有示例风格最接近的语言和命令
- 默认使用清晰的蛇形命名或短横线命名，不使用含糊缩写
- 默认先给最小可运行版本，再给可扩展建议

如果默认值会明显影响结果，例如“到底用 Python 还是 Rust”，AI 应该把它放进待确认项，而不是静默决定。

## 7. 审核规则

当 AI 审核已有工作流时，不要只做文案润色，要按下面的顺序检查：

1. 节点职责是否过载
2. 输入输出是否一致
3. `build` / `path` / `env` 是否完整
4. 运行命令是否和语言绑定一致
5. 是否存在隐含假设没有写出来
6. 是否有更接近仓库示例的写法

审核结果应该分成三类：

- 明确错误
- 可改进项
- 待确认项

## 8. 修订规则

当 AI 修改已有工作流时，必须保留原有结构里仍然正确的部分，只改有问题的地方。

修订时优先做这几件事：

- 修正错误连接
- 补齐缺失字段
- 拆掉职责过重的节点
- 把模糊命名改成稳定命名
- 把“口头约定”补成明确字段或说明

不要为了“更优雅”而整篇重写，除非原工作流已经无法维护。

## 9. 生成模板

下面这段可以直接给 AI：

```text
请按照 Dora 工作流的最佳实践，为我生成一个可运行的 dataflow.yml，并同时给出节点说明和运行步骤。

必须遵守以下规则：
- 先拆节点职责，再定义连接关系，再补构建和运行信息
- 一个节点只负责一个明确任务
- 明确列出每个节点的输入和输出
- 连接关系写在 YAML 里，不要写成节点内部逻辑
- 输出必须包含：目标、节点设计、完整 YAML、运行步骤、假设与待确认项
- 如果信息不足，不要臆造事实，要明确指出缺失信息或使用受控默认值

请优先生成最小可运行版本，并保证命名清晰、结构易改、适合后续 AI 继续扩展。
```

## 10. 审核模板

下面这段用于检查已有工作流：

```text
请审核这份 Dora 工作流配置和说明，重点检查：
- 节点职责是否清晰
- 输入输出是否对齐
- build/path/env 是否完整
- 运行步骤是否可执行
- 是否存在没有写出的默认假设
- 是否有更贴近 Dora 示例仓库的写法

请按三类输出：
1. 明确错误
2. 可改进项
3. 待确认项

如果你建议修改，请给出修订后的 YAML 和修订理由。
```

## 11. 修订模板

下面这段用于让 AI 基于已有草稿继续改：

```text
请基于现有 Dora 工作流草稿进行修订，不要无意义重写。

修订目标：
- 保留正确部分
- 只修改错误连接、缺失字段、命名不清或职责过重的问题
- 继续保持“可运行、可读、可扩展”

输出必须包含：
1. 修订摘要
2. 修订后的完整 YAML
3. 更新后的运行步骤
4. 仍然存在的待确认项
```

## 12. 与仓库示例对齐的建议

当 AI 需要找参考实现时，优先参考这些示例：

- `examples/python-dataflow`
- `examples/python-operator-dataflow`
- `examples/python-dataflow-builder`
- `examples/python-async`
- `examples/multiple-daemons`
- `examples/rust-dataflow`

优先级规则：

- 想写最小 YAML：先看 `python-dataflow`
- 想写 operator 场景：先看 `python-operator-dataflow`
- 想让 AI 以编程方式生成工作流：先看 `python-dataflow-builder`
- 想解释异步节点：先看 `python-async`
- 想解释多机部署和 `operators:`：先看 `multiple-daemons`
- 想给 Rust 用户示例：先看 `rust-dataflow`

## 13. 最终检查清单

在交付前，AI 应逐项确认：

- 我是否解释了工作流目标
- 我是否列出了节点职责
- 我是否给出了完整 YAML
- 我是否选择了正确的数据流细分结构
- 我是否给出了运行命令
- 我是否说明了默认假设
- 我是否指出了待确认项
- 我是否避免臆造 Dora 字段、包名或目录

如果以上任意一项答案是否定的，这份输出就还不能算完成。
