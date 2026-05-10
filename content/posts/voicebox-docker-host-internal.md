---
title: "Claude Codeに任せたら、Docker内のn8nからVOICEBOXに繋がらなくて詰まった話 — host.docker.internal の罠"
date: 2026-05-09
tags: [Claude Code, Docker, n8n, VOICEBOX, host.docker.internal]
description: "Claude Code に YouTube Shorts 自動投稿システムを作ってもらった時、Docker コンテナ内の n8n から Windows ホストの VOICEBOX を叩こうとしてハマった話。localhost ではなく host.docker.internal を使う理由と、Linux Docker での違い。"
slug: "voicebox-docker-host-internal"
draft: false
---

> このシステムは Claude Code (Anthropic) に作ってもらったものです。コードは私が書いていません。この記事も Claude に書いてもらいました。「AI に任せて作ったら、どこで詰まるか」の現場記録として読んでください。

## この記事の前提

私は Claude Code に YouTube Shorts を自動投稿するパイプラインを作ってもらっていて、これはその中で最初に詰まった話です。

パイプライン全体の話はこっちに書きました（先に読まなくても、この記事は単体で読めるように書いています）:

→ [Claude Codeに任せて1ヶ月、宇宙系YouTube自動投稿チャンネルが直面した「日本語品質の壁」](/posts/claude-code-youtube-1month/)

この記事は **その中の VOICEBOX 接続部分** を切り出した詰まりどころのレポートです。

### VOICEBOX とは

Windows / Mac / Linux で動く **無料の音声合成エンジン**。GUI アプリを起動すると、ローカルの `localhost:50021` に音声合成 API のサーバーが立ち上がります。複数の声（speaker）が選べて、条件付きで商用利用も OK。

## なぜ Docker の中の n8n から、ホスト側の VOICEBOX を叩く必要があったか

私が組んでもらったパイプラインの構成はこうなっていました。

| コンポーネント | 動かす場所 |
|---|---|
| n8n（ワークフロー制御） | Docker コンテナ |
| VOICEBOX（音声合成） | Windows ネイティブ |
| FFmpeg（動画レンダリング） | Windows ネイティブ |

n8n は公式の Docker イメージで動かすのが圧倒的に楽です。一方の VOICEBOX は Windows 用の **GUI アプリ** で、起動すると Windows ホスト上の `localhost:50021` に音声合成 API（Engine）が立ち上がる構成。Docker コンテナの中で起動するという発想にはまずなりません。

つまり「コンテナの中の n8n から、コンテナの外（Windows ホスト）の VOICEBOX を HTTP で叩きたい」という構成になります。これは VOICEBOX に限らず、**Whisper・Ollama・ローカル DB など、Windows / Mac ホスト側で動かしているサービスを Docker コンテナから叩く時** に毎回出てくるパターンです。

## VOICEBOX の API 構成

VOICEBOX を起動すると、ホストの `localhost:50021` に Engine の HTTP サーバーが立ち上がります。音声合成は **2 段階の API 呼び出し**になっています。

```
1. POST http://<HOST>:50021/audio_query?speaker=13
   ボディ: text=合成したいテキスト
   レスポンス: 合成パラメータ JSON

2. POST http://<HOST>:50021/synthesis?speaker=13
   ボディ: 上の JSON（speedScale を 1.7 に書き換えた状態）
   レスポンス: WAV バイナリ
```

`speaker` は声のID で、`13` は「青山龍星」です。スピードを 1.7 倍に上げているのは Shorts 向けの調整。

n8n の HTTP Request ノードを 2 個並べて、テキストを渡すと WAV が返ってくる、という流れに落ちます。

## Claude Code に頼むと、たぶんこういう感じになる

実際に依頼した時のやりとりと完全に同じではありませんが、同じシステムを Claude Code に依頼するならこの程度の指示で組めます。

**初回の依頼:**

```
Docker コンテナで動いている n8n のワークフローから、
Windows ホスト上で動いている VOICEBOX (port 50021) を叩いて
音声合成を行うノードを 2 つ組み立ててください。

要件:
- audio_query で合成パラメータを取得
- レスポンスの JSON を編集して speedScale を 1.7 にする
- synthesis で WAV を取得
- speaker は 13（青山龍星）
- 取得した WAV はバイナリとして次のノードに渡す
```

**詰まった時の追加指示:**

```
n8n コンテナから localhost:50021 を叩いたら Connection refused になります。
原因と修正方法を教えてください。
```

このくらいの粒度で投げれば、Claude Code が「コンテナ内の `localhost` はコンテナ自身を指すので、Docker Desktop の `host.docker.internal` を使ってください」と修正してくれます。

## Claude Code だけでは完結しない作業

コードと n8n ノードの設計は Claude が全部やってくれますが、以下は自分で手を動かす必要があります。

- **VOICEBOX のインストール**（Windows 用インストーラを公式から落として実行）
- **VOICEBOX の起動**（GUI アプリを立ち上げる、Engine は自動で port 50021 にバインドされる）
- **Docker Desktop のインストール**
- **n8n コンテナの起動**（docker compose または `docker run`）

ここでも、画面のスクショを Claude に渡せば「次にどのボタンを押すか」「どの設定をどう変えるか」レベルで具体的に指示してくれます。私自身、コードは 1 行も書いていないし、各 GUI でクリックする場所も Claude に教わりながら進めました。

## 詰まったポイント: localhost で繋がらない

n8n のワークフローから VOICEBOX を叩こうとして、最初に書きがちなのが:

```
http://localhost:50021/audio_query?speaker=13
```

これを n8n の HTTP Request ノードに設定して実行すると、**Connection refused** で必ず失敗します。

