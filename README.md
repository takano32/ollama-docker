# ollama-docker

箱に1台の共有 [ollama](https://github.com/ollama/ollama)。量子化したモデル（GGUF）を動かす係で、
埋め込みや小さい LLM を使いたいサービスは、ollama をそれぞれ立てずにここへ繋ぐ（モデルとイメージが重いので1台にまとめる）。
公式イメージ `ollama/ollama` をそのまま使う（版は `.env` の `OLLAMA_VERSION`、既定 0.40.5＝2026-10-16 時点の最新リリース、arm64 あり）。
2つ目のサービス `mcp` は、ollama の小型 LLM を MCP の道具として出す橋渡し（既存の公開パッケージを bunx で起こす。自作しない）。

## 使い方

```sh
cd ~/opt/ollama-docker
docker compose up -d
docker compose exec ollama ollama pull embeddinggemma:300m-qat-q8_0   # 338MB。埋め込み（Hindsight 用）。Gemma の同意手続きは要らなかった
docker compose exec ollama ollama pull qwen3:1.7b                      # 1.4GB。小型 LLM（MCP 用、Q4_K_M）
docker compose exec ollama ollama list
docker compose down                                                    # -v を付けなければモデルは残る
```

- ホストからは `127.0.0.1:11434`（ollama）と `127.0.0.1:11435`（MCP）だけ（`curl http://127.0.0.1:11434/api/tags`）。外には開けない。認証は無い。
- 別の compose から使うときは、その compose に network を足す。ホスト名は `ollama`、ポートは 11434。
  OpenAI 互換の口は `http://ollama:11434/v1`（埋め込みは `/v1/embeddings`）。

  ```yaml
  services:
    app:
      networks: [ollama]
  networks:
    ollama:
      external: true
  ```

- モデルは名前付きボリューム `ollama_models`（コンテナ内 `/root/.ollama`）。要らないモデルは `docker compose exec ollama ollama rm <タグ>`。
- `OLLAMA_MAX_LOADED_MODELS=2` なので、埋め込みと LLM の2つを同時にメモリに置いたままにできる（3つ目を呼ぶと古いほうが下ろされる）。

## MCP の橋渡し（`mcp` サービス）

ollama 自体は MCP のサーバーにならない（HTTP の口だけ）ので、stdio の MCP サーバー
[ollama-mcp-server](https://github.com/hyzhak/ollama-mcp-server)（hyzhak、MIT、npm 1.1.0）を
[mcp-proxy](https://github.com/punkpeye/mcp-proxy)（MIT、npm 6.7.20）で streamable HTTP にして `127.0.0.1:11435/mcp` に出す。
どちらも `oven/bun` の公式イメージの中で `bunx`（版固定）で起こす。初回だけ npm から落とし、名前付きボリューム `ollama_bun_cache` に残す。

Claude Code への登録（このプロジェクトだけ＝local スコープ）と、危ない道具を塞ぐ deny（`.claude/settings.json` の `permissions.deny`）:

```sh
claude mcp add --transport http --scope local ollama http://127.0.0.1:11435/mcp
```

```json
{ "permissions": { "deny": ["mcp__ollama__pull", "mcp__ollama__push", "mcp__ollama__rm", "mcp__ollama__cp", "mcp__ollama__create"] } }
```

道具は9つ。使うのは下の2つで、**モデルを書き換える5つ（`pull`／`push`／`rm`／`cp`／`create`）は使う側の deny で塞ぐ前提**（橋渡し側では選べない）。

| 道具 | 中身 |
|---|---|
| `chat_completion` | `model`・`messages`（system／user／assistant）・`temperature`・`think` を ollama の `/api/chat` に送り、OpenAI の chat.completion の形で返す |
| `run` | `name`（モデル）・`prompt`・`temperature`・`think`・`images` を `/api/generate` に送り、本文だけ返す（thinking があれば `<think>…</think>` を前に付ける） |
| `list`／`show` | 落としてあるモデルの一覧／1つの詳細（読むだけ） |
| `pull`／`push`／`rm`／`cp`／`create` | モデルの取得・公開・削除・複製・派生。**deny で塞ぐ** |

### なぜ `ollama-mcp-server`（hyzhak）か

候補は、前に中身を読んで絞った ollama 用の2つ（どちらも MIT、npm で配布、stdio）。最初は「モデルを書き換える道具が最初から無い」ほうを取るつもりだったが、
thinking を切れないことが決め手になって、こちらにした。

- **`ollama-mcp-server` 1.1.0**（hyzhak、2025-07 公開、MCP SDK 1.13.0、依存は SDK と ollama の JS クライアントの2つ）：`run`／`chat_completion` に **`think` と `temperature` を渡せる**。
  代わりに `pull`／`push`／`rm`／`cp`／`create` の道具も出す（ollama の API をほぼ全部包む作り）。ファイルやシェルに触る道具は無く、外へ繋ぐ先は `OLLAMA_HOST` だけ。
  README に「上限 60 秒」とあるが、1.1.0 のコードに上限の設定は見当たらない（ollama の JS クライアントに素で任せている）。2025-07 から更新が無い。
- `@ask-llm/ollama-mcp` 0.7.0（Lykhoyda/ask-llm、2026-09 公開、MCP SDK 1.30 系）：道具は `ask-ollama`／`ping`／`get-usage-stats` の3つで、モデルを書き換える道具が無い。
  ただし **`think` も `temperature` も渡せない**（`/api/chat` に `model`・`messages`・`stream:false` だけ送る）ので、qwen3 のように既定が thinking のモデルは毎回考えてから答える（下の計測で1問 249 秒）。
  本文に `/no_think` を書いても ollama 0.40.5 では効かなかった（thinking を 400 トークン使い切って本文が空）。
  mcp-proxy 越しの握手と `ping` まで通ることは確かめた（initialize 0.07s、道具3つ）。

### thinking を切る（`think: false` を毎回渡す）

qwen3:1.7b は `/api/show` の `thinking` が `{"values":[false,true],"default":true}` で、**何も渡さなければ考えてから答える**。2 vCPU だと1問 249 秒（思考 1587 字）。
ollama 0.40.5 にはサーバー側で既定を切る環境変数や Modelfile の指定が無い（`envconfig` と `parser` に think の項目が無い）ので、**呼ぶたびに `think: false` を渡す**。
`run`／`chat_completion` の引数にあるので、Claude Code から呼ぶときは必ず付ける（付け忘れると 4 分待たされる）。本文の `/no_think` は効かない。

```
think 無指定      249.0s  eval 369 tok  thinking 1587 字  本文あり
think:false        14.5s  eval  40 tok  thinking なし    （埋め込みの読み込みと同時に走った値。MCP 経由の下の計測では 2.4 秒）
本文に /no_think  155.5s  eval 400 tok  thinking 1468 字  本文なし（上限で切れた）
```

## 計測（2026-10-16、この箱＝aarch64 2 vCPU、GPU なし、ollama 0.40.5）

### 2つのモデルを同時に載せた待機メモリ（`OLLAMA_MAX_LOADED_MODELS=2`）

embeddinggemma:300m-qat-q8_0 と qwen3:1.7b を両方載せ、生成を3回したあとに落ち着いた値（`docker stats`、cgroup v2 の `memory.current − inactive_file`）。

| | ollama（2モデル同時） | llama-server 2本（llama-cpp-docker、同日の計測） |
|---|---|---|
| モデルを動かす側 | **2471 MiB**（cgroup の anon 2506 MiB、file 371 MiB） | 371（埋め込み）＋ 2586（LLM）＝ 2957 MiB |
| MCP の橋渡し | 74 MiB（bun＋mcp-proxy＋橋渡し） | 140 MiB |
| 合計 | **約 2.5 GiB** | 約 3.0 GiB |

- `ollama ps` の見積もりは qwen3:1.7b 1799 MiB（ctx 4096）、embeddinggemma 594 MiB（ctx 2048）。
- 同日の夕方に1つずつ測った値（627 MiB、3019 MiB）より小さい。あのときは `pull` 直後で、書いたばかりのモデルのファイルがページキャッシュとして cgroup に乗っていた
  （今回も `pull` 直後は同じ型で、落ち着くまで待ってから測った）。llama-server の数字も同じ `docker stats` だが、こちらは落ち着くまで待ったかを確かめていないので、差の一部は測り方かもしれない。
- `mem_limit` は 4g。待機 2.5 GiB に生成時の KV キャッシュと、3つ目のモデルを一時的に呼んだときの余地を見込んだ。

### 速さ（MCP 経由、`think: false`、temperature 0）

| 道具 | 問い | 所要 |
|---|---|---|
| `chat_completion` | 一言の自己紹介 | 2.4 秒 |
| `run` | 1文の敬体への言い換え | 5.3 秒 |
| `chat_completion` | 約 120 字の文章を3文に要約 | 10.1 秒 |

握手は initialize 0.06 秒、tools/list 0.02 秒。モデルの読み込みは embeddinggemma 5.1 秒（`/api/embed` の初回）、qwen3:1.7b の pull は 237 秒（1.4GB、この回線）。
前回（同日夕方）の5問の bench は qwen3:1.7b で1問 5.0 秒の中央値だったので、同じ水準。

## 設定（.env、無くても既定で動く）

| 変数 | 意味 | 既定 |
|---|---|---|
| `OLLAMA_VERSION` | イメージのタグ | `0.40.5` |
| `OLLAMA_KEEP_ALIVE` | モデルをメモリから下ろすまでの時間。`-1` は下ろさない | `-1` |
| `OLLAMA_MAX_LOADED_MODELS` | 同時に載せるモデルの数 | `2` |
| `OLLAMA_NUM_PARALLEL` | 1モデルあたりの同時処理数 | `1` |
| `OLLAMA_MEM_LIMIT` | ollama コンテナのメモリ上限 | `4g` |
| `OLLAMA_MCP_PORT` | MCP の橋渡しをホストに出すポート（127.0.0.1 だけ） | `11435` |
| `OLLAMA_MCP_VERSION`／`MCP_PROXY_VERSION`／`BUN_VERSION` | 橋渡しの版 | `1.1.0`／`6.7.20`／`1.4.3-alpine` |
| `OLLAMA_MCP_MEM_LIMIT` | 橋渡しコンテナのメモリ上限 | `512m` |

`OLLAMA_KEEP_ALIVE` を `-1` にしているのは、下ろすと次の呼び出しで読み直し（embeddinggemma Q8 で約2秒、qwen3 1.7B で約 13 秒）が挟まるため。

## 使っているところ

- [hindsight-docker](https://github.com/takano32/hindsight-docker)：意味検索の埋め込み（2026-10-16 の日本語の比較では `embeddinggemma:300m-qat-q8_0` が一番良かった。表は hindsight-docker の README）。
- nilkani06（Claude Code）：`mcp` サービス経由で小型 LLM（`qwen3:1.7b`、`think: false`）を叩く。

## 経緯

2026-10-16 の夜にいちど ollama をやめて [llama-cpp-docker](https://github.com/takano32/llama-cpp-docker)（llama-server 2本）に替え、同じ夜のうちに戻した。
llama-server は1プロセス1モデルなので埋め込みと LLM で2本立てになり、ollama の「1つの口で2つのモデルを扱える」手軽さを取った。
llama-server のときの数字（イメージ 1.2GB、待機 371＋2586 MiB）は llama-cpp-docker の README に残してある。
