# Obsidian Web Clipper コンテンツ抽出ロジック 詳細分析レポート

> 調査対象コミット: `265068ebe5b7ceaefba7066b8fa9856706cb59a5`（branch: `copilot/main`）  
> 調査日: 2026-04-23

---

## 1. 概要

Obsidian Web Clipper は、ブラウザ拡張（Chrome / Firefox / Safari）として Web ページを Obsidian ノートへ一発で保存するツールである。コアとなる機能は **「任意の Web ページから本文・メタデータを抽出し、ユーザー定義テンプレートへ差し込んで Markdown ファイルを生成する」** ことにある。

コンテンツ抽出機能は拡張機能の中核であり、次の 3 つの文脈で実行される。

| 文脈 | エントリポイント | 実行場所 |
|------|----------------|---------|
| ブラウザ拡張（通常） | `src/content.ts` → `getPageContent` ハンドラ | コンテントスクリプト（ページ上） |
| クリップボード/ファイル保存 | `src/content.ts` → `copyMarkdownToClipboard` / `saveMarkdownToFile` | コンテントスクリプト |
| CLI / API | `src/cli.ts`, `src/api.ts` | Node.js プロセス（linkedom） |

---

## 2. 技術スタック一覧

### 2.1 本番依存ライブラリ（`dependencies`）

| ライブラリ | バージョン | 役割 |
|-----------|-----------|------|
| **defuddle** | `^0.18.1` | Web ページ本文抽出・メタデータ取得・HTML → Markdown 変換のコア |
| **dompurify** | `^3.0.9` | HTML サニタイズ（XSS 防止） |
| **linkedom** | `^0.18.0` | CLI / Node.js 環境での DOM 実装 |
| **dayjs** | `^1.11.13` | 日付フォーマット（テンプレート変数 `{{date}}` 等） |
| **lz-string** | `^1.5.0` | ハイライトデータのローカルストレージ圧縮保存 |
| **highlight.js** | `^11.11.1` | リーダーモードでのコードハイライト |
| **lucide** | `^0.359.0` | UI アイコン |

### 2.2 主要ビルド/開発依存

| ツール | 役割 |
|--------|------|
| **webpack 5** | バンドラ（マルチブラウザ対応、`webpack.config.js`） |
| **TypeScript 5** | 型安全な実装 |
| **vitest** | ユニットテスト（フィルタ・パーサ・レンダラ等） |
| **esbuild** | CLI/API ビルド用高速バンドラ |
| **sass** | スタイル（`src/*.scss`） |
| **webextension-polyfill** | Firefox / Safari との API 差異吸収 |

### 2.3 defuddle 内部依存

defuddle は単一パッケージとして提供されるが、内部で次のライブラリを使用する。

| ライブラリ | 役割 |
|-----------|------|
| **Turndown** | HTML → GitHub Flavored Markdown 変換ベース |
| （Readability 非使用） | 独自スコアリング・セレクタリストで代替 |

---

## 3. 抽出パイプライン フロー

### 3.1 ブラウザ拡張（`getPageContent` リクエスト）

```
ユーザーが拡張アイコンをクリック
  │
  ▼
background.ts → sendMessageToTab("getPageContent")
  │
  ▼
[コンテントスクリプト: src/content.ts]
  │
  ├─1─ flattenShadowDom(document)          ←── Shadow DOM を data 属性にシリアライズ
  │       (最大3秒タイムアウト付き)
  │
  ├─2─ window.getSelection()               ←── ユーザー選択範囲の HTML を取得
  │
  ├─3─ new Defuddle(document, {url})       ←── defuddle インスタンス生成
  │       .parseAsync()                    ←── 非同期パース（最大8秒タイムアウト）
  │       ↓ fallback
  │       .parse()                         ←── タイムアウト時は同期パース
  │
  ├─4─ URLの相対→絶対変換                  ←── src/href/srcset を全件処理
  │
  ├─5─ script/style 要素の除去             ←── fullHtml の前処理
  │
  └─6─ sendResponse({content, title, ...}) ←── popup/side-panel へ返却
```

### 3.2 defuddle 内部パイプライン（`parseInternal()`）

