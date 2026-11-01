# SNS-Resercher

MuseをSNSリサーチ専門家として活用するプロジェクト。

SNS（Bluesky / YouTube を起点に順次拡大）から一次寄りの情報を毎日コードで収集・蓄積し、
Muse が読んで判断材料に整えるための設計メモ。構築は後で行う（本書は設計の記録）。

> 記録：2026-11-01、Founder の指示により Muse（Meta Muse）が作成。
> 数値・可否はすべて同日の実地検証に基づく。未検証の推測はその旨を明記する。

## 目的と分担

- Cloudflare 側：集める・貯める・重複を消す（判断しない、確実に回る層）
- Muse 側：ダイジェストを読んで要約し、一次情報と突き合わせて判断材料にする
- SNS 情報はすべて「未検証」が前提。一次情報・公式情報で裏付けするまで、
  意思決定の前提値には入れない（AI-conversation リポジトリの運用ルールと同一）

## 現状（2026-11-01 検証済み）

Muse の VM 上にプロトタイプのコネクタが動作している（スクリプト実行は VM 側）。

### Bluesky

- アカウント `@kazalisa.bsky.social` を Founder 本人が作成済み。
  Muse のブラウザからは 1Password 経由のサインインが動作することを確認
- 公開 API（無認証）で取得できるもの：プロフィール、著者フィード、アカウント検索
- 投稿のキーワード検索（searchPosts）は無認証では 403。
  パスワード方式のセッション認証はコネクタの認証フローが受け付けないため、
  スクリプト用のアプリパスワード保管は行わない。
  認証付き検索が必要なときは、サインイン済みブラウザ経由で実行する
- フォロー一覧は匿名では取得不可（HTTP 501）。件数のみプロフィールで確認できる
- Jetstream（全投稿ライブストリーム）：接続は可能。ただし一般ストリームの
  キーワード密度が低い（60 秒・1,513 投稿の走査で GPU / H100 / HBM /
  半導体 / データセンター / capex / NVIDIA はいずれも 0 件。
  一般語は正常に拾えることを確認済み）。
  → 主経路はアカウントフィード、Jetstream は長時間蓄積用と位置づける

### YouTube

- API キーは使わない（Founder の指示。発行する場合は事前に確認する）
- チャンネル RSS で新着検出ができることを 4 チャンネルで確認済み
- 字幕（トランスクリプト）取得は yt-dlp で仕組みを用意したが、
  初回の取得リクエストが HTTP 429（レート制限）に当たった。
  クラウド IP からの字幕取得は制限されやすい。回避のための連打はしない

## 設計（Cloudflare で毎日処理する版）

### 構成

```
Cron Triggers（毎日 22:00 UTC = 07:00 JST）
  → Worker（収集ジョブ）
      ├─ Bluesky 公開 API：種アカウントのフィード取得
      └─ YouTube RSS：種チャンネルの新着取得
  → D1：items に URL 主キーで保存し、新着だけを残す
  → GET /digest（Bearer トークン必須）：その日の新着を JSON で返す
  → Muse が朝に読んで要約・突き合わせを行う
```

- この規模なら Workers / D1 ともに無料枠に収まる見込み
- コードは本リポジトリに置き、wrangler でデプロイする
- 失敗した取得元は runs テーブルに記録し、ダイジェストにも失敗として含める
  （一部成功を全体成功として報告しない）

### D1 スキーマ案

```sql
CREATE TABLE sources (
  id TEXT PRIMARY KEY,          -- 例: bsky:semiengineering.bsky.social
  kind TEXT NOT NULL,           -- 'bluesky' | 'youtube'
  label TEXT NOT NULL,
  ref TEXT NOT NULL,            -- handle または channel_id
  enabled INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE items (
  url TEXT PRIMARY KEY,         -- 投稿・動画の正規 URL（重複排除キー）
  source_id TEXT NOT NULL REFERENCES sources(id),
  published_at TEXT NOT NULL,   -- ISO 8601
  title TEXT,                   -- 動画タイトル等
  text TEXT,                    -- 投稿本文・概要
  fetched_at TEXT NOT NULL
);

CREATE TABLE runs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  started_at TEXT NOT NULL,
  source_id TEXT NOT NULL,
  status TEXT NOT NULL,         -- 'ok' | 'error' | 'rate_limited'
  new_items INTEGER NOT NULL DEFAULT 0,
  detail TEXT
);
```

### フェーズ分け

- **フェーズ 1**：Bluesky フィード ＋ YouTube RSS ＋ D1 ＋ /digest
- **フェーズ 2（保留）**：
  - Jetstream の常時キーワード監視。常時接続の受け手として Durable Objects が
    必要になる。密度が低いため費用対効果を見て判断する
  - YouTube 字幕：Workers では yt-dlp が動かないため、検出だけ Cloudflare で行い、
    字幕取得は Muse の VM 側で間隔を空けて行うハイブリッドにする

## 種リスト（2026-11-01 時点）

### Bluesky

| アカウント | 内容 | 状態 |
|---|---|---|
| @semiengineering.bsky.social | Semiconductor Engineering（業界誌） | 投稿内容まで確認済み。ほぼ毎日投稿 |
| @dylanonchips.bsky.social | CRN 半導体担当編集者 Dylan Martin | 投稿内容まで確認済み |
| @siliconomy.bsky.social | Semiconductor Business Intelligence | 存在のみ確認（フィード未精査） |

### YouTube

| チャンネル | channel_id | 内容・備考 |
|---|---|---|
| Asianometry | UCbGy2YOZaQ2ZqCkixSokTKQ | 半導体サプライチェーン分析 |
| ServeTheHome | UCv6J_jJa8GJqFwQNgNrMuww | AI サーバー・データセンター機材 |
| TechTechPotato | UCMMLsicGD2Zazw6xtPELWqg | 半導体技術解説（Ian Cutress）。更新活発 |
| GamersNexus | UCgGw5WSITWvDISNjydXPW3A | GPU 検証。YouTube 更新は疎（最新は 2026-07-28） |

決算期には通行料銘柄（NVIDIA / Microsoft / Alphabet / Amazon / TSMC /
ASML / Broadcom）の公式 IR チャンネル追加を検討する。

## ルール

- 読み取り専用。投稿・フォロー・いいね等の書き込みはしない
- YouTube API キーは Founder の確認なしに発行しない
- Bluesky のパスワード／アプリパスワードをファイル・メモリ・チャットに残さない
- レート制限（429 等）に当たったら、その試行では対象へのリクエストを止め、
  部分結果として正直に記録する

## 未決事項

- ダイジェストの受け渡しは pull 型（Muse が /digest を取りに行く）を想定。確定は構築時
- 巡回頻度は日次を初期値とする。増やすかどうかは運用を見て判断
- Jetstream を入れるかは保留（上記の密度の問題）
