# Playwright 数値検証の具体的手順

## browser_evaluate スニペット

対象要素にはセレクタが必要。`data-testid` があればそれを使い、なければクラス名等で特定する。
複数要素をまとめて 1 回の evaluate で取ると往復が減る:

```js
() => {
  const pick = (sel, props) => {
    const el = document.querySelector(sel);
    if (!el) return { error: `not found: ${sel}` };
    const s = getComputedStyle(el);
    return Object.fromEntries(props.map(p => [p, s[p]]));
  };
  return {
    card: pick('[data-testid="pricing-card"]',
      ['width', 'paddingTop', 'paddingRight', 'paddingBottom', 'paddingLeft',
       'borderRadius', 'backgroundColor', 'gap', 'display', 'flexDirection']),
    title: pick('[data-testid="pricing-card"] h2',
      ['fontSize', 'lineHeight', 'fontWeight', 'color', 'marginTop', 'marginBottom']),
    list: pick('[data-testid="pricing-card"] ul',
      ['gap', 'paddingLeft', 'marginTop']),
  };
}
```

boundingBox(実寸)が欲しい場合:

```js
() => {
  const r = document.querySelector('[data-testid="pricing-card"]').getBoundingClientRect();
  return { width: r.width, height: r.height, x: r.x, y: r.y };
}
```

## 値の比較時の注意

- **色は形式を揃える**: getComputedStyle は `rgb(37, 99, 235)` を返す。Figma の HEX
  (`#2563EB`)を RGB に変換して比較する
- **line-height**: Figma は px または %、CSS は px で返る。Figma が「150%」なら
  font-size × 1.5 と比較する
- **rem 使用時**: 実測は px で返るので、root の font-size(通常 16px)で換算して考える
- **box-sizing**: `border-box` 前提で width を比較する。`content-box` の要素は
  width + padding + border が Figma の幅に対応する
- **Figma の stroke**: Figma のストローク位置(inside/center/outside)により
  CSS の border と 1〜2px ずれることがある。inside stroke = border(border-box)が対応
- **UA デフォルトスタイル**: h1〜h6, ul, p のデフォルト margin が意図しない差分の常連。
  リセット CSS の有無を確認する
- **letter-spacing**: Figma の % 指定は font-size に対する割合。px に換算して比較する

## viewport の合わせ方

- `browser_resize` の width は Figma のトップレベルフレーム幅(例: 1440)に合わせる
- 高さはスクロールがあってよいので 900 程度で固定し、フル比較が必要なときだけ
  `browser_take_screenshot` の `fullPage: true` を使う

## よくある差分の原因(デバッグの当たり所)

1. gap のはずが margin で実装している(要素間の余白が margin 相殺で変わる)
2. Figma の Auto Layout padding とコンテナの padding の対応漏れ
3. フォントウェイトの読み込み漏れ(500 を指定しても 400 か 700 に丸められている →
   computed の fontWeight と実際の描画を両方確認)
4. 画像の aspect-ratio / object-fit 未指定でサイズが揺れる
5. flex の min-width: auto によるはみ出し(min-width: 0 で解消することが多い)

## 検証が通らないとき

同じ差分が 2 回の修正で解消しない場合は、原因の仮説(上のリスト等)を明示してから修正する。
それでも解消しない場合は無限ループせず、差分の内容・試したこと・残る仮説をユーザーに報告して判断を仰ぐ。