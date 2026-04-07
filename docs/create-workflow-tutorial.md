# 创建 Dora 工作流：从 YAML 到可运行数据流

这是一篇面向新手和 AI 的 Dora 工作流入门教程。

你可以把 Dora 工作流理解成一张“蓝图”：先用 `dataflow.yml` 声明有哪些节点、节点之间怎么连线，再让 `dora` 按这张蓝图启动进程并传递数据。这个表达方式和 Dora 中文社区的入门教程保持一致，但这里会更偏向“可直接生成、可直接改写”的写法。

这套写法不是 Python 专属。Rust、C、C++ 例子虽然启动方式不同，但工作流的组织方式仍然是同一套：节点、输入、输出、构建命令和运行入口。

## 1. 先理解工作流

一个 Dora 工作流通常由三部分组成：

- `nodes`：工作流里的节点列表。
- `inputs` / `outputs`：节点之间的数据连接。
- `build` / `path` / `env`：节点怎么安装、怎么启动、需要哪些环境变量。

最重要的一点是，Dora 不要求你在代码里手写“谁把数据发给谁”。连接关系写在 YAML 里，节点代码只负责收数据、处理数据、再发出结果。

## 2. 一个最小可用的工作流

下面这个例子参考了仓库里的 `examples/python-dataflow`，它展示了一个最常见的结构：定时器触发采集节点，采集节点把数据传给处理节点，最后再传给可视化节点。

```yaml
nodes:
  - id: camera
    build: pip install opencv-video-capture
    path: opencv-video-capture
    inputs:
      tick: dora/timer/millis/20
    outputs:
      - image
    env:
      CAPTURE_PATH: "0"
      IMAGE_WIDTH: "640"
      IMAGE_HEIGHT: "480"

  - id: object_detection
    build: pip install dora-yolo
    path: dora-yolo
    inputs:
      image: camera/image
    outputs:
      - bbox

  - id: plot
    build: pip install dora-rerun
    path: dora-rerun
    inputs:
      image: camera/image
      bbox: object_detection/bbox
```

这段 YAML 的含义很直接：

- `camera` 每 20ms 接收一次 `tick`
- `camera` 产出 `image`
- `object_detection` 消费 `camera/image`，产出 `bbox`
- `plot` 同时消费 `image` 和 `bbox`

## 3. 怎么自己创建一个工作流

按这个顺序写最稳：

1. 先列出节点职责。
2. 再确定每个节点的输出。
3. 再把下游节点的输入接到对应输出上。
4. 最后补 `build`、`path` 和 `env`。

一个实用的原则是：一个节点只做一件事。比如“采集”“推理”“展示”分开写，后面无论是人改还是 AI 改，都更容易定位问题。

## 4. 运行流程

如果你是本地开发，通常按这个顺序：

```bash
dora build dataflow.yml
dora run dataflow.yml
```

如果你的节点依赖 Python 包，仓库里的示例常常会带 `--uv`，比如：

```bash
cd examples/python-dataflow
uv pip install -e ../../apis/python/node --reinstall
dora build ./dataflow.yml --uv
dora run ./dataflow.yml --uv
```

如果你只是想让工作流先跑起来并且后续再管理它，可以参考 `dora start`。仓库里的快速上手文档也把 `check`、`build`、`run`、`start` 这些命令放在一起介绍了。

## 5. 给 AI 的写法

如果你希望让 AI 帮你生成工作流，最好先给它一份结构化输入，而不是一句泛泛的“帮我写一个 Dora 工作流”。

如果你希望把这件事做成可重复执行的 AI 流程，而不是一次性的 prompt，可以继续看 [`ai-workflow-writing-rules.md`](./ai-workflow-writing-rules.md)。

可以直接按下面这个模板喂给 AI：

```text
请根据以下要求生成 Dora 的 dataflow.yml：

1. 节点列表：
   - 节点名
   - 节点职责
   - 输入
   - 输出
   - 是否需要定时器
   - 运行方式（Python/Rust/C/C++）
   - 需要安装的依赖

2. 约束：
   - 每个节点只负责一个明确任务
   - 输入名必须和上游输出名一致
   - 先给出可运行的 YAML，再给出每个节点的简要说明
   - 如果缺少信息，先列出你需要补充的内容
```

如果你想让 AI 直接产出可维护的工作流，可以再加一句：

```text
请优先生成“易读、易改、便于调试”的版本，不要为了最短而省略关键字段。
```

## 6. 常见坑

