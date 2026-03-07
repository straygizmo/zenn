---
title: "Markdown Studioで「Vibe Zenning」！音声入力とAIで爆速記事作成"
emoji: "🚀"
type: "tech"
topics: ["markdownstudio", "zenn", "writing", "moonshine", "ai"]
published: false
---
「Vibe Zenning」

昨今の「Vibe Coding」「Vibe Working」に倣って、「Vibe Zenning」があってもいいよね。ね！？
というわけで、@akinobukatoさんの[記事](https://zenn.dev/acntechjp/articles/5a8b7b334b15bc)に感動し、Markdown Studioを魔改造させていただきました。
オリジナルのMarkdown Studio
https://github.com/5843435/markdown-sheet

Markdown StudioをZennの爆速記事エディターとして活用する方法をご紹介します。
今回は、ローカル環境で動作する音声認識モデル「[Moonshine](https://github.com/moonshine-ai/moonshine)」を利用した音声入力と、AIによる自動整形を組み合わせた、まさに「Vibe」で記事を書くスタイルを解説します。

:::message
音声入力には、最近話題の高速・高精度な音声認識モデル **Moonshine** を使用しています。ゆっくり喋れば、ローカル環境でもかなりの精度で書き起こしが可能です。
:::

## 1. セットアップと設定

まずは[改造版のMarkdown Studio](https://github.com/straygizmo/markdown-sheet)をセットアップし、Zennモードを使えるように設定しましょう。

### モデルとAPIの設定
右上の設定ボタン（歯車アイコン）をクリックします。

![設定画面を開く](../images/2026-03-07_113824.png)

APIキーや使用するモデル（Moonshineなど）の設定を行います。

![モデルの設定](../images/2026-03-07_113920.png)

### Zennボタンの有効化
「表示」タブから「Zennボタンを表示」にチェックを入れます。これでZenn専用の機能が使えるようになります。

![Zennボタンの表示](../images/2026-03-07_114025.png)

## 2. Zennプロジェクトの作成

画面左上のフォルダ開くボタンをクリックし、記事を管理したいディレクトリを選択します。

![フォルダを開く](../images/2026-03-07_114212.png)

サイドバーに表示される「Zennボタン」をクリックします。

![Zennボタンをクリック](../images/2026-03-07_114325.png)

「Zennプロジェクトの初期化」ダイアログが表示されるので、OKを押して初期化を実行します。

![プロジェクトの初期化](../images/2026-03-07_115336.png)

## 3. 記事の新規作成

フォルダがZennモードになったら、画面右上の「記事の新規作成」ボタンを押します。

![新規作成ボタン](../images/2026-03-07_115742.png)

ファイル名と記事タイトルを入力して、作成ボタンを押します。

![ファイル名とタイトルの決定](../images/2026-03-07_120236.png)

これで、Zennのフロントマター（YAML）が含まれた新しいMarkdownファイルが作成されます。

![エディタ画面](../images/2026-03-07_120435.png)

## 4. 音声入力と執筆

### 音声認識の開始
エディターにカーソルを合わせ、`Ctrl + Space` を押すと音声認識が開始されます。

![音声認識の開始](../images/2026-03-07_123816.png)

あとはマイクに向かって喋るだけです。思考をそのまま言葉にして流し込んでいきましょう。

![音声入力中の様子](../images/2026-03-07_123622.png)

### 画像の挿入
画像ファイルは、あらかじめ `images` フォルダに入れておきます。そこからMarkdownエディタへドラッグ＆ドロップするだけで、簡単にパスを挿入できます。

![画像のドラッグ＆ドロップ](../images/2026-03-07_133008.png)

## 5. AIによる仕上げ

一通り内容を入力し終えたら、最後にAIへ「記事の体裁を整えて」と頼みます。誤字脱字の修正や、構成の整理を自動で行ってくれます。

![AIに依頼](../images/2026-03-07_133254.png)

Zenn専用の内部システムプロンプトで、雑に入力した元の内容（下記参照）が体裁の整った記事になりました。
```
const systemPrompt =
          "あなたはZenn (zenn.dev) 記事のMarkdown編集アシスタントです。ユーザーの指示に従って、与えられたMarkdown本文を編集・加筆・修正してください。\n" +
          "結果はMarkdown本文のみを返してください。説明やコードブロック囲みは不要です。\n\n" +
          "## Zenn記事のルール\n" +
          "- フロントマター(---で囲まれたYAML部分)は必須です。必ずそのまま残してください。指示がある場合のみ編集してください。\n" +
          "  フロントマターの形式:\n" +
          "  ---\n" +
          "  title: \"記事タイトル\"\n" +
          "  emoji: \"🚀\"\n" +
          "  type: \"tech\"  # tech or idea\n" +
          "  topics: [\"python\", \"automation\"]\n" +
          "  published: true  # false = 下書き\n" +
          "  ---\n\n" +
          "## Zenn独自の記法（積極的に活用してください）\n" +
          "- メッセージボックス: :::message ... ::: または :::message alert ... :::\n" +
          "- アコーディオン: :::details タイトル ... :::\n" +
          "- コードブロックにファイル名: ```python:main.py\n" +
          "- diffハイライト: ```diff python\n" +
          "- 数式(KaTeX): $ E = mc^2 $\n" +
          "- Mermaid図: ```mermaid ... ```\n" +
          "- リンクカード: URLを単独の行に置くだけで自動展開\n" +
          "- 画像サイズ指定: ![alt](url =250x) — =〇〇x でpx幅指定\n";
```

:::message success
音声入力で「Vibe（雰囲気）」を伝え、AIで「Tech記事」として完成させる。これがMarkdown Studio流の爆速執筆術です！
:::
---