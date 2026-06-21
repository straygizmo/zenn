---
title: "opencode プライバシー監査レポート — チャット履歴の外部送信について"
emoji: "🛡️"
type: "tech"
topics: ["opencode", "privacy", "security", "audit"]
published: false
---

「opencode? 信用できるの？」みたいな話になったので、これで良いのかと思いながらも以下の手順・条件でClaude Codeに調べてもらいました。

＜手順＞
1. opencodeのリポジトリをクローン
1. クローンしたディレクトリでClaude Codeを起動
1. プロンプトを入力

| 項目 | 条件 |
| --- | --- |
| 実施日 | 2026/6/17 |
| 監査対象 | opencode 1.17.7 |
| 監査者 | Claude Code Opus 4.8 |
| プロンプト | opencodeがAPI設定で指定されたプロバイダ以外にチャット履歴等を外部に送信していない確証を得たいです。監査してもらえますか？ |

![](/images/image-20260617-230731.png)

![](/images/image-20260617-233712.png)

結果、問題ないようでした。
あくまでもデフォルトのツールが監査対象ですので、独自ツールやMCP等を追加した場合は自己責任です。
Claude CodeがMD形式で出力してくれたレポートを以下に貼っておきます。

---

# opencode プライバシー監査レポート — チャット履歴の外部送信について

> 監査日: 2026-06-17
> 対象: opencode CLI / エージェント（クライアントで実行されるコード）
> 目的: チャット履歴・会話内容が「API設定で指定したLLMプロバイダ」以外に送信されていないことの確認

## 結論

**デフォルト状態では、会話内容（チャット履歴・プロンプト・ファイル内容・ツール出力）が「設定済みLLMプロバイダ」以外に送信される経路はありません。** 起動・通常利用だけでセッションデータが外部に出ることはなく、監査範囲のクライアントコードに隠れたゲートウェイ・テレメトリ・解析SDK・クラッシュレポーターは存在しませんでした。

ただし、**ユーザーが明示的に有効化（opt-in）またはツールを起動した場合に限り**会話由来のデータが外部へ出る経路がいくつかあります。これらは設計上の機能であり、いずれも無効化・回避が可能です。以下に全送信先を分類して報告します。

---

## A. 会話内容を運ぶ経路（要注意）

| # | 送信先 | トリガー | 送信内容 | デフォルト | 無効化方法 |
|---|--------|---------|---------|-----------|-----------|
| 1 | **設定済みプロバイダ**（api.anthropic.com 等、または config の `baseURL`） | 推論リクエスト | 会話全体＋ファイル（＝本来の宛先） | — | — |
| 2 | **chatgpt.com** | ChatGPTサブスク(OAuth)で推論する場合 | 会話全体＋session-idヘッダ | その経路を選んだ時のみ | api.openai.com系を使う |
| 3 | **OpenCode Zen** (`opencode.ai/zen/v1`) | `opencode/*` モデルを**明示選択**した時のみ | 会話全体（opencode運営インフラ） | **opt-in** | 当該モデルを選ばない |
| 4 | **opncd.ai**（または `enterprise.url`／ログイン先） | **share機能** | **セッション全文：全メッセージ・ツール入出力・ファイル差分・モデル情報** | **OFF（opt-in）** | 後述（最重要） |
| 5 | **mcp.exa.ai / search.parallel.ai** | 組込 `websearch` ツールをモデルが起動 | 検索クエリ（会話由来の文字列） | ツール起動時のみ | websearchツールを無効化 |
| 6 | **任意のURL** | 組込 `webfetch` ツール（モデルが選択、許可制） | モデルが指定したURL先 | ツール起動時のみ | permission/ツール無効化 |
| 7 | **OTLPエンドポイント** | OpenTelemetry有効化時 | プロンプト/応答が含まれ得る | **二重opt-in** | 既定で無効。下記注意 |
| 8 | social-cards.sst.dev | `opencode github` Action（CI） | セッションタイトル(base64)を画像URLに埋込 | CI利用時のみ | — |

### 最重要：share機能（#4）

会話全文をopencodeのサーバ（既定 `opncd.ai`）へアップロードする唯一の経路です。**完全にopt-in**で、`/share` 実行・`share: "auto"`・`OPENCODE_AUTO_SHARE`・`--share`・GitHub連携のいずれかが必要。これらをしなければ、イベント監視は動いていても `if (!share) return` で全て短絡し、**何も送信されません**（`packages/opencode/src/share/share-next.ts:127`）。

- 確実に封じるには config に `"share": "disabled"`（試行時に例外で停止）、または環境変数 `OPENCODE_DISABLE_SHARE=1`（モジュール全体のキルスイッチ）。
- 該当コード: `packages/opencode/src/share/share-next.ts:23,127,179-200,210,259,314,350`、`packages/opencode/src/share/session.ts:28,43`、`packages/core/src/config.ts:46`

