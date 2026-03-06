# Locust 概要

**Locust** は Python で書かれたオープンソースの負荷テスト（ロードテスト）フレームワーク。最新バージョンは **2.43.3**、MIT ライセンス。

## 核心コンセプト

テストシナリオを **純粋な Python コード** で記述する。XML や GUI ベースの設定は不要。

```python
from locust import HttpUser, task

class HelloWorldUser(HttpUser):
    @task
    def hello_world(self):
        self.client.get("/hello")
        self.client.get("/world")
```

## 主な特徴

| 特徴 | 詳細 |
|---|---|
| **Pure Python** | テストは通常のPythonコード。任意のライブラリをimport可能 |
| **高並行性** | gevent（greenlet）ベースのイベント駆動。1プロセスで数千の同時ユーザーを処理 |
| **分散実行** | `--master` / `--worker` で複数マシンに負荷を分散可能 |
| **Web UI** | `localhost:8089` でリアルタイム監視（RPS、レスポンスタイム、エラー率）。実行中に負荷変更も可 |
| **ヘッドレスモード** | `--headless` でCLIのみ実行。CI/CD組み込みに最適 |
| **プロトコル非依存** | HTTPが主だが、カスタムクライアントで任意のプロトコルをテスト可能 |
| **拡張性** | プラグイン、OpenTelemetry連携、Kubernetes Operator、VS Code拡張あり |

## 使い方の流れ

1. `pip install locust`
2. `locustfile.py` を作成（上記のようなUserクラスを定義）
3. `locust` コマンドで起動 → ブラウザで `http://localhost:8089` にアクセス
4. ユーザー数・スポーンレートを設定してテスト開始
5. ヘッドレスの場合: `locust --headless --users 10 --spawn-rate 1 -H http://your-server.com`

## アーキテクチャ

- 各仮想ユーザーは独立した **greenlet**（軽量コルーチン）で動作
- ブロッキングコードをそのまま書ける（コールバック不要）
- イベント駆動なのでスレッドベースより圧倒的に軽量

## Sources

- [Locust 公式サイト](https://locust.io/)
- [What is Locust? — 公式ドキュメント](https://docs.locust.io/en/stable/what-is-locust.html)
- [Your first test — クイックスタート](https://docs.locust.io/en/stable/quickstart.html)
- [GitHub リポジトリ](https://github.com/locustio/locust)
