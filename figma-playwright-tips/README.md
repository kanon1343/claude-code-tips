# Figma MCP × Playwright MCP — デザイン忠実実装ワークフロー

Claude Code で Figma デザインを Next.js / React / CSS に忠実に実装し、**Playwright で数値検証まで自動化**するためのセットアップとスキル集。

「実装 → 自分でスクショ → Claude に貼って指摘」の手動ラウンドトリップを、
「Figma の数値取得 → 実装 → getComputedStyle で数値照合 → 自己修正」の自動ループに置き換えます。

## 仕組み

```
Figma MCP                          Playwright MCP
─────────                          ──────────────
get_metadata        ─┐             browser_navigate (localhost のみ)
get_variable_defs    ├→ 実装 →     browser_resize (Figma フレーム幅に合わせる)
get_design_context   │             browser_evaluate (getComputedStyle で実測)
get_screenshot      ─┘                    │
      │                                   │
      └───────── 数値 vs 数値で照合 ←──────┘
                        │
              差分検出 → 修正 → 再検証(ループ)
                        │
              最後にスクショで全体確認(1回だけ)
```

ポイントは **ズレの検出を画像認識ではなく数値比較で行う**こと。
スクショの目測では「余白がちょっと広い気がする」までしか分かりませんが、
`getComputedStyle` と Figma メタデータの突き合わせなら「24px であるべき所が 32px」と確定できます。

## ディレクトリ構成

```
figma-playwright-tips/
├── README.md                  ... このファイル
├── setup.md                   ... セットアップ手順(MCP 追加・localhost 制限・permissions)
├── templates/
│   ├── settings.example.json  ... .claude/settings.json の例
│   └── claude-md-rules.md     ... CLAUDE.md に貼るルールテンプレート
└── skills/
    └── figma-frontend-impl/
        ├── SKILL.md           ... 実装+検証ワークフロースキル本体
        └── references/
            └── verification.md ... getComputedStyle 検証の具体的手順・スニペット
```

## クイックスタート

```bash
# 1. Figma MCP(公式プラグイン。MCP 設定 + Figma 公式スキルが入る)
claude plugin install figma@claude-plugins-official

# 2. Playwright MCP(ブラウザの通信先を localhost に制限)
claude mcp add playwright -- npx @playwright/mcp@latest \
  --allowed-origins "http://localhost:*"

# 3. スキルを配置
cp -r skills/figma-frontend-impl ~/.claude/skills/

# 4. permissions と CLAUDE.md を設定(setup.md 参照)
```

使い方:

```
このFigmaフレームを実装して: https://www.figma.com/design/xxxx?node-id=123-456
実装後、localhost:3000/pricing で数値検証まで行って
```

詳細は [setup.md](./setup.md) を参照。

## 前提条件

- Figma の **Dev シートまたは Full シート**(Professional / Organization / Enterprise プラン)。
  Starter プランや View シートは月 6 ツールコールまでの制限があり実用不可
- Node.js 18+
- Claude Code
- 対象プロジェクトの dev server がローカルで起動できること

## セキュリティ設計(要点)

| レイヤ | 設定 | 効果 |
|---|---|---|
| Playwright MCP | `--allowed-origins "http://localhost:*"` | ブラウザの通信先を localhost に制限(ページのサブリソース含む) |
| Claude Code permissions | 検証系ツールのみ allow | 意図しないツール実行は都度確認 |
| 運用 | `--dangerously-skip-permissions` を使わない | 権限確認の防御層を維持 |

注意: `--allowed-origins` は公式に「セキュリティ境界ではない」(リダイレクトに効かない)と明記されています。
ガードレールとして有効ですが、ハードな保証が必要な場合はネットワークレベル(プロキシ/サンドボックス)の制御を検討してください。
詳細は [setup.md のセキュリティ節](./setup.md#4-セキュリティ設定) を参照。

## よくあるハマりどころ

- **Figma MCP を入れてもズレる** → スクショの雰囲気で実装しているのが原因。`get_metadata` /
  `get_variable_defs` の数値を使わせ、実装後の数値照合を必須化する(このリポジトリのスキルがそれ)
- **スクショのピクセル差分で比較したくなる** → Figma とブラウザはフォントレンダリングが異なるため
  完全一致は原理的に不可能。誤検知だらけになるので数値照合を主、スクショは粗チェックのみに
- **大きいフレームを一括変換** → コンテキスト溢れで精度が落ちる。公式推奨どおり
  Card / Header などコンポーネント単位に分割して取得する
- **Figma ファイル側が汚い** → Auto Layout 未使用・変数未使用・`Group 5` のような命名だと
  メタデータ自体が汚く、ツールでは救えない。デザイン側の整備が先

## 参考リンク

### 公式

- [Figma MCP Server Guide (figma/mcp-server-guide)](https://github.com/figma/mcp-server-guide) — 公式ベストプラクティス・ルール例・公式スキル
- [Figma MCP developer docs](https://developers.figma.com/docs/figma-mcp-server/) — ツール一覧・プラン別制限
- [Claude Code and Figma: Set up the MCP server (Figma Help)](https://help.figma.com/hc/en-us/articles/39888612464151-Claude-Code-and-Figma-Set-up-the-MCP-server)
- [Playwright MCP (microsoft/playwright-mcp)](https://github.com/microsoft/playwright-mcp) — 設定オプション・セキュリティ注意
- [Claude Code: Settings / Permissions](https://code.claude.com/docs/en/settings)
- [Claude Code: Security](https://code.claude.com/docs/en/security)

### コミュニティ実例

- [Figma MCP でデザイン通りに AI コーディングするためのプラクティス (Zenn)](https://zenn.dev/peoplex_blog/articles/2604-figma-ai-coding-practices)
- [Figma MCPの精度を更に上げるTips (Zenn)](https://zenn.dev/canly/articles/78dcc98c3dfb46)
- [Claude Code × Figma MCP × Playwright MCP チーム開発の取り組み (Qiita)](https://qiita.com/kamihork/items/9a938ed04ff35e9e3f9e)
- [Pixel-Perfect UI with Playwright and Figma MCP](https://vadim.blog/pixel-perfect-playwright-figma-mcp/)
- [Figma-to-React pipeline that visually verifies its own work (Medium)](https://medium.com/@aliafsah1988/how-to-turn-claude-code-into-a-figma-to-react-pipeline-that-visually-verifies-its-own-work-030246f600a9)