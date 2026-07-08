# セットアップ手順

Figma MCP + Playwright MCP を Claude Code に導入し、localhost 制限・permissions・CLAUDE.md ルールまで設定する手順。

## 前提

- Figma: Professional / Organization / Enterprise プランの **Dev または Full シート**
  (Starter・View シートは月 6 ツールコール制限があり実用不可)
- Node.js 18+
- Claude Code インストール済み

## 1. Figma MCP の追加

推奨は公式プラグイン。MCP 設定に加え、Figma 公式のスキル(デザイン実装・Code Connect 連携等)が入る:

```bash
claude plugin install figma@claude-plugins-official
```

MCP だけ手動で入れる場合:

```bash
claude mcp add --transport http figma https://mcp.figma.com/mcp
```

初回利用時に Claude Code 内で `/mcp` → figma を選択し、ブラウザで OAuth 認証する。

確認:

```bash
claude mcp list
```

Claude Code 内で `get_design_context` 等のツールが見えれば OK。

> デスクトップ版(Figma アプリ内蔵のローカルサーバー、`http://127.0.0.1:3845/mcp`)もあるが、
> リモート版(`mcp.figma.com`)の方がアプリ起動不要で機能追加も先行するため基本はリモート版を推奨。

## 2. Playwright MCP の追加(localhost 制限つき)

```bash
claude mcp add playwright -- npx @playwright/mcp@latest \
  --allowed-origins "http://localhost:*"
```

- `--allowed-origins` はブラウザが通信できるオリジンの許可リスト。
  ページ遷移だけでなく**ページが読み込むサブリソース(CDN・フォント・アナリティクス)もブロック**される
- `http://localhost:*` のワイルドカードポートが公式サポートされているので、dev server のポートが変わっても効く
- localhost ページが外部リソースに依存している場合(Google Fonts 等)は表示が崩れるので、
  必要なオリジンだけセミコロン区切りで追加する:

```bash
--allowed-origins "http://localhost:*;https://fonts.googleapis.com;https://fonts.gstatic.com"
```

**注意(公式ドキュメントより)**: このフラグは「セキュリティ境界ではなく、リダイレクトには効かない」。
Claude がうっかり外部サイトへ行くのを防ぐガードレールとしては十分だが、
監査要件レベルの保証が必要ならネットワークレイヤ(プロキシ / devcontainer の egress 制限)で行うこと。

## 3. permissions の設定

プロジェクトの `.claude/settings.json`(チーム共有するなら)または `.claude/settings.local.json`(個人用)に、
検証ループで使うツールを事前許可しておくとループが承認ダイアログで止まらない。

[templates/settings.example.json](./templates/settings.example.json) をコピーして調整:

```json
{
  "permissions": {
    "allow": [
      "mcp__figma__get_design_context",
      "mcp__figma__get_metadata",
      "mcp__figma__get_variable_defs",
      "mcp__figma__get_screenshot",
      "mcp__playwright__browser_navigate",
      "mcp__playwright__browser_resize",
      "mcp__playwright__browser_snapshot",
      "mcp__playwright__browser_take_screenshot",
      "mcp__playwright__browser_evaluate"
    ]
  }
}
```

補足:

- MCP ツールの permission ルールは**ツール単位**。`browser_navigate(url:...)` のような
  引数レベルのマッチは書けないため、URL の制限は手順 2 の `--allowed-origins`(サーバー側)が担当する
- navigate を毎回目視確認したい場合は allow から `browser_navigate` を外す(デフォルトで都度確認になる)
- `--dangerously-skip-permissions` はこのワークフローでは使わない。
  権限確認はプロンプトインジェクション対策の最後の防御層

## 4. セキュリティ設定

このワークフローで発生する通信は 3 経路。「localhost 以外に一切通信しない」構成は成立しない前提で、
各経路を制御する:

| 経路 | 行き先 | 制御 |
|---|---|---|
| Playwright のブラウザ | localhost(+ページのサブリソース) | `--allowed-origins` |
| Figma MCP | `mcp.figma.com` | Figma アカウント権限の範囲のみ。公式サーバー以外(コミュニティ製 Figma MCP)は使わない |
| Claude Code 本体 | Anthropic API | スクショ・メタデータは会話コンテキストとして送信される。未公開デザインを扱う場合は所属組織のポリシーを確認 |

## 5. CLAUDE.md にルールを追加

[templates/claude-md-rules.md](./templates/claude-md-rules.md) の内容をプロジェクトの `CLAUDE.md` に貼る。
Figma 公式ガイド推奨のアセット取り扱いルール + 数値照合の強制ルールのセット。

これを書かないと、Claude は `get_design_context` の出力とスクショの雰囲気で実装しがちで、
「Figma MCP を入れたのにズレる」状態になる。

## 6. スキルの配置

```bash
cp -r skills/figma-frontend-impl ~/.claude/skills/
```

プロジェクト単位で入れる場合は `<project>/.claude/skills/` に配置。

## 7. 動作確認

dev server を起動した状態で:

```
このFigmaフレームを実装して: <FigmaのフレームURL>
対象: src/components/PricingCard.tsx
実装後、localhost:3000/pricing で数値検証まで行って
```

期待する動き:

1. Figma からメタデータ・変数・スクショを取得
2. 実装
3. Playwright で localhost を開き、Figma フレームと同幅にリサイズ
4. getComputedStyle の実測値と Figma の数値の突き合わせ表を出す
5. 差分を修正して再検証、一致したら最終スクショで全体確認

## トラブルシューティング

- **Figma ツールが出てこない** → `/mcp` で認証状態を確認。プラン/シートの権限も確認
  (Dev/Full シートでないとほぼ使えない)
- **レスポンスが途切れる・遅い** → 選択フレームが大きすぎる。公式推奨どおり
  コンポーネント単位に分割して `get_metadata` でノードマップ → 必要ノードだけ再取得
- **ツールコール上限** → Organization: 200 回/日、Enterprise: 600 回/日(Tier 1 REST API 相当のレート制限)。
  無駄撃ちを減らすには get_screenshot を最終確認に限定する
- **localhost の表示が崩れる** → 外部フォント等が `--allowed-origins` でブロックされている。
  必要オリジンを追加する