- 输入名和输出名没对上。
- `path` 指向的节点程序不在环境里。
- `build` 依赖没装完就直接 `run`。
- 把“数据连接”写进节点代码，导致工作流不好维护。

如果遇到问题，优先检查：

1. YAML 里的 `source_node/output_name` 是否写对。
2. 节点包是否已经安装。
3. 节点命令是否能单独运行。

## 7. 一个更适合 AI 复用的工作流模板

```yaml
nodes:
  - id: source
    build: <install command>
    path: <node entry>
    inputs:
      tick: dora/timer/millis/20
    outputs:
      - data

  - id: processor
    build: <install command>
    path: <node entry>
    inputs:
      data: source/data
    outputs:
      - result

  - id: sink
    build: <install command>
    path: <node entry>
    inputs:
      result: processor/result
```

你可以把这份模板直接交给 AI，让它补全具体的包名、节点名和数据类型。

## 8. 推荐你继续看的内容

- Dora 中文社区教程： [介绍：Dora-rs是什么？](https://doracc.com/guide/start/introduction.html)
- Dora 中文社区教程： [快速上手](https://doracc.com/guide/start/quick-start.html)
- Dora 中文社区教程： [数据流](https://doracc.com/guide/tutorial/dataflow.html)
- Dora 中文社区教程： [操作符](https://doracc.com/guide/tutorial/operator.html)
- 仓库规则： [`ai-workflow-writing-rules.md`](./ai-workflow-writing-rules.md)
- 仓库示例：`examples/python-dataflow`
- 仓库示例：`examples/python-operator-dataflow`
- 仓库示例：`examples/python-dataflow-builder`
- 仓库示例：`examples/rust-dataflow`

这篇教程的目标不是把所有 Dora 能力一次讲完，而是让你能快速写出第一个工作流，并且后续能交给 AI 继续扩写、改写和排错。

## 9. 按官方 next page 继续往下学

如果你是按 Dora 中文社区的教程顺序往下读，可以把这条链路直接当成工作流学习路线：

| 页面 | 你会学到什么 | 跟创建工作流的关系 |
| --- | --- | --- |
| [介绍](https://doracc.com/guide/start/introduction.html) | Dora 是什么，为什么用数据流来建模 | 决定你为什么要写 `dataflow.yml` |
| [数据流](https://doracc.com/guide/tutorial/dataflow.html) | 蓝图、节点、输入输出连接 | 决定你怎么画工作流 |
| [节点](https://doracc.com/guide/tutorial/node.html) | 每个节点是独立进程，职责单一 | 决定你怎么拆分任务 |
| [操作符](https://doracc.com/guide/tutorial/operator.html) | 一个节点里如何组织多个轻量处理单元 | 适合复杂节点内部逻辑 |
| [事件流处理](https://doracc.com/guide/tutorial/event-stream.html) | `INPUT`、`STOP`、`RELOAD` 等事件如何到达节点 | 决定节点代码怎么收消息 |
| [数据信息 / Arrow Data](https://doracc.com/guide/tutorial/data-message.html) | 大数据如何高效传输，为什么有共享内存和 `DropToken` | 决定你怎么理解图像、点云等大消息 |
| [API 绑定](https://doracc.com/guide/tutorial/api-binding.html) | 不同语言如何和运行时通信 | 决定你怎么写 Python / Rust / C / C++ 节点 |
| [命令行接口](https://doracc.com/guide/tutorial/dora-cli.html) | `build`、`run`、`start`、`list`、`check`、`up`、`destroy` | 决定你怎么把工作流真正跑起来 |
| [守护进程](https://doracc.com/guide/tutorial/dora-daemon.html) | 单机上负责启动和管理节点的后台进程 | 决定本地运行时怎么工作 |
| [协调器](https://doracc.com/guide/tutorial/dora-coordinator.html) | 多机器分布式编排、节点分发和状态管理 | 决定分布式工作流怎么启动和停止 |

你可以把这条链路理解为：

1. 先用 `数据流` 画蓝图。
2. 再用 `节点` 和 `算子` 填内容。
3. 然后让 `事件流` 和 `Arrow Data` 负责传递消息。
4. 再用 `API 绑定` 写业务代码。
5. 最后用 `CLI`、`守护进程` 和 `协调器` 把它跑起来。

这也是为什么这篇文档先讲“怎么创建工作流”，再补“官方教程 next page 到底”的原因：前者解决你怎么写，后者解决你写完以后系统怎么跑。
