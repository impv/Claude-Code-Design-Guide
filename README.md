# Claude Code Design Guide

Claude Code の機能を「**なぜ選ぶのか**」の観点で整理した設計判断ガイド。

## 公式ドキュメントとの違い

公式ドキュメントは各機能の**設定方法**を教える。このガイドは3つの設計軸で全機能を位置づけ、**いつ何を使うか**の判断基準を示す。

- **確定性**: settings.json / Hooks（確定的）から Subagents（推論的）までのグラデーション
- **コンテキストコスト**: 各機能がコンテキストウィンドウに与える影響
- **強制度**: CLAUDE.md（方向づけ）→ deny ルール（ブロック）→ サンドボックス（OS レベル遮断）の段階

## ガイド本体

[claude-code-design-guide.md](./claude-code-design-guide.md)

## 対象読者

Claude Code を使っていて「CLAUDE.md に書くべきか、settings.json に書くべきか」「Skills と Subagents のどちらを使うべきか」といった判断に迷う人。

## 公式ドキュメント

- [Claude Code Documentation](https://code.claude.com/docs)
