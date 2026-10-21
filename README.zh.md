[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [한국어](README.ko.md) · [العربية](README.ar.md)

# 模型自己写记忆

一条记忆的边界由谁来划。

这个仓库是测评集。笔记在 [`cases.json`](cases.json)。调用模型的运行脚本在 [codeagent_memory](https://github.com/HanochZhu/codeagent_memory) 的 `eval/write_memory`，第一次运行时下载这份文件。本仓库里也有同一份脚本。

## 目录

- [测什么](#measure)
- [两种写法](#arms)
- [四篇笔记](#notes)
- [怎么记分](#scoring)
- [运行](#run)

<a id="measure"></a>

## 测什么

cam 里其他评测都是测试框架写好正文，再调用 `cam add`。没有一个让模型自己决定一条记忆写到哪里为止。这里把一篇排查笔记交给模型，让它通过 `cam mcp` 自己调用 `cam_add`，写进一个临时项目。

<a id="arms"></a>

## 两种写法

两种写法看同一批笔记，工具也相同。差别在指令。`legacy` 也看不到 `memories` 参数。

| 写法 | 指令 |
| --- | --- |
| `legacy` | 只存一条记忆，正文是整篇调查。 |
| `split` | 一条记忆是一条事实，或一个问题的解决路径（现象、原因、改法、如何验证）。两段内容能分别被问到，或者会分别过时，就分开存。同一次解决得到的多条放进一次 `cam_add` 的 `memories`，用 `#0`、`#1` 在 `links` 里互指。 |

`split` 的这句话就是 cam 的写回规则。

<a id="notes"></a>

## 四篇笔记

四篇笔记都是公开仓库 [codeagent_memory](https://github.com/HanochZhu/codeagent_memory) 里的行为。每条必选短语都出现在出处文件中。每条 case 的 `source` 字段记下了路径。

| 笔记 | 期望 | 金标 | 出处 |
| --- | --- | --- | --- |
| `recall-pipeline` | 拆开 | 七条事实：RRF、两路门槛、snippet 截断、默认向量模型、hash 开关、拒绝回退、召回不走 `solution_links`。其中四组应当互连。 | `src/memory/recall`、`src/memory/embed.rs` |
| `stale-binary` | 留在一条 | 一个问题的解决路径：可执行文件换了之后，MCP 在工具结果里附上重启提示。拆开算错。 | `src/mcp.rs` |
| `recall-and-forgetting` | 拆开 | 融合常数和遗忘曲线，两条应当互连。 | `src/memory/recall/fuse.rs`、`src/memory/ebbinghaus.rs` |
| `read-and-delete` | 拆开 | `cam read` 怎么读，以及记忆不删除。这两条互不相关。 | 仓库 `README.md` |

<a id="scoring"></a>

## 怎么记分

[`score.py`](score.py) 给写下的记忆记分，不另请一个模型。

| 结果 | 规则 |
| --- | --- |
| 覆盖 | 有一条记忆包含该事实要求的全部短语。 |
| 独占 | 这条记忆里没有另一条金标事实。 |
| 链接 | 两条记忆之间有边，或者两条事实落在同一条记忆里。 |

短语是公开源码里的标识符（`RRF_K`、`S * 1.7`）。模型换一种说法、把短语丢掉，就算漏。这会低估流畅改写，但能看出一条正文是不是塞进了好几条事实。

<a id="run"></a>

## 运行

`cam` 必须接受 `memories` 数组。`0.1.5` 还没有这个参数。设置 `CAM_BIN`，或把这个二进制放进 `PATH`。

`DEEPSEEK_API_KEY` 放在环境变量、`.deepseek.env` 或 `.env`。这三个文件不入库。默认模型是 `deepseek-flash`，用 `MODEL` 覆盖。写入走 `cam mcp`，向量用 hash embedder，项目目录是临时的。每次运行把 JSON 写到 `results/`，该目录不入库。

在本仓库：

```bash
python3 test_score.py
python3 run.py
MODEL=deepseek-flash python3 run.py --arm split --case recall-pipeline
```

在 codeagent_memory 的检出里，运行脚本第一次会把 `cases.json` 下载到 `eval/write_memory/data/`。`CAM_EVAL_WRITE_MEMORY` 可以改这个路径。

```bash
python3 eval/write_memory/test_score.py
python3 eval/write_memory/run.py
```

`test_score.py` 不调用模型。