### OpenTelemetry（#7）の注意点

既定で完全に無効（`OTEL_EXPORTER_OTLP_ENDPOINT` 未設定なら何も出ない）。**かつ** config の `experimental.openTelemetry` が必要。ただし**有効化した場合**、コードが `recordInputs/recordOutputs: false` を設定していないため、AI SDK標準動作でプロンプト・応答本文がスパンに含まれます。送信先は**ユーザー自身が指定したエンドポイントのみ**で、opencode運営の既定コレクタは存在しません。

- 該当コード: `packages/core/src/observability/otlp.ts:7,51,56`、`packages/opencode/src/session/llm.ts:208-222,344-352`、`packages/core/src/flag/flag.ts:16`

---

## B. 会話内容を含まない経路（メタデータ・認証・ダウンロードのみ）

- **models.dev** — モデルカタログのGETのみ（`OPENCODE_MODELS_URL` で変更／`OPENCODE_DISABLE_MODELS_FETCH` で無効化）。`packages/core/src/models-dev.ts:142-164`
- **console.opencode.ai 等** — `opencode account login` のOAuthデバイスフロー（トークン・org/userメタデータのみ）。`packages/opencode/src/account/account.ts:220,287,300,358,378,399`
- **更新チェック** — npm registry / api.github.com / brew / choco / scoop へバージョン文字列GET（User-Agentのみ送信、`autoupdate: false` または `OPENCODE_DISABLE_AUTOUPDATE=1` で無効化）。`packages/opencode/src/installation/index.ts:230-273`、`packages/opencode/src/cli/upgrade.ts:8-12`
- **各種OAuth** — auth.x.ai / auth.openai.com / github.com / digitalocean 等（認証コード・トークンのみ）
- **バイナリ／パッケージDL** — ripgrep, LSPサーバ（jetbrains/eclipse/hashicorp）, npm（いずれも会話内容なし）
- **app.opencode.ai** — Web UIアセットのフォールバック取得のみ。`packages/opencode/src/server/shared/ui.ts:9,89`

テレメトリ／解析SDK（PostHog/Sentry/Segment/Mixpanel等）は**CLIに一切バンドルされていません**。`stats`パッケージやPostHog参照は**サーバ／CI側のインフラ**で、ユーザーのマシンからは何も取得しません（`packages/stats/`、`script/stats.ts`、`.github/workflows/stats.yml`）。

---

## C. 監査の限界（コード静的解析で保証できない範囲）

1. **サードパーティプラグイン**（npm経由でロード）と **ユーザー設定のMCPサーバ** は任意の通信を開けます。これらはopencodeの信頼境界外（SECURITY.md にも明記）で、URL grepでは網羅不可。導入するプラグイン/MCPは個別に確認が必要です。
2. `webfetch`/`websearch` はエージェントが意図的に使うツールであり、送信先は設計上open-ended。
3. ネットワーク層で確実に保証したい場合、opencodeは `HTTP(S)_PROXY` を尊重するため、**全egressをプロキシ経由にしてホスト制限する**のが最も堅牢です（`packages/opencode/src/util/proxy-env.ts`）。

---

## D. 推奨：最も厳格な「プロバイダ以外には送らない」設定

```jsonc
// opencode config
{
  "share": "disabled",
  "autoupdate": false
}
```

```bash
# 環境変数
OPENCODE_DISABLE_SHARE=1
OPENCODE_DISABLE_AUTOUPDATE=1
OPENCODE_DISABLE_MODELS_FETCH=1   # カタログ取得も止める場合
# OTEL_EXPORTER_OTLP_ENDPOINT は設定しない
```

加えて、`opencode/*`（Zen）やChatGPT-OAuthモデルを選ばず自前のAPIキーでプロバイダを使い、`websearch`/`webfetch` を使わない（または許可制で都度承認する）、プラグイン/MCPを入れないこと。これで会話内容が出る宛先は**自分で設定したプロバイダのみ**に限定できます。

---

## 総合判定

「設定済みプロバイダ以外に勝手にチャット履歴を送る」挙動は確認されませんでした。会話内容が外部に出る他の経路はすべて opt-in 機能かエージェントが明示的に使うツールに限られ、上記設定で無効化できます。注意べきは以下3点：

1. share を誤って有効（`auto`）にしないこと
2. OTel 有効化時はプロンプトがスパンに含まれること
3. プラグイン/MCP は信頼境界外であること

---

## 付録: 監査方法

本監査は4つの並列サブエージェントによる静的コード解析で実施:

1. share / sync / control-plane / account の送信点
2. provider / llm の推論リクエスト経路
3. telemetry / update / stats / クラッシュレポート
4. クライアント実行コード全体の外部接続先ホスト網羅インベントリ

対象コミット: `dev` ブランチ HEAD（`10b6672be`）時点。
