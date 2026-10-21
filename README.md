# 模型自己写记忆

这套测评问的是：一条记忆的边界由谁来划。

[cam](https://github.com/HanochZhu/codeagent_memory) 里已有的评测都是测试框架写好正文，再调用 `cam add`。没有一个让模型自己决定一条记忆写到哪里为止。这里把一篇排查笔记交给模型，让它通过 `cam mcp` 自己调用 `cam_add`，写进一个临时项目。

## 两种写法

两种写法看同一批笔记，工具也相同。差别只在指令，以及 `legacy` 看不到 `memories` 参数。

| 写法 | 指令 |
| --- | --- |
| `legacy` | 只存一条记忆，正文是整篇调查。 |
| `split` | 一条记忆是一条事实，或一个问题的解决路径（现象、原因、改法、如何验证）。两段内容能分别被问到，或者会分别过时，就分开存。同一次解决得到的多条放进一次 `cam_add` 的 `memories`，用 `#0`、`#1` 在 `links` 里互指。 |

`split` 的这句话和 cam 写回指令一致：能分别被问到，或者会分别过时，就分开存。

## 四篇笔记

笔记是手工标注的，金标在 `cases.json`。

| 笔记 | 期望 | 金标 |
| --- | --- | --- |
| `tool-truncation` | 拆开 | 七条事实：预算、尾部文件、实验开关、媒体上限、已删除的批量上限、shell、Unity。其中四组应当互连。 |
| `lsp-toast` | 留在一条 | 一个 bug 的完整解决路径。拆开就算错。 |
| `recall-and-forgetting` | 拆开 | 两套机制，并且应当互连。 |
| `read-and-delete` | 拆开 | 两条互不相关的策略。 |

## 怎么记分

记分不另请一个模型，规则在 `score.py`。

一条金标事实被覆盖，是指有一条记忆包含它要求的全部短语。它算独占，是指这条记忆里没有另一条金标事实。要求的链接算命中，是指这两条记忆之间有边，或者两条事实落在同一条记忆里。

短语用的是标识符（`buildOversizedToolResultOutputs`、`S * 1.7`）。模型换一种说法、把短语丢掉，就算漏。这会低估流畅改写，但能看出一条正文是不是塞进了好几条事实。

## 一次结果

`deepseek-flash`，2026-10-21，hash embedder，四篇笔记：

| 写法 | 平均条数 | 最长正文 | 覆盖率 | 独占率 |
| --- | ---: | ---: | ---: | ---: |
| `legacy` | 1.0 | 845 字 | 0.88 | 0.38 |
| `split` | 3.5 | 370 字 | 0.62 | **0.62** |

split 把截断那篇拆成 8 条，LSP 修复仍是 1 条。该连的 4 组边一条都没连上。覆盖率下降，是因为有的金标短语被拆到不同记忆里，或者被换了说法。两边都漏掉了「记忆不删除」这条策略。这条规则能拆开长正文，还没能让模型把拆开的事实连起来。

## 运行

需要一个会接受 `memories` 数组的 `cam`。`0.1.5` 还没有这个参数。设置 `CAM_BIN`，或者让这个 `cam` 出现在 `PATH` 上。密钥放在环境变量、`.deepseek.env` 或 `.env` 里，这三个文件不会被提交。

```bash
python3 test_score.py
python3 run.py
MODEL=deepseek-flash python3 run.py --arm split --case tool-truncation
```

默认模型是 `deepseek-flash`。写入走 `cam mcp`，向量用 hash embedder，项目目录是临时的。每次运行的明细写到 `results/`，该目录不入库。

## Design

The other cam benchmarks store text the harness already wrote. This set gives an investigation note to a model and lets the model call `cam_add` through `cam mcp`.

`legacy` asks for one full write-up and hides the `memories` parameter. `split` states the current rule: one memory is one fact, or the solution path for one problem. Pieces that can be asked about separately, or that can become outdated separately, are stored apart and linked with `#0`, `#1`, ... inside one `cam_add`.

Scoring is lexical. A gold fact is covered when one memory contains every required phrase, and exclusive when that memory contains no other gold fact. A required link counts when those memories share an edge, or both facts land in the same memory. Required phrases are identifiers, so a paraphrase that drops them is a miss.

On 2026-10-21, `deepseek-flash` moved the mean exclusive rate from 0.38 to 0.62 and the longest body from 845 to 370 characters. Coverage fell from 0.88 to 0.62. The truncation note became 8 memories, the LSP fix stayed one memory, and none of the 4 expected links were written.
