# Rust CLI Global Access Methods

## 概要
Rustで作成したCLIを、任意の作業ディレクトリから実行可能にする代表的な方法を整理する。個人利用からチーム配布まで、運用しやすい選択肢を比較する。

## 詳細
CLIをどこからでも使えるようにする方法は主に以下の5つ。

1. `cargo install --path` でユーザー環境へインストール  
   - `~/.cargo/bin` に配置されるため、`PATH` が通っていれば即利用できる。  
   - 開発中のローカルCLIを個人利用する場合に最も手軽。

2. リリースバイナリを固定ディレクトリへ配置して `PATH` に追加  
   - `cargo build --release` で生成したバイナリを `~/bin` などにコピーして運用する。  
   - Rustツールチェーンがない環境でも実行しやすい。

3. シンボリックリンクを作成  
   - 実体バイナリへのリンクを `~/bin` などに置く。  
   - バイナリ更新の反映が簡単だが、元ファイル削除時のリンク切れに注意。

4. シェル関数・alias でラップ  
   - 長いコマンドやデフォルト引数を隠蔽できる。  
   - 操作性向上に有効だが、実行ファイル配置の仕組みは別途必要。

5. Homebrew 配布  
   - チーム運用で導入・更新・ロールバックを標準化しやすい。  
   - 初期整備コストは高め。

実運用では、まず `cargo install --path` で始め、チーム配布が必要になった段階で Homebrew 化する流れが実践的。

## コード例
```bash
# 1) ローカルパスからインストール
cargo install --path /path/to/deploy-manual-cli --force

# 2) PATH に cargo bin を追加（zsh）
echo 'export PATH="$HOME/.cargo/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# 3) リリースバイナリを配置
cargo build --release
cp ./target/release/deploy-manual-cli ~/bin/
```

## 参考
- Rust Cargo Book: `cargo install`
- Homebrew Formula Cookbook

---
Generated: 2026-02-12
