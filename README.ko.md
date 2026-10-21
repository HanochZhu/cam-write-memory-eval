[English](README.md) · [中文](README.zh.md) · [日本語](README.ja.md) · [한국어](README.ko.md) · [العربية](README.ar.md)

# 모델이 직접 쓰는 기억

기억 한 건이 어디에서 끝나는가.

이 저장소는 케이스 집합이다. 노트는 [`cases.json`](cases.json)에 있다. 모델을 호출하는 실행기는 [codeagent_memory](https://github.com/HanochZhu/codeagent_memory)의 `eval/write_memory`에 있으며, 처음 실행할 때 이 파일을 받는다. 같은 실행기가 이 저장소에도 있다.

## 차례

- [무엇을 재는가](#measure)
- [두 가지 쓰기](#arms)
- [노트 네 편](#notes)
- [채점](#scoring)
- [실행](#run)

<a id="measure"></a>

## 무엇을 재는가

cam의 다른 벤치마크는 하네스가 본문을 쓴 다음 `cam add`를 호출한다. 모델이 기억 한 건의 끝을 스스로 정하지 않는다. 여기서는 조사 노트 한 편을 주고, `cam mcp`로 `cam_add`를 호출해 임시 프로젝트에 쓰게 한다.

<a id="arms"></a>

## 두 가지 쓰기

두 방식은 같은 노트와 같은 도구를 본다. 차이는 지시문이다. `legacy`에는 `memories` 매개변수도 보여 주지 않는다.

| 방식 | 지시 |
| --- | --- |
| `legacy` | 기억은 한 건이다. 본문은 조사 전체이다. |
| `split` | 한 건은 사실 하나이거나, 한 문제의 해결 경로(증상, 원인, 수정, 확인 방법)이다. 따로 물을 수 있거나 따로 낡을 수 있는 내용은 나눈다. 같은 해결에서 나온 여러 건은 한 번의 `cam_add` `memories`에 넣고 `#0`, `#1`로 `links`를 건다. |

`split`의 문장은 cam의 기록 규칙과 같다.

<a id="notes"></a>

## 노트 네 편

네 편 모두 공개 저장소 [codeagent_memory](https://github.com/HanochZhu/codeagent_memory)의 동작이다. 필수 구절은 인용한 파일에 있다. 각 케이스의 `source`가 그 경로이다.

| 노트 | 기대 | 정답 | 출처 |
| --- | --- | --- | --- |
| `recall-pipeline` | 나누기 | 사실 일곱 개. RRF, 두 관문, snippet 자르기, 기본 임베딩 모델, hash 스위치, 폴백 거부, recall이 `solution_links`를 따라가지 않음. 그중 네 쌍은 연결해야 한다. | `src/memory/recall`, `src/memory/embed.rs` |
| `stale-binary` | 한 건으로 유지 | 해결 경로 하나. 실행 파일이 바뀌면 MCP가 도구 결과에 재시작 안내를 붙인다. 나누면 오답이다. | `src/mcp.rs` |
| `recall-and-forgetting` | 나누기 | 융합 상수와 망각 곡선. 서로 연결해야 한다. | `src/memory/recall/fuse.rs`, `src/memory/ebbinghaus.rs` |
| `read-and-delete` | 나누기 | `cam read`가 읽는 방식과, 기억은 삭제하지 않는다는 정책. 둘은 서로 무관하다. | 저장소 `README.md` |

<a id="scoring"></a>

## 채점

[`score.py`](score.py)가 저장된 기억을 채점한다. 다른 모델을 부르지 않는다.

| 결과 | 규칙 |
| --- | --- |
| 포함 | 그 사실의 필수 구절을 모두 담은 기억이 하나 있다. |
| 독점 | 그 기억에 다른 정답 사실이 없다. |
| 링크 | 두 기억 사이에 변이 있거나, 두 사실이 같은 기억에 들어갔다. |

구절은 공개 소스의 식별자이다 (`RRF_K`, `S * 1.7`). 풀어 쓰다 구절을 빠뜨리면 오답이다. 유창한 다시 쓰기는 낮게 치지만, 본문 하나가 사실 여러 개를 담고 있는지는 드러난다.

<a id="run"></a>

## 실행

`cam`은 `memories` 배열을 받아야 한다. 릴리스 `0.1.5`에는 그 매개변수가 없다. `CAM_BIN`을 두거나, 그 바이너리를 `PATH`에 둔다.

`DEEPSEEK_API_KEY`는 환경 변수, `.deepseek.env`, 또는 `.env`에 둔다. 이 파일은 커밋하지 않는다. 기본 모델은 `deepseek-flash`이다. `MODEL`로 바꾼다. 쓰기는 `cam mcp`와 hash embedder를 쓰고, 프로젝트 디렉터리는 임시이다. 실행마다 `results/`에 JSON을 쓴다. 이 디렉터리는 커밋하지 않는다.

이 저장소에서:

```bash
python3 test_score.py
python3 run.py
MODEL=deepseek-flash python3 run.py --arm split --case recall-pipeline
```

codeagent_memory 체크아웃에서는 처음에 `cases.json`을 `eval/write_memory/data/`로 받는다. 경로는 `CAM_EVAL_WRITE_MEMORY`로 바꿀 수 있다.

```bash
python3 eval/write_memory/test_score.py
python3 eval/write_memory/run.py
```

`test_score.py`는 모델을 호출하지 않는다.