```
parse()
  │
  ├─A─ _normalizeAttributes()              ←── React SSR の srcSet → srcset 正規化
  ├─B─ _resolveNoscriptImages()            ←── Next.js base64 プレースホルダ → 実 URL
  │
  ├─C─ [初回] parseInternal(デフォルト設定)
  │      │
  │      ├─ MetadataExtractor.extract()    ←── title/author/published 等取得
  │      ├─ ExtractorRegistry.findExtractor()  ←── サイト別特殊エクストラクタ探索
  │      │    (YouTube/GitHub/Reddit/Medium/NYT 等 20+ サイト対応)
  │      │
  │      ├─ _evaluateMediaQueries()        ←── モバイルCSSの max-width ルール評価
  │      ├─ applyMobileStyles()            ←── モバイルスタイルをクローン DOM に適用
  │      │
  │      ├─ removeHiddenElements()         ←── display:none / visibility:hidden / Tailwind hidden
  │      ├─ removeBySelector()             ←── EXACT_SELECTORS + PARTIAL_SELECTORS で除去
  │      ├─ removeByContentPattern()       ←── ニュースレター/SNS共有/関連記事ヒューリスティック
  │      ├─ removeEyebrowLabel()           ←── カテゴリラベル/アイキャッチ除去
  │      ├─ ContentScorer.score()          ←── 各ブロック要素をスコアリング
  │      ├─ standardizeContent()           ←── math/code/見出し/画像を正規化
  │      ├─ _deduplicateImages()           ←── 重複画像除去（srcset / noscript 由来）
  │      └─ removeOrphanedDividers()       ←── 孤立した <hr> 除去
  │
  ├─D─ [ワードカウント < 200] リトライ (removePartialSelectors=false)
  ├─E─ [ワードカウント < 50]  リトライ (removeHiddenElements=false)
  ├─F─ [ワードカウント < 50]  リトライ (removeLowScoring=false, インデックスページ想定)
  │
  ├─G─ _stripUnsafeElements()             ←── script/style/event handler 属性を最終除去
  │
  └─H─ schema.org テキストとの照合・最良コンテンツ候補の再選択
```

### 3.3 Markdown 変換 → テンプレート適用

```
defuddled.content (クリーン HTML)
  │
  ├─ processHighlights()                   ←── <mark> 要素のインジェクション
  │
  ├─ createMarkdownContent(content, url)   ←── Turndown + カスタムルール 15+ 個
  │    (defuddle/full から import)
  │
  └─ initializePageContent()
       │
       ├─ buildVariables()                  ←── {{title}}, {{content}}, {{date}} 等を生成
       ├─ addSchemaOrgDataToVariables()      ←── {{schema:@Article:headline}} 等を生成
       │
       └─ compileTemplate()
            ├─ tokenize() → parse() → render()   ←── AST ベースのテンプレートエンジン
            └─ processVariables()                  ←── selector:/schema:/prompt: 特殊変数処理
```

---

## 4. 各ステージの詳細解説

### 4.1 取得 — Shadow DOM の展開

Shadow DOM を持つページ（Web Components 等）では、コンテントスクリプトからは `shadowRoot` の中身が読めない。そのため、**メインワールドに別スクリプトを注入**して shadow innerHTML を `data-defuddle-shadow` 属性に書き出す。

- **ファイル**: [`src/flatten-shadow-dom.js`](https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/flatten-shadow-dom.js)  
  ```js
  // L4-9
  document.querySelectorAll('*').forEach(function(el) {
      if (el.shadowRoot && el.shadowRoot.innerHTML) {
          el.setAttribute('data-defuddle-shadow', el.shadowRoot.innerHTML);
      }
  });
  ```
