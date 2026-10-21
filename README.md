[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [한국어](README.ko.md) · [العربية](README.ar.md)

# Model-written memories

Who decides where one memory ends.

This repository is the case set. The notes are [`cases.json`](cases.json). The runner lives in [codeagent_memory](https://github.com/HanochZhu/codeagent_memory) at `eval/write_memory` and downloads this file on first use. The same runner is also in this repository.

## Contents

- [What it measures](#measure)
- [Two arms](#arms)
- [Notes](#notes)
- [Scoring](#scoring)
- [Run](#run)

<a id="measure"></a>

## What it measures

The other cam benchmarks store text the harness already wrote, then call `cam add`. None of them ask a model to decide where one memory stops. Here the model receives one investigation note and calls `cam_add` itself, through `cam mcp`, into a throwaway project.

<a id="arms"></a>

## Two arms

Both arms see the same notes and the same tool. The difference is the instruction. `legacy` is also not shown the `memories` parameter.

| Arm | Instruction |
| --- | --- |
| `legacy` | Store one memory. The body is the full investigation. |
| `split` | One memory is one fact, or the solution path for one problem (symptom, cause, change, and how to verify). If two pieces can be asked about separately, or can become outdated separately, store them apart. Several memories from the same solution go in one `cam_add` as `memories`, using `#0`, `#1`, ... for `links`. |

The `split` sentence is the cam write-back rule.

<a id="notes"></a>

## Notes

The four notes are public behavior of [codeagent_memory](https://github.com/HanochZhu/codeagent_memory). Every required phrase occurs in the cited file. Each case's `source` field records that path.

| Note | Expect | Gold | Source |
| --- | --- | --- | --- |
| `recall-pipeline` | split | Seven facts: RRF, the two gates, snippet clipping, the default embedding model, the hash switch, refusing fallback, and that recall does not walk `solution_links`. Four pairs should be linked. | `src/memory/recall`, `src/memory/embed.rs` |
| `stale-binary` | keep | One solution path: after the executable changes, MCP adds a restart notice to the tool result. Splitting it is a miss. | `src/mcp.rs` |
| `recall-and-forgetting` | split | The fusion constant and the forgetting curve, linked to each other. | `src/memory/recall/fuse.rs`, `src/memory/ebbinghaus.rs` |
| `read-and-delete` | split | How `cam read` works, and that memories are never deleted. The two policies are unrelated. | repository `README.md` |

<a id="scoring"></a>

## Scoring

[`score.py`](score.py) scores the stored memories. It does not call another model.

| Outcome | Rule |
| --- | --- |
| Covered | One memory contains every required phrase of the fact. |
| Exclusive | That memory contains no other gold fact. |
| Link | The two memories share an edge, or both facts landed in the same memory. |

The phrases are identifiers from the public source (`RRF_K`, `S * 1.7`). A paraphrase that drops them counts as a miss. That under-counts a fluent rewrite, and it shows whether one body is holding several facts.

<a id="run"></a>

## Run

`cam` must accept a `memories` array. Release `0.1.5` does not. Set `CAM_BIN`, or put that binary on `PATH`.

Put `DEEPSEEK_API_KEY` in the environment, `.deepseek.env`, or `.env`. Those files are gitignored. The default model is `deepseek-flash`. `MODEL` overrides it. Writes use `cam mcp` and the hash embedder. The project directory is temporary. Each run writes JSON under `results/`, which is gitignored.

From this repository:

```bash
python3 test_score.py
python3 run.py
MODEL=deepseek-flash python3 run.py --arm split --case recall-pipeline
```

From a codeagent_memory checkout, the runner downloads `cases.json` to `eval/write_memory/data/` on first use. `CAM_EVAL_WRITE_MEMORY` overrides that path.

```bash
python3 eval/write_memory/test_score.py
python3 eval/write_memory/run.py
```

`test_score.py` does not call the model.
