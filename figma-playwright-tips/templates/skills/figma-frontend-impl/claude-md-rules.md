# CLAUDE.md に追加するルールテンプレート

以下をプロジェクトの `CLAUDE.md` にコピペする。
前半は [Figma 公式ガイド](https://github.com/figma/mcp-server-guide)推奨のルール、後半が数値照合の強制ルール。

```markdown
## Figma MCP ルール

### 必須フロー(スキップ禁止)
1. まず get_design_context で対象ノードの構造化表現を取得する
2. レスポンスが大きすぎる/途切れる場合は get_metadata でノードマップを取得し、
   必要なノードだけ get_design_context で再取得する
3. get_screenshot で対象バリアントの見た目のリファレンスを取得する
4. 上記が揃ってから実装を開始する
5. get_design_context の出力(React + Tailwind 形式)はデザインの表現であって
   最終コードではない。このプロジェクトの規約・コンポーネント・トークンに翻訳する

### アセットの取り扱い
- Figma MCP はアセット用エンドポイント(画像・SVG)を提供する
- IMPORTANT: Figma MCP が localhost ソースの画像/SVG を返したら、そのソースを直接使う
- IMPORTANT: 新しいアイコンパッケージを追加しない。アセットはすべて Figma ペイロードにある
- IMPORTANT: localhost ソースが提供されている場合、プレースホルダーを作らない

### 数値照合ルール(デザインずれ防止)
- 余白・色・タイポグラフィの値をスクリーンショットから目測しない。
  必ず get_metadata / get_variable_defs の数値を使う
- Figma 変数に対応する CSS カスタムプロパティ/既存トークンがあればそれを使い、
  リテラル値をハードコードしない
- 実装後は Playwright MCP で対象ページを開き、browser_resize で Figma フレームと
  同じ幅にした上で、browser_evaluate の getComputedStyle で実測値を取得し、
  Figma の数値と突き合わせて差分表を報告・修正する
- スクリーンショット比較は数値照合がすべて一致した後の最終確認のみ。
  Figma とブラウザはフォントレンダリングが異なるため、ピクセル完全一致は求めない
```