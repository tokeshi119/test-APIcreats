# Cursor AI運用テンプレート

## 目的
このリポジトリは、Cursorを使ったAI駆動開発を安定運用するためのテンプレートです。
プロジェクトごとに同じ構成を配置し、ルールとスキルを再利用します。

## 使い方（テンプレ運用）
1. このリポジトリを新規プロジェクトにコピーする
2. ルートに `AGENTS.md` / `rules.md` / `skills.md` を置く
3. `.cursor/rules` に共通ルールを置く
4. `.cursor/skills` に作業スキルを置く

## GitHubでテンプレ化する手順
1. GitHubのリポジトリ画面を開く
2. Settings → General に移動する
3. 「Template repository」を有効化する

## ディレクトリ構成
```
.
├─ .cursor/
│  ├─ rules/
│  └─ skills/
├─ AGENTS.md
├─ rules.md
└─ skills.md
```

## 更新ルール
- ルールは短く、行動可能な形で書く
- スキルは1ファイル=1テーマにする
- 使った後に随時更新する
