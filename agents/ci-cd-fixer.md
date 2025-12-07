# CI/CD Fixer Agent

CI失敗時の自動診断・修正を行うエージェント。3回失敗した場合はCursor（PM）にエスカレーションします。

---

## 呼び出し方法

```
Task tool で subagent_type="ci-cd-fixer" を指定
```

## 入力

```json
{
  "run_id": "string (optional)",
  "max_retries": 3
}
```

## 出力

```json
{
  "status": "fixed" | "escalated" | "no_issues",
  "attempts": number,
  "fixes_applied": [
    {
      "file": "string",
      "issue": "string",
      "fix": "string"
    }
  ],
  "escalation_report": "string (if escalated)"
}
```

---

## 処理フロー

### Step 1: CI状態の確認

```bash
# 最新のCI実行を取得
gh run list --limit 5

# 失敗している場合は詳細を取得
gh run view {{run_id}} --log-failed
```

### Step 2: エラー分類

エラーログを分析し、以下のカテゴリに分類：

| カテゴリ | パターン | 自動修正可能 |
|---------|---------|-------------|
| **TypeScript エラー** | `TS\d{4}:`, `error TS` | ✅ Yes |
| **ESLint エラー** | `eslint`, `Parsing error` | ✅ Yes |
| **テスト失敗** | `FAIL`, `AssertionError` | ⚠️ 場合による |
| **ビルドエラー** | `Build failed`, `Module not found` | ✅ Yes |
| **依存関係エラー** | `npm ERR!`, `Could not resolve` | ✅ Yes |
| **環境エラー** | `env`, `secret`, `permission` | ❌ No（エスカレーション） |

### Step 3: 自動修正の実行

**TypeScript エラーの場合**:
```bash
# エラー箇所を特定
npx tsc --noEmit 2>&1 | head -50

# 修正を適用（Editツール使用）
```

**ESLint エラーの場合**:
```bash
# 自動修正を試行
npx eslint --fix .
```

**依存関係エラーの場合**:
```bash
# package-lock.json を再生成
rm -rf node_modules package-lock.json
npm install
```

### Step 4: 修正のコミット & プッシュ

```bash
git add -A
git commit -m "fix: CI エラーを修正

- {{修正内容1}}
- {{修正内容2}}

🤖 Generated with Claude Code (CI auto-fix)"

git push
```

### Step 5: CI再実行の確認

```bash
# 新しいCI実行を監視
gh run watch
```

### Step 6: 結果判定

**成功した場合**:
```json
{
  "status": "fixed",
  "attempts": 1,
  "fixes_applied": [
    {
      "file": "src/index.ts",
      "issue": "TS2322: Type 'string' is not assignable to type 'number'",
      "fix": "型を修正"
    }
  ]
}
```

**3回失敗した場合**:
エスカレーションレポートを生成

---

## エスカレーションレポートフォーマット

```markdown
## ⚠️ CI失敗エスカレーション

**失敗回数**: 3回
**最新のrun_id**: {{run_id}}
**ブランチ**: {{branch}}

---

### エラー内容

{{エラーログの要約（最大50行）}}

---

### 試した修正

| 試行 | 修正内容 | 結果 |
|------|---------|------|
| 1 | {{修正1}} | ❌ 失敗 |
| 2 | {{修正2}} | ❌ 失敗 |
| 3 | {{修正3}} | ❌ 失敗 |

---

### 推定原因

{{根本原因の推測}}

---

### Cursorへの依頼

以下の対応を検討してください：
1. {{具体的な依頼1}}
2. {{具体的な依頼2}}

---

### 参考情報

- CI ログ: `gh run view {{run_id}} --log`
- 関連ファイル: {{関連ファイル一覧}}
```

---

## 自動修正できないケース

以下の場合は即時エスカレーション（修正を試みない）：

1. **環境変数・シークレット関連**: 設定変更が必要
2. **権限エラー**: GitHub/デプロイ先の設定が必要
3. **外部サービス障害**: 一時的な問題の可能性
4. **設計上の問題**: 根本的な修正が必要

---

## 注意事項

- **3回ルール厳守**: 4回以上の自動修正は行わない
- **破壊的変更禁止**: テストを削除したり、エラーを握りつぶす修正は禁止
- **変更を記録**: 全ての修正をコミットログに残す
- **環境を汚さない**: `node_modules` 削除時は慎重に
