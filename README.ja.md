[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [한국어](README.ko.md) · [العربية](README.ar.md)

# モデルが自分で書く記憶

1件の記憶はどこで終わるのか。

このリポジトリはケース集である。ノートは [`cases.json`](cases.json)。モデルを呼ぶ実行部は [codeagent_memory](https://github.com/HanochZhu/codeagent_memory) の `eval/write_memory` にあり、初回実行時にこのファイルをダウンロードする。同じ実行部はこのリポジトリにもある。

## 目次

- [何を測るか](#measure)
- [二つの書き方](#arms)
- [四つのノート](#notes)
- [採点](#scoring)
- [実行](#run)

<a id="measure"></a>

## 何を測るか

cam の他のベンチマークは、ハーネスが本文を書いてから `cam add` を呼ぶ。モデル自身に、1件の記憶がどこで終わるかを決めさせない。ここでは調査ノートを1枚渡し、`cam mcp` 経由で `cam_add` を呼ばせ、一時プロジェクトへ書かせる。

<a id="arms"></a>

## 二つの書き方

両方とも同じノートと同じツールを見る。違うのは指示である。`legacy` には `memories` 引数も見せない。

| 書き方 | 指示 |
| --- | --- |
| `legacy` | 記憶は1件。本文は調査全体。 |
| `split` | 1件は1つの事実、または1つの問題の解決経路（症状、原因、変更、確認方法）。別々に尋ねられる、または別々に古くなる内容は分ける。同じ解決から出た複数件は1回の `cam_add` の `memories` に入れ、`#0` や `#1` で `links` を張る。 |

`split` の文は cam の書き戻し規則と同じである。

<a id="notes"></a>

## 四つのノート

4件とも公開リポジトリ [codeagent_memory](https://github.com/HanochZhu/codeagent_memory) の振る舞いである。必須句は引用先のファイルにある。各ケースの `source` がそのパスである。

| ノート | 期待 | 金の事実 | 出典 |
| --- | --- | --- | --- |
| `recall-pipeline` | 分割 | 7事実。RRF、二つの関門、snippet 切り詰め、既定の埋め込みモデル、hash スイッチ、フォールバック拒否、recall が `solution_links` を辿らないこと。そのうち4組はリンクする。 | `src/memory/recall`、`src/memory/embed.rs` |
| `stale-binary` | 1件のまま | 1本の解決経路。実行ファイルが替わったあと、MCP はツール結果に再起動の通知を足す。分割は誤り。 | `src/mcp.rs` |
| `recall-and-forgetting` | 分割 | 融合の定数と忘却曲線。互いにリンクする。 | `src/memory/recall/fuse.rs`、`src/memory/ebbinghaus.rs` |
| `read-and-delete` | 分割 | `cam read` の読み方と、記憶は削除しないこと。二つは無関係。 | リポジトリの `README.md` |

<a id="scoring"></a>

## 採点

[`score.py`](score.py) が保存された記憶を採点する。別のモデルは呼ばない。

| 結果 | 規則 |
| --- | --- |
| 被覆 | その事実の必須句をすべて含む記憶が1件ある。 |
| 独占 | その記憶が他の金の事実を含まない。 |
| リンク | 二つの記憶の間に辺がある、または二つの事実が同じ記憶に入った。 |

句は公開ソースの識別子である（`RRF_K`、`S * 1.7`）。言い換えて句を落とすとミスになる。流暢な書き換えは過小評価するが、1件の本文が複数の事実を抱えているかは分かる。

<a id="run"></a>

## 実行

`cam` は `memories` 配列を受け取れなければならない。リリース `0.1.5` にはその引数がない。`CAM_BIN` を設定するか、そのバイナリを `PATH` に置く。

`DEEPSEEK_API_KEY` は環境変数、`.deepseek.env`、または `.env` に置く。これらはコミットしない。既定モデルは `deepseek-flash`。`MODEL` で上書きする。書き込みは `cam mcp` と hash embedder を使い、プロジェクトは一時ディレクトリである。各実行は `results/` に JSON を書く。このディレクトリはコミットしない。

このリポジトリで:

```bash
python3 test_score.py
python3 run.py
MODEL=deepseek-flash python3 run.py --arm split --case recall-pipeline
```

codeagent_memory のチェックアウトでは、初回に `cases.json` を `eval/write_memory/data/` へダウンロードする。パスは `CAM_EVAL_WRITE_MEMORY` で変えられる。

```bash
python3 eval/write_memory/test_score.py
python3 eval/write_memory/run.py
```

`test_score.py` はモデルを呼ばない。
