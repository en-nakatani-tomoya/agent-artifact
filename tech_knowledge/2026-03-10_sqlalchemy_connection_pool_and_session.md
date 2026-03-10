# sqlalchemy connection pool and session

## 概要
SQLAlchemyのEngine（コネクションプール）とSessionの役割の違い、およびscoped_sessionによるスレッドローカルなセッション管理について整理

## 詳細
## コネクションプール (Engine)

SQLAlchemyのEngineは内部にコネクションプールを持つ。DBへのTCP接続を事前に確立しプール内に保持する。クエリ実行時はプールから空き接続を借りて使い、終わったら返却する。接続の確立（TCP handshake + 認証）は重い処理なので、これを使い回すのがプールの意義。

### 主要パラメータ
- pool_recycle: 接続の再利用期限（秒）。期限を超えた接続はプールから除去され新規作成される
- pool_timeout: プールから接続を取得する際の待機時間（秒）
- pool_size: プール内の接続数（デフォルト5）

## セッション (Session)

セッションはEngineの上に乗る作業単位（Unit of Work）。コネクションプールから接続を借り、クエリを実行し、トランザクションを管理する（commit / rollback）。close()は**DB接続を切断するのではなく、プールに返却する**だけ。セッションは軽量で作成・破棄のコストは小さいため、リクエストごとに作って閉じるのが正しい使い方。

## scoped_session

scoped_sessionはスレッドごとにセッションを管理するレジストリ。同一スレッド内では同じセッションが返り、remove()でスレッドからセッションを除去（接続はプールに返却）する。マルチスレッドのAPIサーバーで、スレッドをまたいでセッションが混ざらないようにするための仕組み。

## シングルトンパターンとの組み合わせ

Handlerクラスをシングルトンにする場合、Engineは1回だけ作成し共有するが、Sessionはリクエスト単位で取得する必要がある。sessionを直接属性として持つのではなく、@propertyでscoped_session経由で取得し、__exit__ではscoped_session.remove()で後片付けする設計が適切。

## コード例
```
from sqlalchemy import create_engine
from sqlalchemy.orm import scoped_session, sessionmaker

# Engine（コネクションプール）は1回だけ作成
engine = create_engine('mssql+pyodbc://...', pool_recycle=3600, pool_timeout=10)

# scoped_sessionでスレッドローカルなセッション管理
_session_factory = scoped_session(sessionmaker(bind=engine))

# セッション取得（同一スレッドでは同じセッションが返る）
session = _session_factory()
session.execute(...)

# 後片付け（接続をプールに返却 + レジストリから除去）
_session_factory.remove()
```

## 参考
- https://docs.sqlalchemy.org/en/20/core/pooling.html
- https://docs.sqlalchemy.org/en/20/orm/contextual.html

---
Generated: 2026-03-10
