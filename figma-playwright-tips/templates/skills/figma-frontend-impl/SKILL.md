---
name: figma-frontend-impl
description: >
  FigmaデザインをNext.js / React / CSSで忠実に実装し、Playwright MCPによる数値検証(getComputedStyle vs Figmaメタデータ)まで行うワークフロー。
  会話にFigmaのURL・フレーム・ノードIDが登場したとき、または「デザイン通りに実装」「デザインカンプを反映」「Figmaから実装」
  「ピクセルパーフェクト」「デザインとズレている・違う」「デザイン照合・検証して」といった依頼があったときは必ずこのスキルを使うこと。
  新規実装だけでなく、既存実装とFigmaデザインの差分調査・修正の依頼にも使う。
  Use this skill whenever a Figma URL appears or the user asks to implement, match, or verify a design against Figma.
---

# Figma → Next.js/React 実装+数値検証ワークフロー

Figma デザインを実装し、実装結果を Playwright で**数値レベルで**検証・自己修正するための手順。

中心となる原則: **ズレの検出は画像認識ではなく数値比較で行う。**
スクリーンショットの目測では「余白が少し広い気がする」までしか分からないが、
`getComputedStyle` の実測値と Figma メタデータの突き合わせなら「gap が 16px のはずが 12px」と確定でき、
確実に修正できる。スクリーンショットは工程の最初(リファレンス把握)と最後(全体の粗チェック)だけに使う。

## 前提チェック

開始前に確認し、欠けていればユーザーに伝える:

- Figma MCP のツール(`get_design_context` 等)が利用可能か
- Playwright MCP のツール(`browser_navigate` 等)が利用可能か
- dev server が起動しているか(していなければ起動するか、ユーザーに URL を確認)

## フェーズ 1: デザインコンテキストの取得

Figma URL から node-id を特定し、以下の順で取得する:

1. `get_design_context` — 対象ノードの構造化表現(React + Tailwind 形式)。
   これは**デザインの表現であって最終コードではない**。プロジェクトの規約に翻訳する前提で読む
2. レスポンスが大きい・途切れる場合: `get_metadata` で高レベルのノードマップを取得し、
   必要なノードだけ `get_design_context` で再取得する。
   大きな画面は Card / Header / Sidebar のようなコンポーネント単位に分割して扱う
   (一括で扱うとコンテキストが溢れ、精度が大きく落ちる)
3. `get_variable_defs` — 選択範囲で使われている変数(色・スペーシング・タイポ・radius)。
   トークンマッピングの原本になる
4. `get_screenshot` — 見た目のリファレンス。**ここから数値を目測しない**

このフェーズで「照合表の左列」を確定させる: フレーム幅、各要素の width/height、
padding、gap、font-size、line-height、font-weight、色、border-radius を数値でメモする。

### アセット

- Figma MCP が localhost ソースの画像/SVG を返したら、そのソースを直接使う
- 新しいアイコンパッケージを追加しない(アセットは Figma ペイロードに揃っている)
- プレースホルダーを作らない

## フェーズ 2: トークンマッピング

実装前に `get_variable_defs` の変数をプロジェクトの既存定義に対応付ける:

- 既存の CSS カスタムプロパティ(`globals.css` / `variables.css` 等)や
  デザイントークン定義を検索し、Figma 変数と同値のものがあればそれを使う
- 対応するトークンがない場合はハードコードせず、CSS カスタムプロパティとして追加してから使う
- 既存コンポーネント(Button, Input, Typography 等)を探し、再利用する。
  同等機能の重複実装はしない

なぜ重要か: トークン経由にしておけば色・スペーシングは構造的に「ズレようがない」状態になり、
検証フェーズの差分がレイアウト起因のものだけに絞れる。

## フェーズ 3: 実装

- Next.js / React のプロジェクト規約(App Router / Pages Router、CSS Modules / グローバル CSS)に従う
- Figma の Auto Layout は flexbox に対応させる(方向・gap・padding・align)
- 値はフェーズ 1 の数値とフェーズ 2 のトークンから取る。目測・推測の値を書かない
- レスポンシブ挙動が Figma に定義されていない場合は実装後にユーザーに確認事項として報告する

## フェーズ 4: Playwright による数値検証ループ

実装したら必ず検証する。ユーザーに「確認してください」と返す前に、自分で検証して直す。

1. `browser_navigate` で対象ページ(localhost)を開く
2. `browser_resize` で **Figma フレームと同じ幅**にする(高さはコンテンツに合わせてよい)
3. `browser_evaluate` で対象要素の `getComputedStyle` を取得する
   (具体的なスニペットと注意点は [references/verification.md](references/verification.md) を読む)
4. Figma の数値(フェーズ 1)と実測値の**突き合わせ表**を作る
5. 差分があれば修正し、2〜4 を繰り返す。1 回の修正ごとに再検証する
6. すべて一致したら `browser_take_screenshot` を 1 枚撮り、`get_screenshot` の Figma 画像と
   並べて全体の構造(要素の欠落・重なり・折り返し)だけ確認する。
   フォントレンダリング差によるサブピクセルの違いは差分として扱わない

### 照合表フォーマット

```
| 項目 | Figma | 実測 | 判定 |
|---|---|---|---|
| カード padding | 24px | 24px | ✅ |
| タイトル font-size | 20px | 20px | ✅ |
| リスト gap | 16px | 12px | ❌ → gap: var(--space-4) に修正 |
```

## フェーズ 5: 完了報告

以下を簡潔に報告する:

- 最終の照合表(全項目 ✅ の状態)
- 使用したトークンと新規追加したトークン
- 判断が必要な残課題(レスポンシブ挙動、ホバー等のインタラクション、Figma に未定義の状態)

## してはいけないこと

- スクリーンショットから余白・サイズ・色を目測して実装する
- 検証をスキップして「実装しました」と報告する
- Figma とブラウザのスクショのピクセル完全一致を追いかける(フォントレンダリング差で原理的に不可能。
  数値が一致していれば OK とする)
- localhost 以外の URL へ `browser_navigate` する(このワークフローで必要になることはない)
- トークンが存在するのにリテラル値をハードコードする