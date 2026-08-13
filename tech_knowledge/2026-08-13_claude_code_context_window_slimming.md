# claude code context window slimming

## 概要
Claude Code のコンテキストウィンドウを圧迫する MCP ツール・システムツールの特定と削減手順。/context で内訳を確認し、MCP サーバー削除と permissions.deny で常時ロード分を削る。

## 詳細
## 調査方法
/context コマンドでカテゴリ別のトークン内訳が見られる。重要なのは deferred（遅延ロード）区分で、'MCP tools (deferred)' はスキーマ未ロード（名前のみ）なので実質コンテキストを消費しない。削減対象は常時ロードされる 'MCP tools' と 'System tools' のみ。

## MCP サーバーの削減
- 一覧: claude mcp list
- 削除: claude mcp remove <name>（スコープ指定は -s user/-s project/-s local）
- claude mcp list に出るのに remove で 'No MCP server named ...' になる場合、~/.mcp.json（ホームディレクトリ直下の .mcp.json）に定義されていることがある。この場合はファイルを直接編集して mcpServers からエントリを削除する
- プロジェクト単位の無効化: .claude/settings.json の disabledMcpjsonServers / enabledMcpjsonServers
- デスクトップアプリ組み込みの extension（iOS Simulator, Claude_Browser, visualize, claude-in-chrome 等）は CLI 設定では消せず、アプリの Settings → Extensions からオフにする

## システムツール（ビルトイン）の削減
- ~/.claude/settings.json の permissions.deny にツール名を列挙（例: Workflow, Artifact）。Workflow は単体で約6-7k トークンと最大級
- 確実にコンテキストから外すなら起動フラグ --disallowedTools "Workflow,Artifact" も使える（deny はバージョンによって定義がコンテキストに残る場合がある）

## 反映確認
変更後は新セッションを開いて /context で確認する。

## 参考
- https://docs.anthropic.com/en/docs/claude-code/mcp
- https://docs.anthropic.com/en/docs/claude-code/settings

---
Generated: 2026-08-13