### 何が起きているか

Docker コンテナの中で `localhost` を解決すると、それは **コンテナ自身** を指します。コンテナの中には VOICEBOX なんて動いていないので、当然繋がりません。

n8n コンテナから見たネットワーク構成は:

```
┌──────────────────────┐
│ n8n コンテナの中     │
│   localhost = ここ  │ ← VOICEBOX はここにいない
└──────┬───────────────┘
       │
       ↓（Docker Desktop のブリッジ）
┌──────────────────────┐
│ Windows ホスト       │
│   localhost:50021    │ ← VOICEBOX はここ
└──────────────────────┘
```

つまり、コンテナの中の `localhost` と、ホストの `localhost` は **別物** です。

### 解決: host.docker.internal を使う

Docker Desktop（Windows / Mac 版）には、コンテナの中から **ホスト側を指す特別な DNS 名** が用意されています。

```
http://host.docker.internal:50021/audio_query?speaker=13
```

これに変えるだけで繋がります。`host.docker.internal` は Docker Desktop が自動的に解決してくれて、コンテナの中からホストマシンの IP を引いてくれる仕組みです。

### Linux の Docker では事情が違う

ここが地味に重要なところで、**Linux のネイティブ Docker では `host.docker.internal` はデフォルトで使えません**。

Docker Engine 20.10 以降であれば、コンテナ起動時に `--add-host=host.docker.internal:host-gateway` を渡せば使えるようになります。

```bash
docker run --add-host=host.docker.internal:host-gateway ...
```

または `docker-compose.yml` に書く場合:

```yaml
services:
  n8n:
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

WSL2 経由で Docker Desktop を使っている場合は、Windows 側が橋渡しするので `host.docker.internal` がそのまま使えます。素の Linux サーバーで運用する時だけ要注意です。

### Windows ファイアウォールの罠

`host.docker.internal:50021` で繋ぐ設定にしても、Windows ファイアウォールが VOICEBOX へのインバウンドをブロックしている場合があります。

VOICEBOX は通常 `127.0.0.1` にバインドされていて、`host.docker.internal` で来た接続はホスト側からは Docker 用の仮想 NIC 経由で見えます。これを許可するには、ファイアウォールで VOICEBOX の Engine（多くの場合 `vv-engine.exe`）を許可する設定が必要なことがあります。

私の環境ではセットアップ時にデフォルトで許可されたので追加設定はしませんでしたが、初回起動の時に Windows が「許可しますか」と聞いてくる確認ダイアログを見落とすと、後でこの罠に当たります。

## 学び・教訓

### 1. Docker のネットワーク隔離を雰囲気で理解しておく必要がある

`localhost` は「実行している場所」によって意味が変わる、というのが Docker の基本動作です。これを知らないと、Claude Code が出してくる構成図を見ても「なぜ `host.docker.internal` を使う必要があるのか」が分からないまま、コピペで動かすことになります。

動かすだけなら問題ないですが、後から自分で構成を変えたい時に詰まります。

### 2. Claude Code には最初から構成を伝えるのが速い

「n8n のワークフローから VOICEBOX を呼び出すノードを作って」だけだと、Claude も「ローカル環境前提」で実装を出してくる可能性があります。

最初の依頼で **「n8n は Docker コンテナの中で動いていて、VOICEBOX は Windows ホスト側で動いています」** と環境構成を明示しておくと、最初から `host.docker.internal` を使う実装が出てきます。これは Claude Code に限らず、AI に作業を依頼する時の一般則だと思います。

### 3. エラーメッセージは Claude にそのまま貼るのが速い

「Connection refused」というエラーが出た時、ググって原因を探すより、エラーメッセージと自分の構成を Claude に貼って「直して」と頼む方が圧倒的に速いです。

私はネットワーク周りの知識が浅いので、最初は「VOICEBOX の起動を忘れたかな」と疑ったりしましたが、Claude にエラーログと n8n のノード設定を見せた瞬間に「コンテナの中の `localhost` の話です」と教えてくれました。

### 4. 「Docker から外」を扱うパターンは意外と汎用

VOICEBOX に限らず、Whisper, Ollama, ローカル DB, ファイルサーバーなど、**ホスト側で動かしたいものを Docker コンテナの中から叩く** という構成はよくあります。

このパターンを 1 回詰まって解決しておけば、似た構成で迷わなくなります。`host.docker.internal` という名前だけでも記憶しておくと、応用が効きます。

## 全体像が気になった人へ

この記事は YouTube Shorts 自動投稿パイプラインの中の、VOICEBOX 接続部分だけを切り出したものです。「結局そのパイプラインで何が起きたか」「Claude Code に任せて 1 ヶ月運用したらどうだったか」のまとめはこちらに書いています:

→ [Claude Codeに任せて1ヶ月、宇宙系YouTube自動投稿チャンネルが直面した「日本語品質の壁」](/posts/claude-code-youtube-1month/)

## このシステムから派生する記事ネタ

このパイプラインを作る中で出てきた、別記事として書きたい知見:

- n8n 2.14.2+ の `workflow_history` published snapshot 問題と DB 直接更新
- Docker から Windows ネイティブ FFmpeg を呼ぶブリッジサーバー設計
- 重複防止 4 層ガードの設計思想と実装
- VOICEBOX 誤読対策の読み辞書管理
- Sonnet 創作タスクで多様性を確保する温度設定とランダム制約

順次書いていきます。

---

新記事の告知は X([@AI_makase5](https://x.com/AI_makase5)) で行っています。
記事への感想・誤りの指摘・「次に書いてほしい派生テーマ」のリクエストは X DM までどうぞ。
