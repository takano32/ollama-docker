# ollama-docker

箱に1台の共有 [ollama](https://github.com/ollama/ollama)。量子化したモデル（GGUF）を動かす係で、
埋め込みや小さい LLM を使いたいサービスは、ollama をそれぞれ立てずにここへ繋ぐ（モデルとイメージが重いので1台にまとめる）。
公式イメージ `ollama/ollama` をそのまま使う（版は `.env` の `OLLAMA_VERSION`、既定 0.40.5、arm64 あり）。

## 使い方

```sh
cd ~/opt/ollama-docker
docker compose up -d
docker compose exec ollama ollama pull embeddinggemma:300m-qat-q8_0   # 338MB。Gemma の同意手続きは要らなかった
docker compose exec ollama ollama list
docker compose down                                                    # -v を付けなければモデルは残る
```

- ホストからは `127.0.0.1:11434` だけ（`curl http://127.0.0.1:11434/api/tags`）。外には開けない。認証は無い。
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
- MCP のサーバーにはならない（HTTP の口だけ）。MCP で呼びたくなったら、橋渡しの MCP サーバーを別に足す。

## 設定（.env、無くても既定で動く）

| 変数 | 意味 | 既定 |
|---|---|---|
| `OLLAMA_VERSION` | イメージのタグ | `0.40.5` |
| `OLLAMA_KEEP_ALIVE` | モデルをメモリから下ろすまでの時間。`-1` は下ろさない | `-1` |
| `OLLAMA_MAX_LOADED_MODELS` | 同時に載せるモデルの数 | `1` |
| `OLLAMA_NUM_PARALLEL` | 1モデルあたりの同時処理数 | `1` |
| `OLLAMA_MEM_LIMIT` | コンテナのメモリ上限 | `3g` |

`OLLAMA_KEEP_ALIVE` を `-1` にしているのは、下ろすと次の呼び出しで読み直し（embeddinggemma Q8 で約2秒）が挟まるため。
`OLLAMA_MAX_LOADED_MODELS=1` なので、2つ目のモデルを呼ぶと1つ目が下ろされる。使うサービスが増えて取り合うなら上げる（メモリと相談）。

## 使っているところ

- [hindsight-docker](https://github.com/takano32/hindsight-docker)：意味検索の埋め込み（2026-10-16 の日本語の比較では `embeddinggemma:300m-qat-q8_0` が一番良かった。表は hindsight-docker の README）。