- **呼び出し側**: [`src/utils/flatten-shadow-dom.ts`](https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/utils/flatten-shadow-dom.ts) — Shadow DOM が存在する場合のみスクリプトをインジェクト（L3-27）
- **タイムアウト**: [`src/content.ts`](https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/content.ts#L200-L201) で 3 秒タイムアウト付き `Promise.race()` でラップ

### 4.2 前処理 — URL 絶対化・クリーンアップ

コンテントスクリプト内で `fullHtml` を生成する前に、相対 URL を絶対 URL に変換する。

- **ファイル**: [`src/content.ts`](https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/content.ts#L237-L262)
  - `src` / `href` / `srcset` の全属性を `new URL(value, document.baseURI).href` で変換
  - `srcset` はカンマ区切りを個別解析し、幅記述子（`400w` 等）を保持
  - `data:` / `#` / `//` から始まる値はスキップ
- **script / style 要素の除去**: L231-234

defuddle 内部では `_normalizeAttributes()` が React SSR の `srcSet` → `srcset` などの属性名正規化を行い、`_resolveNoscriptImages()` が Next.js の `<noscript>` 内の実画像 URL を base64 プレースホルダと差し替える（`node_modules/defuddle/dist/defuddle.js` L54-55, L359 周辺）。

### 4.3 本文抽出 — Defuddle スコアリング

Defuddle は **Mozilla Readability のフォークではなく**、独自アルゴリズムを採用する。主な戦略：

#### (a) EXACT_SELECTORS による確定的除去

[`node_modules/defuddle/dist/constants.js`](https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/node_modules/defuddle/dist/constants.js) の `EXACT_SELECTORS` には **広告・ナビゲーション・ヘッダ・フッタ・コメント欄** に相当する CSS セレクタが 70+ 個定義されている（例: `'.ad'`, `'nav'`, `'[role="navigation"]'`, `'[id="comments"]'`, `'[role="dialog"]'`, `'.menu'` 等）。

除去時は「footnote コンテナの祖先・子孫」「heading 内の anchor link」「code/pre ブロック内の要素」を保護する例外処理が施されている（`node_modules/defuddle/dist/removals/selectors.js` L79-108）。

#### (b) PARTIAL_SELECTORS による正規表現マッチ

`PARTIAL_SELECTORS` には 300+ の部分文字列パターンが列挙されており、クラス名・ID・`data-*` 属性に対してまとめて正規表現マッチを行う。検査対象属性は 8 種類（`class`, `id`, `data-component`, `data-test`, `data-testid`, `data-test-id`, `data-qa`, `data-cy`）。

**見出し要素（H1-H6）は `class` のみを検査**し、自動生成スラグが ID に入る場合の誤検知を防止（`node_modules/defuddle/dist/removals/selectors.js` L56-65）。

**code/pre ブロック内部は言語名クラスが誤検知を起こすため除外**（L46-53）。

#### (c) ContentScorer によるスコアリング

[`node_modules/defuddle/dist/removals/scoring.js`](https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/node_modules/defuddle/dist/removals/scoring.js) のスコア計算（`scoreElement()` 関数）：

| 要素 | スコア加減 |
|------|----------|
| テキスト語数 | +語数 |
| `<p>` タグ数 | ×10 |
| カンマ数（散文度） | +カンマ数 |
| 画像密度（img/words） | −密度×3 |
| 日付・著者パターン一致 | +10 |
| content/article/post クラス | +ボーナス |
| advertisement/ad/nav クラス | −ペナルティ |
| ナビゲーション系テキスト（"follow us", "subscribe" 等） | 検出時に除去 |

#### (d) モバイル CSS を利用したノイズ除去

`_evaluateMediaQueries()` で `max-width` メディアクエリのスタイルルールを収集し、`applyMobileStyles()` でクローン DOM に適用する。これにより、「モバイルで非表示になる広告・サイドバー」を `display:none` 検出で除去できる（`defuddle.js` L894-976）。

### 4.4 サニタイズ

defuddle の `_stripUnsafeElements()` が `script / style / noscript / frame / object / embed / applet / base` 要素を除去し、全要素の `on*` イベントハンドラ属性、危険な URI（`javascript:` 等）、`srcdoc` 属性を削除する（`defuddle.js` L178-230）。

iframe は **Same-Origin Policy で分離されているため** 削除しない（コメント明記）。

### 4.5 Markdown 変換 — Turndown カスタムルール

`createMarkdownContent()` ([`node_modules/defuddle/dist/markdown.js`](https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/node_modules/defuddle/dist/markdown.js)) は Turndown に 15 個以上のカスタムルールを追加する：

| ルール名 | 対象 | 変換内容 |
|---------|------|---------|
| `table` | `<table>` | GFM テーブル / レイアウトテーブル判定・ArXiv 数式テーブル対応 |
| `list` / `listItem` | `<ul>/<ol>/<li>` | タブインデントネスト・タスクリスト（`[ ]`/`[x]`）対応 |
| `figure` | `<figure>` | figcaption を画像 alt/キャプションとして保持 |
| `image` | `<img>` | srcset から最高解像度 URL を選択 |
| `embedToMarkdown` | `<iframe>` | YouTube/Vimeo/その他 embed をマークダウンリンクに変換 |
| `highlight` | `<mark>` | `==テキスト==`（Obsidian ハイライト記法） |
| `strikethrough` | `<del>/<s>/<strike>` | `~~テキスト~~` |
| `preformattedCode` | `<pre><code>` | バッククォートフェンス + 言語タグ（`data-lang`, `class="language-*"` を検出） |
| `math` | `<math>` | MathML → LaTeX（`$...$` / `$$...$$`）、インライン/ブロック自動判定 |
| `katex` | `.math/.katex` | `data-latex` → `$...$` / `$$...$$` |
| `citations` | `<sup id="fnref:*">` | `[^id]` 形式の脚注参照 |
| `footnotesList` | `<ol>` in `#footnotes` | `[^id]: 内容` 形式の脚注定義 |
| `arXivEnumerate` | `<ol.ltx_enumerate>` | ArXiv 番号付きリスト正規化 |
| `complexLinkStructure` | 見出しを含む `<a>` | 見出し＋本文＋元URLリンクに分離 |
| `removals` | backref リンク | 脚注バックリンクを除去 |

### 4.6 テンプレート適用

抽出データは `buildVariables()` ([`src/utils/shared.ts`](https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/utils/shared.ts#L40-L93)) が辞書化し、`compileTemplate()` ([`src/utils/template-compiler.ts`](https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/utils/template-compiler.ts)) が適用する。

テンプレートエンジンは独自実装の AST ベースレンダラ（`tokenizer.ts` → `parser.ts` → `renderer.ts`）で、`if/elseif/else/endif`、`for/endfor`、`set` 構文、50+ 個のパイプフィルタ（`|date:`, `|replace:`, `|strip_tags` 等）をサポートする。

---

## 5. 「よくできている」と評価できる工夫点

### 工夫 1: 段階的フォールバックを持つ適応型リトライ

**ファイル**: [`node_modules/defuddle/dist/defuddle.js`](https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/node_modules/defuddle/dist/defuddle.js) — `parse()` メソッド L58-135

ワードカウントをしきい値として 4 段階のリトライ戦略を実装している：

1. **通常パース**（デフォルト設定）
2. **ワード数 < 200** → 部分セレクタ除去を無効化してリトライ（除去過剰の可能性）
3. **ワード数 < 50** → 隠し要素除去を無効化してリトライ（動的コンテンツが hidden で提供される場合）
4. **ワード数 < 50** → スコアリングと部分セレクタを完全無効化してリトライ（インデックスページ）

加えて、schema.org の本文テキストと抽出結果を比較し、schema.org が 1.5 倍以上多い場合は DOM から該当要素を探して再抽出する。これにより、フィードページやリスティングページでの誤抽出を自動補正できる。

---

### 工夫 2: モバイル CSS を利用した広告/サイドバー除去

**ファイル**: `defuddle.js` — `_evaluateMediaQueries()` (L894-962) / `applyMobileStyles()` (L964-976)

ページ自身のスタイルシートから `max-width: ≤767px` 以下のメディアクエリルールを収集し、DOM クローンに適用する。これにより、サイト自身がモバイルで非表示にしている要素（広告バナー、横断ナビ等）を `display:none` 検出で自動除去できる。**外部の広告セレクタリストに依存しない、サイト固有ロジックを活用した賢い手法**である。

---

### 工夫 3: コードブロック内を誤検知から保護する例外処理

**ファイル**: `node_modules/defuddle/dist/removals/selectors.js` — L44-53

```js
if (tag === 'CODE' || tag === 'PRE' || el.querySelector('pre') || el.closest('code, pre')) {
    return;
}
```

部分セレクタのマッチング処理でコードブロック内の要素をスキップする。例えばシンタックスハイライトのクラス名に `ad-`, `nav-`, `comment` などが含まれる場合の誤除去を防止している。同様に EXACT_SELECTORS でも `el.closest('pre, code')` チェックが入っている（L26-29）。

---

### 工夫 4: Shadow DOM を main world スクリプトインジェクションで展開

**ファイル**: [`src/flatten-shadow-dom.js`](https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/flatten-shadow-dom.js) L1-10 / [`src/utils/flatten-shadow-dom.ts`](https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/utils/flatten-shadow-dom.ts) L3-27

コンテントスクリプト（isolated world）からは `shadowRoot` の中身にアクセスできないという Chrome の制限を、**メインワールドに別スクリプトをインジェクト**し `data-defuddle-shadow` 属性に innerHTML を書き出すという迂回策で解決している。Shadow DOM が存在しない場合は即座に `Promise.resolve()` を返す早期リターンで不要なオーバーヘッドを排除している。

---

### 工夫 5: Next.js 等の lazy-load 画像を noscript から救出

**ファイル**: `defuddle.js` — `_resolveNoscriptImages()` (L359 周辺)

Next.js の `<Image>` コンポーネントは JavaScript が有効なブラウザ向けに base64 のプレースホルダ gif を `src` に設定し、実画像 URL を `<noscript>` タグ内にのみ記述する。Defuddle は `data-nimg` 属性を持つ `<img>` を検出し、隣接する `<noscript>` 内の実 URL を昇格させてから noscript を削除する。これにより、動的レンダリングに依存した画像も正しく取得できる。

---

### 工夫 6: スコアリング中の複合ヒューリスティック（散文検出）

**ファイル**: `node_modules/defuddle/dist/removals/scoring.js` — `scoreElement()` 関数

散文コンテンツの特徴として **カンマの頻度**（`score += commas`）を加点要素に採用している。ナビゲーションリンクや箇条書きにはカンマが少なく、記事本文にはカンマが多いという経験則を利用した軽量な手法である。さらに `<p>` タグ密度（×10 加点）、日付・著者のパターンマッチ（+10）、クラス名ベースのコンテンツ/非コンテンツ識別を組み合わせてスコアを計算する。

---

### 工夫 7: サイト別特殊エクストラクタ（20+ サイト対応）

**ファイル**: `node_modules/defuddle/dist/extractors/` 配下（youtube.js, github.js, reddit.js, medium.js, nytimes.js, substack.js 等）

YouTube には InnerTube API（非公式）を利用したトランスクリプト取得、GitHub には Issues/PR のスレッド構造を再現する HTML 生成、Wikipedia には脚注・数式の特殊処理など、サイト固有の DOM 構造に合わせたエクストラクタが 20 社以上分実装されている。`ExtractorRegistry.findPreferredAsyncExtractor()` が URL と schema.org データを照合して最適なエクストラクタを選択する。

YouTube エクストラクタは CJK 文字（`\u3002\uFF01\uFF1F`）を意識した文分割ロジックも持ち、日本語・中国語・韓国語のトランスクリプトにも対応している（`extractors/youtube.js` 冒頭定数定義）。

---

### 工夫 8: Turndown の srcset 最高解像度選択

**ファイル**: `node_modules/defuddle/dist/markdown.js` — `getBestImageSrc()` 関数 (L15-50)

`srcset` のトークン解析で、**CDN URL 内のカンマ（例: `w_424,c_limit,f_webp`）と幅記述子のカンマを区別**するため、カンマ分割ではなくホワイトスペースによるトークン化を採用している。`Nw` パターンに一致するトークンを幅として認識し、最大幅の URL を `src` として採用する。

---

### 工夫 9: コンテントスクリプトのゾンビプロセス対策

**ファイル**: [`src/content.ts`](https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/content.ts#L23-L29) / [`src/utils/content-extractor.ts`](https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/utils/content-extractor.ts#L104-L125)

拡張機能更新後に古いコンテントスクリプトが残存（ゾンビ化）する問題に対し、**generation カウンタ**（`window.obsidianClipperGeneration`）を採用している。新しいスクリプトが注入されるたびにカウンタをインクリメントし、古い世代のリスナーは自分より新しい世代を検出するとメッセージを無視して黙って退場する。さらに `extractPageContent()` では 1 回目の失敗時に `forceInjectContentScript` を送って強制再注入し、2 回目もリトライする二段構えのエラーリカバリを実装している。

---

### 工夫 10: テンプレートエンジンのエラー耐性設計

**ファイル**: [`src/utils/template-compiler.ts`](https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/utils/template-compiler.ts#L60-L73)

テンプレートのコンパイルエラーが発生しても出力を中断せず、`console.error` でログを出した上で **部分的な出力を返す**。`hasDeferredVariables` フラグによる最適化（特殊変数が存在しない場合は後処理正規表現を省略）も実装されており、シンプルなテンプレートでは不要なパースコストを回避する。

---

## 6. 既知の制約・改善余地

### 6.1 ページネーション未対応

複数ページに分割された記事は、ユーザーが閲覧中のページのみ抽出される。ページ間をまたいだ本文結合の仕組みはない。

### 6.2 JavaScript レンダリング後の動的コンテンツ

コンテントスクリプトはページロード済みの DOM に対して動作するため、ユーザー操作（クリック・スクロール）によって後から挿入されるコンテンツは取得できない。`flattenShadowDom()` のタイムアウト 3 秒も、重い SPA では不足する場合がある。

### 6.3 PARTIAL_SELECTORS の誤検知リスク

300+ のパターンを正規表現でまとめてマッチするため、独自クラス名を持つ正当なコンテンツ要素が誤除去されるリスクがある。例えば `byline`, `author-bio`, `avatar` などは記事本文に登場することもある。欠損コンテンツ発生時に `removePartialSelectors=false` でリトライする仕組みはあるが、ユーザーには透明でない。

### 6.4 MathML → LaTeX 変換の精度

ArXiv や Wikipedia の数式は `extractLatex()` 関数で変換されるが、複雑な MathML（多段ネスト、カスタムオペレータ等）では LaTeX への逆変換が不完全になる可能性がある。

### 6.5 CLI/Node.js モードの非同期エクストラクタ制限

YouTube トランスクリプト等の `extractAsync()` は fetch を必要とするが、Node.js 環境では `options.fetch` を明示的に渡す必要がある。デフォルトでは同期 `extract()` にフォールバックするため、CLI からのトランスクリプト取得は制約がある。

---

## 7. 参考リンク

| リソース | URL |
|---------|-----|
| リポジトリ (ogra fork) | https://github.com/ogra/obsidian-clipper |
| 公式リポジトリ | https://github.com/obsidianmd/obsidian-clipper |
| defuddle npm | https://www.npmjs.com/package/defuddle |
| Turndown | https://github.com/mixmark-io/turndown |
| Mozilla Readability（比較対象） | https://github.com/mozilla/readability |
| DOMPurify | https://github.com/cure53/DOMPurify |
| linkedom | https://github.com/WebReflection/linkedom |
| webextension-polyfill | https://github.com/mozilla/webextension-polyfill |

### 主要コードパーマリンク

| 説明 | リンク |
|------|--------|
| Shadow DOM 展開スクリプト | https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/flatten-shadow-dom.js |
| コンテントスクリプト getPageContent | https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/content.ts#L198-L296 |
| 抽出エラーリカバリ (generation カウンタ) | https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/content.ts#L23-L29 |
| URL 絶対化処理 | https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/content.ts#L237-L262 |
| extractPageContent フォールバック | https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/utils/content-extractor.ts#L104-L125 |
| ハイライト処理 (processHighlights) | https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/utils/content-extractor.ts#L204-L249 |
| テンプレート変数辞書構築 | https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/utils/shared.ts#L40-L93 |
| テンプレートコンパイラ | https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/utils/template-compiler.ts |
| schema.org 変数展開 | https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/utils/shared.ts#L99-L135 |
| CSS セレクタ抽出 | https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/utils/shared.ts#L257-L280 |
| フィルタ一覧 (filters.ts) | https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/utils/filters.ts |
| clip-utils (reader mode 対応) | https://github.com/ogra/obsidian-clipper/blob/265068ebe5b7ceaefba7066b8fa9856706cb59a5/src/utils/clip-utils.ts |
